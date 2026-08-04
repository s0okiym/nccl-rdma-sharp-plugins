# IB Plugin 深度解析（master 分支，net API v6–v11）

本文基于 `master` 分支代码（提交 `6b636fe` 之后），系统讲解 `src/ib_plugin.c`（约 2300 行）
及其支撑代码的实现。master 线相比 v3 时代（见姊妹篇
`notes/ib_plugin_analysis_hpcx-v2.7.0.md`）有巨大演进：多版本 API 并存、多网卡融合
（vDev）、多 QP 条带化、tagged multi-recv、ECE、DMA-BUF、自适应路由等。文中引用格式
为 `文件:行号`。

## 1. 总体架构与多版本分发

### 1.1 六个导出符号

master 不再只导出一个 `ncclNetPlugin_v3`，而是**同时导出 v6–v11 六个符号**
（`src/p2p_plugin.c:86-114`），NCCL 按自身编译的 API 版本选用：

```c
ncclNet_v11_t ncclNetPlugin_v11 = { "NCCL RDMA Plugin v11", pluginInit_v11 };
ncclNet_v10_t ncclNetPlugin_v10 = { "NCCL RDMA Plugin v10", pluginInit_v10 };
... v9 / v8 / v7 / v6 同构
```

每个符号初始只有 name 和 init 两个字段。任一版本的 `pluginInit_vN` 首次被调用时都
执行 `pluginSetup()`（`p2p_plugin.c:123`）：读 `NCCL_PLUGIN_P2P`
（`ib`/`ucx`/`ucx_rma`/`ucx_uct`/`ucx_uct_read`，默认 `ib`），然后用**结构体整体
赋值把全部六个版本表一次性替换**为选中后端的表（`p2p_plugin.c:144-189`），再转发
调用该版本的 `init`。之后 NCCL 再读符号时拿到的就是完整函数表。

### 1.2 IB 后端的六张函数表与版本适配层

`ib_plugin.c:2201-2335` 定义 `ibPlugin_v6` … `ibPlugin_v11`。**v11 是原生实现**，
其余版本通过一串 `_vN` 包装函数做签名适配，形成调用链：

| 版本 | 适配点 |
|---|---|
| v11 | 原生：`init(ctx, commId, config, …)` 支持每 communicator 上下文、`makeVDevice`、`finalize`、`setNetAttr` |
| v10 | `init` 去掉 ctx/config；`connect` 无 comm config |
| v9 | `isend/irecv` 去掉 proxy handle（`phandle`）参数 |
| v8 | `isend` size 为 `int`；`irecv` sizes 为 `int*` |
| v7 | `regMr` size 为 `int` |
| v6 | `connect/accept` 无 device-offload handle 参数 |

例：`ncclIbConnect_v6` → `ncclIbConnect_v9` → `ncclIbConnect_v10` → `ncclIbConnect`
（`ib_plugin.c:2191-2151`）。`getProperties` 也类似：内部统一产出 v11 的
`ncclNetProperties_t`，再逐字段拷贝降级到 v6–v10（`ib_plugin.c:392-499`）。

`ibPlugin_v11` 全貌（`ib_plugin.c:2201`）：

```c
const ncclNet_v11_t ibPlugin_v11 = {
  .name = "IBext_v11",
  .init = ncclIbInit, .devices = ncclIbDevices, .getProperties = ncclIbGetProperties,
  .listen = ncclIbListen, .connect = ncclIbConnect, .accept = ncclIbAccept,
  .regMr = ncclIbRegMr, .regMrDmaBuf = ncclIbRegMrDmaBuf, .deregMr = ncclIbDeregMr,
  .isend = ncclIbIsend, .irecv = ncclIbIrecv, .iflush = ncclIbIflush, .test = ncclIbTest,
  .closeSend = ncclIbCloseSend, .closeRecv = ncclIbCloseRecv, .closeListen = ncclIbCloseListen,
  NULL /* getDeviceMr */, NULL /* irecvConsumed */,   // 不支持 GPU 侧 device offload
  ncclIbMakeVDevice, ncclIbFinalize, ncclIbSetNetAttr
};
```

## 2. 两级设备模型：物理 dev + 虚拟 merged dev

这是 master 最重要的结构性变化。v3 时代一个 NCCL "dev" 就是一张网卡一个端口；
master 拆成两层（`include/p2p_plugin.h`）：

- **物理设备** `ncclIbDevs[MAX_IB_DEVS]`（上限 32，`p2p_plugin.h:137`）——一个
  ibv 端口一项，持有 `ibv_context`、缓存的 `portAttr`、共享 PD（`pd` + `pdRefs`
  引用计数）、**MR 缓存**（`mrCache`）、自适应路由标志 `ar`、provider 能力
  （mlx5 / data direct）、`dmaBufSupported`、per-dev 错误统计等。
- **虚拟设备** `ncclIbMergedDevs[MAX_IB_VDEVS]`（32×8=256，`p2p_plugin.h:56`）——
  NCCL 看到的 "dev" 都是 vDev。每个 vDev 是 `ncclNetVDeviceProps_t vProps`
  （`ndevs` + 物理 dev 下标数组，最多 `NCCL_IB_MAX_DEVS_PER_NIC=4` 个物理设备）
  加上聚合名（`mlx5_0+mlx5_1`）和聚合带宽。

设备枚举时每发现一个物理端口，自动为它创建一个单设备的 vDev
（`p2p_plugin.c:668-673`）。之后 NCCL 核心可通过 `makeVDevice` 回调
（`ncclIbMakeVDevice`，`ib_plugin.c:2135` → `ncclIbMakeVDeviceInternal`，
`p2p_plugin.c:469`）把多张物理卡**融合成一个 vDev**（多轨 rail/NIC fusion）：
校验 `NCCL_IB_MERGE_NICS`、数量 ≤4、链路类型一致，速度求和、名字用 `+` 拼接。
`devices()`/`getProperties()` 返回的都是 vDev 视图；`connect/accept` 拿到的 `dev`
参数也是 vDev 下标，再从 `vProps.devs[]` 取出物理设备逐个建 verbs 资源。

## 3. 关键数据结构

### 3.1 物理设备 `ncclIbDev`（`p2p_plugin.h:106`）

```c
typedef struct ncclIbDev {
  pthread_mutex_t lock;        // 保护 pdRefs / mrCache
  int device; uint64_t guid;
  uint8_t portNum, link, isSharpDev;
  int speed;                   // Mbps，支持 active_speed_ex（NDR/XDR）
  struct ibv_context* context;
  int pdRefs; struct ibv_pd* pd;   // 全进程每设备共享一个 PD
  char devName[MAXNAMESIZE];   // data-direct 设备为 "<name>_dma"
  char *pciPath, *virtualPciPath;
  int realPort, maxQp;
  float latency;
  struct ncclIbMrCache mrCache;    // 见 §7
  int ar;                          // 自适应路由（IB 默认开）
  struct ibv_port_attr portAttr;   // 枚举时缓存，后续不再 query
  struct ncclIbStats stats;
  int dmaBufSupported;             // 1 / -1（探测结果缓存）
  enum ncclIbProvider ibProvider;  // NONE / MLX5
  union { struct { int dataDirect; } mlx5; } capsProvider;
} ncclIbDev;
```

### 3.2 通信对象：公共基座 + 收发特化

```c
typedef struct ncclIbNetCommBase {            // p2p_plugin.h:622（在 ib_plugin.c）
  ncclNetVDeviceProps_t vProps;   // 本 comm 使用的 vDev（物理 dev 列表）
  bool isSend;
  struct ncclIbRequest reqs[MAX_REQUESTS];  // 请求池（64）
  struct ncclIbQp qps[NCCL_IB_MAX_QPS];     // QP 数组（最多 128）
  int nqps, qpIndex, devIndex;              // qpIndex/devIndex 为轮转游标
  struct ncclSocket sock;                   // bootstrap TCP（状态机封装）
  int ready;
  int nRemDevs, nDataQps;                   // 对端设备数 / 数据 QP 数
  struct ncclIbDevInfo remDevs[NCCL_IB_MAX_DEVS_PER_NIC]; // 对端 per-dev 信息
  struct ncclIbStats stats;                 // 异步致命错误计数（见 §9）
} ncclIbNetCommBase;                          // 32 字节对齐（静态断言保证）

struct ncclIbQp { struct ibv_qp* qp; int devIndex; int remDevIdx; };
// devIndex：本地 devs[] 下标；remDevIdx：对端 remDevs[] 下标
```

每个物理设备在 comm 内的资源是 `ncclIbNetCommDevBase`（`p2p_plugin.h:93`）：
`ibDevN`（物理 dev 下标）、共享 `pd`、**自己的 CQ**（`cq_context` 指向
`comm->base.stats`）、本地 GID 信息（`gidInfo`，错误日志用）。

发送端（`ib_plugin.c:640`）：

```c
struct ncclIbSendComm {
  struct ncclIbNetCommBase base;
  struct ncclIbSendFifo fifo[MAX_REQUESTS][NCCL_NET_IB_MAX_RECVS]; // 64×8 credit 槽
  struct ibv_sge sges[NCCL_NET_IB_MAX_RECVS];
  struct ibv_send_wr wrs[NCCL_NET_IB_MAX_RECVS + 1];  // 预分配的 WR 链
  struct ncclIbSendCommDev devs[NCCL_IB_MAX_DEVS_PER_NIC];  // per-dev：base + fifoMr
  struct ncclIbRequest* fifoReqs[MAX_REQUESTS][NCCL_NET_IB_MAX_RECVS]; // 槽→请求反查
  struct ncclIbRemSizesFifo remSizesFifo;   // 对端 sizesFifo 的地址/rkey + 本地 staging
  uint64_t fifoHead;
  int ar;   // 本连接启用 AR：所有合并 dev 都开 AR 才为 1
};
```

接收端（`ib_plugin.c:674`）：

```c
struct ncclIbRecvComm {
  struct ncclIbNetCommBase base;
  struct ncclIbRecvCommDev devs[NCCL_IB_MAX_DEVS_PER_NIC]; // base + gpuFlush + fifoMr/fifoSge + sizesFifoMr
  struct ncclIbRemFifo remFifo;      // 本地 staging elems[64][8] + 对端 fifo 地址
  int sizesFifo[MAX_REQUESTS][NCCL_NET_IB_MAX_RECVS];      // 每组接收的实际字节数（对端 RDMA 写）
  int gpuFlushHostMem, flushEnabled;
};
```

### 3.3 credit 槽 `ncclIbSendFifo`（`ib_plugin.c:584`）

```c
struct ncclIbSendFifo {     // 32 字节，静态断言要求整体 32B 对齐
  uint64_t addr;            // 接收缓冲地址
  uint64_t size;
  uint32_t rkeys[NCCL_IB_MAX_DEVS_PER_NIC]; // 每个对端 dev 一个 rkey（多轨各自 DMA）
  uint32_t nreqs;           // 本组包含几个接收（multi-recv）
  uint32_t tag;             // NCCL proxy tag，发送端按 tag 匹配
  uint64_t idx;             // 就绪序号 = 接收端 fifoTail+1
  char padding[16];
};
```

### 3.4 请求 `ncclIbRequest` 与 wr_id 编码（`p2p_plugin.h:66`）

```c
struct ncclIbRequest {
  struct ncclIbNetCommBase* base;
  int type;                       // UNUSED / SEND / RECV / FLUSH
  struct ncclSocket* sock;        // 错误日志取对端地址用
  int events[NCCL_IB_MAX_DEVS_PER_NIC];              // 每个 dev 还欠几个完成事件
  struct ncclIbNetCommDevBase* devBases[NCCL_IB_MAX_DEVS_PER_NIC]; // 到哪个 CQ 上 poll
  int nreqs;
  union { struct { int size; void* data; uint32_t lkeys[4]; int offset; } send;
          struct { int* sizes; } recv; };
};
```

完成模型从 v3 的 `done` 标志改为**事件计数**：`ncclIbAddEvent()`（`ib_plugin.c:730`）
在每次 post WR 前给对应 dev 的 `events[devIndex]++`；`test()` 轮询时逐 WC 递减；
全部归零即完成。不变式：请求被释放时 events 必然全 0，因此 `ncclIbGetRequest()`
（`ib_plugin.c:1457`）复用槽位时只需轻量复位。

`MAX_REQUESTS = NCCL_NET_MAX_REQUESTS(8) × NCCL_NET_IB_MAX_RECVS(8) = 64`
（`p2p_plugin.h:25-27`）。`wr_id` 不再直接存指针，而是存**请求在池中的下标**：
单请求时 `wr_id = req - base->reqs`，解码 `reqs + (wc->wr_id & 0xff)`；multi-send
时把最多 8 个下标按字节打包 `wr_id += idx << (r*8)`（`ib_plugin.c:1638`），完成时
逐字节取出同组所有发送请求一起递减（`ib_plugin.c:2050-2058`）。

### 3.5 连接元数据与 handle

```c
struct ncclIbQpInfo { uint32_t qpn; struct ibv_ece ece; int ece_supported; int devIndex; };

typedef struct ncclIbDevInfo {      // 每个物理 dev 一份
  uint32_t lid; uint8_t ib_port; enum ibv_mtu mtu;
  uint8_t link_layer, is_global;
  union ibv_gid gid;                // 本端 GID（RoCE 路由 / IB 跨子网 FLID）
  uint32_t fifoRkey;                // 本端 fifo（或 sizesFifo）的 rkey
  union ibv_gid remoteGid;          // 仅本地用：记录对端 GID 供错误日志
} ncclIbDevInfo;

typedef struct ncclIbConnectionMetadata {   // bootstrap 上双向交换的大结构
  struct ncclIbQpInfo qpInfo[NCCL_IB_MAX_QPS];  // 128
  struct ncclIbDevInfo devs[NCCL_IB_MAX_DEVS_PER_NIC]; // 4
  char devName[MAX_MERGED_DEV_NAME];
  uint64_t fifoAddr; int ndevs; int tc; int sl;
} ncclIbConnectionMetadata;

struct ncclIbHandle {               // listen 句柄，≤ NCCL_NET_HANDLE_MAXSIZE(128)
  union ncclSocketAddress connectAddr;
  uint64_t magic;                   // NCCL_SOCKET_MAGIC 校验用
  struct ncclIbCommStage stage;     // 关键：连接状态机随 handle 走（见 §5）
};
```

## 4. 初始化

### 4.1 入口与引用计数

v11 的 `init` 按 communicator 调用，但插件内部只应初始化一次：
`ncclIbInitDevices()`（`ib_plugin.c:694`）用 `ibRefCount` 保证只有一次真正执行
`nccl_p2p_ib_init()`；`ncclIbFinalize()` 递减。每 comm 的 `ctx` 只是一个保存
`trafficClass` 的小配置（`ib_plugin.c:714-726`），用于 SL/TC 默认值（见 §5.4）。
入口还做一组 **32 字节对齐静态断言**（`ib_plugin.c:701-706`）：fifo/sges/wrs 必须
32B 对齐、fifo 元素必须是 32B 倍数——开启 PCI relaxed ordering 后，保证一个
credit 写不会被拆成乱序的两半。

### 4.2 设备枚举（`nccl_p2p_ib_init`，`p2p_plugin.c:526`）

在 v3 流程基础上的主要增强：

1. `ncclFindInterfaces()` 找 OOB 网口（新 socket 层，`socket.h:77`）。
2. `NCCL_IB_HCA` 过滤逻辑不变（`^` 黑名单 / `=` 精确匹配）。
3. 每个设备先判 provider：`wrap_mlx5dv_is_supported()` → mlx5 或 NONE。
4. **Data Direct DMA**（`p2p_plugin.c:594-614`）：mlx5 设备且
   `NCCL_IB_DATA_DIRECT>0` 且 `ncclMlx5dvDmaBufCapable()` 时，查询 data-direct
   sysfs 路径；此类设备被暴露两次——普通 PCIe 设备与 data-direct 设备
   （devName 加 `_dma` 后缀、pciPath 用 data-direct 路径、
   `capsProvider.mlx5.dataDirect=1`）。默认只暴露 `_dma`（devOffset=1），
   `NCCL_IB_DATA_DIRECT=2` 时两个都暴露。
5. 速率换算支持 `active_speed_ex`（NDR 100G/ lane、XDR 200G/ lane；
   `ibv_speeds` 表扩到 9 项、widths 表加 2，`p2p_plugin.c:749-760`）。
6. **AR（自适应路由）**：IB 链路默认 `ar=1`，RoCE 默认 0；
   `NCCL_IB_ADAPTIVE_ROUTING` 可覆盖（默认哨兵 -2，`p2p_plugin.c:75,650-651`）。
7. **SHARP 标记简化**：所有 IB 链路设备都标 `isSharpDev=1` 并把 `maxQp` 压到
   `NCCL_SHARP_MAX_COMMS`（不再按 PCI ID 判 ConnectX-6，`p2p_plugin.c:653-658`）。
8. 每个物理设备一个**异步事件线程**（detached、命名为 `NCCL IbAsync %2d`），
   负责把 fatal 事件计入统计（见 §9）。
9. 每个物理端口自动建一个单设备 vDev（见 §2）。
10. 末尾检测 relaxed ordering 能力（`NCCL_IB_PCI_RELAXED_ORDERING`：0 关 /
    1 强制开但不支持则告警 / 2 自动，默认 2 → `ncclIbRelaxedOrderingEnabled`），
    并打汇总日志 `NET/IB : Using [0]… [RO]; OOB …`。

### 4.3 GID 自动选择（`ncclIbGetGidIndex`，`ib_plugin.c:338`）

- **IB 链路**：优先取 `NCCL_IB_ROUTABLE_FLID_GID_INDEX`（默认 1）且 FLID 非零的
  GID（跨子网路由需要），否则退回 0。
- **RoCE**：`NCCL_IB_GID_INDEX ≥ 0` 时直接用；否则**自动遍历 GID 表**
  （`ncclUpdateGidIndex`，`ib_plugin.c:296`），按三个偏好逐步收敛：地址族
  （`NCCL_IB_ADDR_FAMILY`，默认 AF_INET，识别 IPv4-mapped）、子网前缀
  （`NCCL_IB_ADDR_RANGE`，CIDR 匹配）、RoCE 版本
  （`NCCL_IB_ROCE_VERSION_NUM`，默认 2，读 sysfs `gid_attrs/types`）；跳过未配置
  与 link-local 的 GID。容器里 sysfs 不可读时静默容错（`ib_plugin.c:277-282`）。

### 4.4 PCI 路径合并（`nccl_p2p_ib_pci_path`，`p2p_plugin.c:727`）

realpath 末位清 `'0'` 合并同一 PCI function 的多端口；`NCCL_IB_MERGE_VFS=1`
（默认）时再清两位，把同一物理卡的**多个 VF** 也合并。`realPort` 按
`ncclIbMatchVfPath` 前缀比较累计。NCCL 拓扑据此识别"同一张卡"。

### 4.5 属性上报（`ncclIbGetPhysProperties`，`p2p_plugin.c:302`）

- `ptrSupport`：`HOST` 恒有；nv_peer_mem/nvidia_peermem 三路径探测（见 §4.6）
  成功加 `CUDA`；IB/UCX-UCT 类插件且 DMA-BUF 探测成功加 `DMABUF`。
- `regIsGlobal=1`：声明注册与 comm 无关（MR 缓存 per-device 全局共享，见 §7）。
- `forceFlush=1`：仅 mlx5 data-direct 设备（NCCL 必须走 flush 路径）。
- `maxRecvs=NCCL_NET_IB_MAX_RECVS(8)`（IB/UCX/UCT），其余后端 1。
- `netDeviceType=NCCL_NET_DEVICE_HOST`：不支持 GPU 侧 unpack offload
  （`getDeviceMr/irecvConsumed` 为 NULL）。
- vDev 属性 = 首个物理 dev 属性 + 聚合 name/speed + vProps
  （`nccl_p2p_ib_get_properties`，`p2p_plugin.c:344`）。

### 4.6 GDR 与 DMA-BUF 探测

- GDR：`pthread_once` 检查 `/sys/kernel/mm/memory_peers/nv_mem*/version` 或
  `/sys/module/nvidia_peermem/version`（`p2p_plugin.c:241-255`）。
- DMA-BUF：per-device `pthread_once`，用 `fd=-1` 空调用 `ibv_reg_dmabuf_mr`，
  errno 为 `EOPNOTSUPP/EPROTONOSUPPORT` 判不支持（`EBADF` 说明内核支持），
  结果缓存在 `ibDev->dmaBufSupported`（`p2p_plugin.c:257-300`）。

## 5. 连接建立：可重入状态机

### 5.1 设计动机与状态机

v8+ API 要求 `connect/accept` **绝不阻塞**：没建好就返回 `*sendComm==NULL`，
NCCL 反复调用直到成功。插件把建链过程拆成显式状态机
（`enum ncclIbCommState`，`ib_plugin.c:543`），状态保存在 `ncclIbCommStage`
（state + offset + buffer + comm 指针）里——connect 侧的 stage 放在 **handle 中**
（NCCL 会为同一连接持有一份 handle 副本），accept 侧放在 listenComm 中。
每次进入函数先按 `stage->state` `goto` 到断点续跑；每步用
`ncclSocketProgress`（非阻塞收发，按 offset 累计）推进，没传完就 return，
下次接着传。底层 socket 也是异步的：`ncclSocketConnect` 非阻塞发起，
`ncclSocketReady` 轮询完成（`socket.h:58-87`，含 magic 握手校验与 abortFlag）。

### 5.2 握手时序总览

```
connect 侧（sender）                        accept 侧（receiver）
──────────────────                          ─────────────────
Start: ncclIbMalloc(sendComm)
socketInit(async) + Connect                 Start: ncclIbMalloc(recvComm)
→ Connect: SocketReady 轮询                   socketInit + Accept(listenSock)
─ vProps(dev list) ───────────────►         → Accept: SocketReady 轮询
  (SendDevList/RecvDevList)                 ← vProps(dev list) ─
  双方各算 nqps = max(本地,对端)              ncclIbCheckVProps 求交集+告警
  ×NCCL_IB_QPS_PER_CONNECTION
建 per-dev PD(共享)/CQ；QP 按 dev 轮转创建    收 metadata → 建 per-dev 资源、QP
query_ece 收集 ECE                           set_ece → RTR → RTS → 再 query_ece
─ metadata(qpn/ece/devs/fifoRkey/ ──►        (fifoTc=true 走 NCCL_IB_FIFO_TC)
  fifoAddr/tc/sl) ─  (Send)                  注册 remFifo/sizesFifo MR
→ Connecting: 收对端 metadata ◄──           若 GDR/dmabuf：每 dev 建 gpuFlush 自环 QP
  set_ece → 逐 QP RTR → RTS                  ─ 回传 metadata ─  (Send)
  校验链路类型一致                             → PendingReady: 收 ready
→ Connected: 发 ready ──────────►            *recvComm = rComm
*sendComm = comm
```

### 5.3 关键细节

- **QP 数量与条带**：`nqps = max(localNdevs, remoteNdevs) × NCCL_IB_QPS_PER_CONNECTION`
  （默认每 dev 1 QP，`ib_plugin.c:728`）。QP 在本地 dev 上轮转创建
  （`devIndex = (devIndex+1) % ndevs`），第 q 个 QP 的对端设备下标
  `remDevIdx = remMeta.qpInfo[q].devIndex`——两端按同样规则条带化，天然对齐。
- **ECE**（enhanced connection establishment，`NCCL_IB_ECE_ENABLE` 默认 1）：
  connect 侧 `ibv_query_ece` 随 qpInfo 发出；accept 侧 RTR 前 `ibv_set_ece`，
  成功后再 `query_ece` 取协商结果回传；connect 侧最后 `set_ece` 收尾
  （`ib_plugin.c:974-979, 1336-1353, 1106-1107`）。
- **MTU 协商**：`MIN(对端 mtu, 本端口 active_mtu)`（`ib_plugin.c:1110`）。
- **SL/TC 优先级**（`ib_plugin.c:1041-1042`）：`NCCL_IB_SL/NCCL_IB_TC`（≠-1 时）
  > v11 comm config 的 `trafficClass` > 默认 0。credit 类流量可单独用
  `NCCL_IB_FIFO_TC`（`ncclIbRtrQp` 的 `fifoTc` 参数，接收端建 QP 时传 true）。
- **RTR 寻址**（`ncclIbRtrQp`，`ib_plugin.c:797`）：RoCE 一律 GRH（dgid=对端
  GID，sgid_index=本地选定 GID，hop_limit=255）；IB 先比子网前缀
  （`ncclIbExtractLocalSubnetPrefix` 取 subnet_prefix 低 16 位），同子网用 dlid，
  跨子网或 `is_global`（`NCCL_IB_IS_GLOBAL` 或端口 `IBV_QPF_GRH_REQUIRED`）时
  加 GRH 并从对端 GID 提取 **FLID** 做路由（FLID 为 0 告警回退 dlid）。
- **RTS**：`timeout = NCCL_IB_TIMEOUT`（默认 20）、`retry_cnt = NCCL_IB_RETRY_CNT`
  （7）、`rnr_retry=7`（`ncclIbRtsQp`，`ib_plugin.c:845`）。
- **链路类型一致性**：本地多 dev 之间、本地与对端之间都要求 IB/RoCE 同构，
  否则报错并提示用 `NCCL_IB_HCA` 过滤（`ib_plugin.c:1030-1036, 1068-1079`）。
- **fifo 注册**：connect 侧把整块 `comm->fifo`（64×8 个槽）按每 dev 注册
  （rkey 放进 `meta.devs[i].fifoRkey`，`ib_plugin.c:994`）；accept 侧注册
  `remFifo.elems`（credit staging）与 `sizesFifo`（其 rkey 经 `meta.devs[i].fifoRkey`
  回传，`ib_plugin.c:1408-1409`），`meta.fifoAddr` 双向各指各的——发送方指自己的
  credit fifo，接收方指自己的 sizesFifo。

## 6. 内存注册与 MR 缓存

### 6.1 每设备 MR 缓存（`ncclIbRegMrDmaBufInternal2`，`ib_plugin.c:1482`）

v3 每次 `regMr` 都直接 `ibv_reg_mr`；master 在**物理设备**上维护缓存
（`ncclIbDevs[].mrCache`，按地址排序的槽数组 + 引用计数，配合 `regIsGlobal=1`
声明）：

- 地址向页对齐取整，按"页区间覆盖"查找：新注册区间若被已有槽完全覆盖，直接
  `refs++` 复用，不做系统调用。
- 未命中时在有序位置插入新槽（容量 32 起步倍增），注册 flags 恒含
  `LOCAL_WRITE|REMOTE_WRITE|REMOTE_READ|REMOTE_ATOMIC`。
- **relaxed ordering**：全局 `ncclIbRelaxedOrderingEnabled` 且调用方未带
  `NCCL_NET_MR_FLAG_FORCE_SO` 时追加 `IBV_ACCESS_RELAXED_ORDERING`，并改用
  IBVERBS 1.8 的 `ibv_reg_mr_iova2`（显式 iova，`ib_plugin.c:1510-1516`）。
- **DMA-BUF 路径**（`regMrDmaBuf`，fd≠-1）：普通设备走 `ibv_reg_dmabuf_mr`；
  mlx5 data-direct 设备走 `mlx5dv_reg_dmabuf_mr`（带 `mlx5_access=1`，
  `ib_plugin.c:1502-1508`）。
- `deregMr` 反向：`refs--` 到 0 才真正 `ibv_dereg_mr`，槽用"与末尾交换"删除，
  缓存空了就整体释放（`ib_plugin.c:1578-1602`）。

### 6.2 comm 级封装

同一段 buffer 要在 comm 的**每个物理 dev 的 PD** 上各注册一次（多轨各自 DMA），
`mhandle` 因此是 `ncclIbMrHandle { struct ibv_mr* mrs[4] }`（`ib_plugin.c:618`）。
`ncclIbGetNetCommDevBase()`（`ib_plugin.c:1541`）按 `base->isSend` 在
send/recv comm 的不同偏移处取 per-dev base。

## 7. 数据路径

### 7.1 模型总览

核心仍是 v3 的 **credit-based 单边 RDMA 写**：数据不经双边 SEND，接收端发布
credit（地址+rkey+tag+序号），发送端凭 credit 直接 `RDMA_WRITE(_WITH_IMM)` 进
接收缓冲。master 在三个维度上扩展：

1. **multi-recv**：一次 `irecv` 最多带 `NCCL_NET_IB_MAX_RECVS(8)` 组
   （buffer,size,tag），对应 NCCL 的 grouped receive；credit 槽变成
   `fifo[64][8]` 的二维结构，同一 slot 内 8 个 elem 共享一个 `idx` 序号。
2. **多轨**：每个物理 dev 一套 CQ/PD/MR，rkey 按对端 dev 数组下发；
   数据按 dev/QP 条带化。
3. **多 QP**：最多 128 个 RC QP 轮转使用，流控深度
   `MAX_REQUESTS=64`（`max_send_wr=2*64`，因为一条消息最多 2 个 WR：
   数据 WRITE + 收尾 WRITE_WITH_IMM，`ib_plugin.c:779`）。

### 7.2 irecv：挂接收 + 发 credit

`ncclIbIrecv(recvComm, n, data[], sizes[], tags[], mhandles[], …)`（`ib_plugin.c:1889`）：

1. 取请求 `req`（type=RECV，nreqs=n），per-dev `devBases` 预填。
2. **挂空 recv WR**（`sg_list=NULL`）：在 `nDataQps`（= `max(本地ndevs, 对端ndevs)`；
   `NCCL_IB_SPLIT_DATA_ON_QPS=1` 时为全部 nqps）个 QP 上各挂一个，轮转
   `qpIndex`，并给每个 QP 的 dev 计一个 event。这些 WR 不收数据，只承接
   `RDMA_WRITE_WITH_IMM` 产生的 CQE（每个数据 QP 一条 imm，见 §7.4）。
3. `ncclIbPostFifo()`（`ib_plugin.c:1809`）发 credit：
   - slot = `fifoTail % 64`；`req->recv.sizes = comm->sizesFifo[slot]` 并清零
     （接收完成时的字节数由发送端写到这里，见 §7.4）；
   - 填 `localElem[i] = {addr, size, tag, nreqs=n, idx=fifoTail+1,
     rkeys[j]=每个本地 dev 的 MR rkey}`；
   - 选 CTS QP：按 `base.devIndex` 轮转（QP 本来就按 dev 条带创建，所以
     devIndex 即 QP 下标），用 `remDevs[ctsQp->remDevIdx].fifoRkey` 把
     n 个 elem 一次 `IBV_WR_RDMA_WRITE` 到对端 `fifo + slot*8*sizeof(elem)`；
   - **非 signaled + 定期排水**：默认不带 `IBV_SEND_SIGNALED`（省 CQE），仅当
     `slot == ctsQp->devIndex` 时升级为 signaled 并计入 req 的 event——
     注释（`ib_plugin.c:1852-1874`）引用了 unsignaled completion 的经典陷阱：
     全 unsignaled 会让 SQ 永远无法排空，必须周期性产生 WC。该条件保证每个
     CTS QP 每轮都被排水一次；
   - `NCCL_IB_USE_INLINE=1` 时 credit 走 inline（QP 创建时
     `max_inline_data = sizeof(ncclIbSendFifo)`，`ib_plugin.c:783`）。

### 7.3 isend：等 credit、按 tag 匹配

`ncclIbIsend(sendComm, data, size, tag, mhandle, phandle, &req)`（`ib_plugin.c:1724`）：

1. 入口校验 `ready` 与致命错误计数（§9）。
2. slot = `fifoHead % 64`；**等整组 credit 到齐**：`slots[0].idx != fifoHead+1`
   就对 NCCL 返回 `*request=NULL` 下次再来；`nreqs>1` 时自旋等其余 elem 的
   `idx` 也都等于 `fifoHead+1`（网卡 DMA 是逐个写的），随后
   `__sync_synchronize()` 保证后续字段读取顺序。
3. **tag 匹配**：遍历组内找 `reqs[r]==NULL && slots[r].tag==tag` 的 elem；
   找不到同样返回 NULL。发送 size 收敛为 `min(size, 接收端 size)`——v3 的
   "size 不一致直接报错"在这里变为**截断**（并对非法 credit 打 WARN，
   `ib_plugin.c:1748-1756`）。
4. 建发送请求：type=SEND，记录 size/data/offset=0，预存每个 dev 的 lkey；
   按将要使用的 QP 集合（从当前 `qpIndex` 起 `nDataQps` 个）给对应 dev 计
   event（`ib_plugin.c:1767-1785`）。
5. **组齐才发**：该 elem 记入 `fifoReqs[slot]` 后，若组内还有 NULL（multi-recv
   的其他 tag 还没被 isend 命中），直接返回——等所有兄弟请求都匹配上，才由
   最后一员调用 `ncclIbMultiSend()` 一次性发出，清槽、`fifoHead++`。

### 7.4 ncclIbMultiSend：WR 链、条带化与 AR

`ncclIbMultiSend(comm, slot, phandle)`（`ib_plugin.c:1620`）：

1. **组 WR 链**：组内每个请求一个 `IBV_WR_RDMA_WRITE`（数据），`wr->next`
   串链；最后一个 WR 换成 `IBV_WR_RDMA_WRITE_WITH_IMM` + `IBV_SEND_SIGNALED`，
   `wr_id` 打包全组请求下标（§3.4）。imm_data 规则：
   - `nreqs==1`：imm = 消息 size（接收端 `sizes[0]` 直接取自 imm）；
   - `nreqs>1`：imm=0，真实 sizes 数组先写入本地 `remSizesFifo.elems[slot]`，
     由最后一个 WR 一并 RDMA 到对端 `sizesFifo[slot]`（remote_addr 指过去）。
2. **AR 大消息拆分**（`ib_plugin.c:1653-1666`）：连接启用 AR 且
   `size > NCCL_IB_AR_THRESHOLD`（默认 8192）时，数据 WR 保持普通
   `RDMA_WRITE`（不 Signaled），另加一个 0 字节 `RDMA_WRITE_WITH_IMM` 收尾——
   自适应路由下包序不保证，把"完成通知"与"数据到达"解耦，0 字节 imm 包一定
   在数据之后触发接收端 CQE。
3. **多 QP 条带化**：外层循环 `nqps` 次（默认 nDataQps，`IB_SPLIT_DATA_ON_QPS=1`
   时全部 QP），每次取 `qpIndex` 对应 QP：每个请求的 chunk =
   `DIVUP(DIVUP(size, nqps), 128)×128`——**128 字节对齐切条**，保证 LL/LL128
   协议的字节序假设；按 `qp->remDevIdx` 选对端 rkey、按 `qp->devIndex` 选本地
   lkey，整条 WR 链 post 到该 QP；然后推进各请求的 offset/地址，
   `qpIndex` 轮转（`ib_plugin.c:1673-1719`）。多轨时不同 QP 物理上就走不同网卡。

发送端完成：每个数据 QP 上的最后一个（signaled）WR 产生一个 WC，test 中按
wr_id 解包，组内所有发送请求在该 dev 的 event 各减一。

### 7.5 test：事件计数 + 按 dev 轮询

`ncclIbTest()`（`ib_plugin.c:1976`）：

1. 若 `events[0..3]` 全 0 → 完成：RECV 返回 `sizes[]`（nreqs==1 时来自
   imm_data，见 §7.4；nreqs>1 时来自 `req->recv.sizes` 指向的 `sizesFifo`
   槽），SEND 返回 `sizes[0]=send.size`，释放请求。
2. 否则对每个 `events[i]>0` 的 dev，在其 CQ 上 `ibv_poll_cq`（一次最多 4 个
   WC），逐 WC 处理：
   - 错误 WC：打详细日志（对端地址、status/opcode 字符串化、reqSize、
     vendor_err、请求类型、RoCE 时的 local/remote GID、HCA 名），返回
     **`ncclRemoteError`**（master 新增的错误码语义）；
   - SEND 请求的 WC：按 wr_id 每字节解出一个同组请求，`events[i]--`；
   - `IBV_WC_RECV_RDMA_WITH_IMM`：`nreqs==1` 时 `sizes[0]=imm_data`，
     `events[i]--`；
   - credit 的 signaled WC：同样 `events[i]--`（它在 postFifo 时计过数）。
3. 每个 dev 的 CQ 处理完后检查该物理 dev 的异步致命错误计数（§9）。
4. 本轮没有任何 CQE 则返回未完成，NCCL 下次再调。

### 7.6 iflush

`ncclIbIflush(recvComm, n, data[], sizes[], mhandles[], &req)`（`ib_plugin.c:1937`）：
找最后一个非零 size 的 buffer（同组只需 flush 一次），对 comm 的**每个物理 dev**
在其 gpuFlush 自环 QP 上发 1 字节 `IBV_WR_RDMA_READ`（remote 是 GPU 显存，
本地是 host dummy 页），各计一个 event；返回请求由 NCCL 走 `test` 等待——
v3 是函数内忙等，master 改成了**异步 iflush**，不再阻塞 NCCL 进度。
`flushEnabled` 条件：GDR 或 DMA-BUF 可用，且 `NCCL_GDR_FLUSH_DISABLE=0`
（`ib_plugin.c:1356`）；自环 QP 每 dev 一个，accept 时建好（`ib_plugin.c:1370-1390`）。
原理同 v3：RDMA READ 的完成迫使先前到达的 PCIe 写对 GPU 可见。

## 8. 错误处理与异步事件

master 把"网卡异步致命错误"接入了数据面：

- QP 的 `qp_context` 与 CQ 的 `cq_context` 都指向 `comm->base.stats`（
  `ib_plugin.c:774, 752`）。
- 异步事件线程（`ncclIbAsyncThreadMain`，`p2p_plugin.c:377`）按事件类型分发：
  `DEVICE_FATAL` → dev stats；`CQ_ERR` → 该 CQ 的 comm stats；
  `QP_FATAL/REQ_ERR/ACCESS_ERR` → 该 QP 的 comm stats；其余非致命事件只告警；
  最后才 ack（避免 use-after-free）。计数用 `__atomic_fetch_add`。
- `ncclIbStatsCheckFatalCount()`（`ib_plugin.c:66`）在 isend/irecv/test 入口及
  test 内层检查：一旦发现 fatal 计数非零，返回 `ncclSystemError` 让通信器进入
  错误流程，并**停止继续 poll 以压制错误刷屏**。
- WC 错误返回 `ncclRemoteError`，日志包含 GID/HCA/请求类型等定位信息；
  status/opcode 有完整的字符串化 helper（`ibvWcStatusStr/ibvWcOpcodeStr`）。

## 9. 资源释放

- `ncclIbCloseSend/Recv`（`ib_plugin.c:2083-2124`）：关 socket → 逐 QP
  `destroy_qp` → 逐 dev 注销 fifoMr/remSizesFifo.mrs（recv 侧还有 gpuFlush
  QP/hostMr、sizesFifoMr）→ `ncclIbDestroyBase`：destroy CQ 并给共享 PD
  `pdRefs--`，归零才 `dealloc_pd`（`ib_plugin.c:757-769`）。
- `ncclIbCloseListen`：关 listen socket。
- MR 缓存随进程存活（per-device），不随 comm 释放；`ncclIbFinalize` 只减
  `ibRefCount`（设备资源实际不回收，进程退出时由内核清理）。

## 10. 环境变量速查（master）

| 变量 | 默认 | 作用 |
|---|---|---|
| `NCCL_PLUGIN_P2P` | `ib` | 后端选择：ib / ucx / ucx_rma / ucx_uct / ucx_uct_read |
| `NCCL_IBEXT_DISABLE` | 0 | 禁用 IB 插件 |
| `NCCL_IB_HCA` | 全收 | 网卡:端口过滤（`^` 黑名单，`=` 精确） |
| `NCCL_IB_GID_INDEX` | -1 | RoCE GID 下标；-1 = 自动选择 |
| `NCCL_IB_ADDR_FAMILY` / `NCCL_IB_ADDR_RANGE` | AF_INET / 无 | GID 自动选择的地址族 / 子网前缀 |
| `NCCL_IB_ROCE_VERSION_NUM` | 2 | GID 自动选择的 RoCE 版本偏好 |
| `NCCL_IB_ROUTABLE_FLID_GID_INDEX` | 1 | IB 跨子网 FLID 路由的 GID 下标 |
| `NCCL_IB_IS_GLOBAL` | 0 | 强制 GRH |
| `NCCL_IB_TIMEOUT` / `NCCL_IB_RETRY_CNT` | 20 / 7 | QP RTS 超时 / 重传 |
| `NCCL_IB_PKEY` | 0 | pkey 下标 |
| `NCCL_IB_SL` / `NCCL_IB_TC` | -1 / -1 | SL / TC（-1 时用 comm config 或 0） |
| `NCCL_IB_FIFO_TC` | 0 | credit（fifo）流量的独立 TC |
| `NCCL_IB_USE_INLINE` | 0 | credit 写走 inline |
| `NCCL_IB_QPS_PER_CONNECTION` | 1 | 每 dev 的 QP 数（总 QP = 该值 × dev 数取双端最大） |
| `NCCL_IB_SPLIT_DATA_ON_QPS` | 0 | 1 = 数据条带化用全部 QP（默认只用 nDataQps） |
| `NCCL_IB_ADAPTIVE_ROUTING` | IB 开 RoCE 关 | AR 开关（-2 未设置时按链路自动） |
| `NCCL_IB_AR_THRESHOLD` | 8192 | AR 下走"数据+0字节imm"拆分的消息阈值 |
| `NCCL_IB_PCI_RELAXED_ORDERING` | 2 | 0 关 / 1 强制 / 2 自动 |
| `NCCL_IB_MERGE_NICS` / `NCCL_IB_MERGE_VFS` | 1 / 1 | 允许 makeVDevice 融合 / PCI 路径合并 VF |
| `NCCL_IB_DATA_DIRECT` | 1 | mlx5 data-direct 设备暴露策略（2=两种都暴露） |
| `NCCL_IB_ECE_ENABLE` | 1 | ECE 协商 |
| `NCCL_IB_RETURN_ASYNC_EVENTS` | 1 | 异步 fatal 事件接入数据面错误返回 |
| `NCCL_GDR_FLUSH_DISABLE` | 0 | 关闭 GDR flush |
| `NCCL_IB_WARN_RAIL_LOCAL` | 0 | 双端物理 dev 不匹配时告警 |
| `NCCL_SHARP_MAX_COMMS` | 1 | SHARP 设备的 maxComms |

## 11. 设计要点小结

1. **一份实现，六张 API**：v11 原生实现 + 逐层 `_vN` 签名适配，同一份 .so 服务
   不同年代的 NCCL。
2. **两级设备模型**：物理 dev 管资源（共享 PD、MR 缓存、统计），vDev 面向
   NCCL 拓扑与融合（rail），comm 在 vDev 上建、资源落在每个物理 dev 上。
3. **credit FIFO 二维化 + tag**：`fifo[64][8]` 支持 grouped multi-recv；`idx`
   序号替代 v3 的 ready 标志；tag 匹配解耦收发顺序；组齐才发（one shot）。
4. **事件计数完成模型**：`events[dev]` 统一度量"还欠几个 CQE"，send/recv/
   flush/credit-signaled 共用一套 test 逻辑；wr_id 按字节打包请求下标。
5. **多 QP 条带 + AR 感知**：默认每 dev 一 QP 轮转、128B 对齐切条；AR 下
   "数据不 signaled + 0 字节 imm 收尾"规避乱序；credit 默认 unsignaled +
   周期性 signaled 排水。
6. **连接建立零阻塞**：显式状态机 + stage 随 handle/listenComm 走 + 异步
   socket，任意时刻可重入续跑；ECE/FLID/GID 自动选择把 RoCE/跨子网配置
   自动化。
7. **错误可观测、可传播**：异步线程把 QP/CQ/dev fatal 事件经 context 指针
   计入各 comm 统计，数据面入口检查；WC 错误带完整上下文并区分
   `ncclRemoteError`。

## 12. 附 A：一次完整收发时序（单轨单 QP 简化）

```
接收端 irecv(buf,size,tag)                发送端 isend(data,size,tag)
────────────────────────                  ─────────────────────────
post_recv（空 WR，接 imm）
postFifo: RDMA_WRITE ─ credit{addr,rkeys, ──► 本地 fifo[slot]
                              tag,idx=head+1}
                                        轮询 slots[0].idx==head+1
                                        tag 匹配 → 建 SEND req
                                        RDMA_WRITE_WITH_IMM ─数据──► buf
                                        (imm=size)
test: poll 发送 CQ → events 清零 → done
                                        test: poll 接收 CQ → RECV_RDMA_WITH_IMM
iflush（GDR 时）:                          sizes[0]=imm → events 清零 → done
  自环 QP RDMA_READ 1B（异步，NCCL test 等）
```

multi-recv（n>1）时：credit 一次写 n 个 elem；发送端等全组 tag 都命中后
WR 链一次发出，sizes 数组经 sizesFifo 送达；多 QP 时每 QP 一份 128B 对齐
的条带 + 每 QP 一条 imm。

## 13. 附 B：与 v3 时代（hpcx-v2.7.0）的关键差异

| 维度 | v3（tag hpcx-v2.7.0） | master |
|---|---|---|
| 导出符号 | `ncclNetPlugin_v3` 一个 | v6–v11 六个，结构体替换分发 |
| 设备模型 | 一卡一 dev | 物理 dev + vDev 融合（≤4/vDev） |
| QP | 每 comm 1 个（+flush 1 个） | 最多 128 个，按 dev 条带 |
| 请求池 | 128，done 标志 | 64，events[4] 计数；wr_id 下标打包 |
| 接收 | 单 buffer | grouped multi-recv（8，带 tag） |
| 大小不一致 | 报错 | 截断 + WARN |
| sizes 回传 | imm_data | imm（n=1）或 sizesFifo RDMA（n>1） |
| 建链 | 两段式惰性 ready | 全状态机非阻塞，connect/accept 可重入 |
| RoCE GID | 固定下标（默认 0） | 自动遍历（地址族/前缀/版本） |
| IB 路由 | dlid only | 跨子网 FLID + GRH |
| MR | 每 comm 直接注册 | per-device 缓存 + 引用计数 + DMA-BUF |
| flush | 同步忙等 | 异步 iflush 请求 |
| 错误 | WARN + ncclSystemError | fatal 统计传播、ncclRemoteError、GID 日志 |
| AR/ECE/data-direct | 无 | 有 |
