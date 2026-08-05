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

## 14. 附 C：`ncclIbIsend` / `ncclIbMultiSend` 逐段精讲

§7 给出了数据路径的整体模型，本附录对发送侧这两个函数做逐行级拆解
（`ib_plugin.c:1620-1807`）。

### 14.1 分工与三个游标

- `ncclIbIsend`：**匹配阶段**——等接收端 credit、按 tag 认领槽位、建请求；
  一组请求全部认领完才调 `ncclIbMultiSend`。
- `ncclIbMultiSend`：**发射阶段**——把整组请求组装成 WR 链，按多 QP 切条 post。

贯穿始终的三个游标：

- `fifoHead`：单调递增的**组序号**，第 G 组消息用 `slot = G % 64` 这一行 fifo；
- `qpIndex`：本条消息从哪个 QP 开始切条，每条消息发完轮转 +1（跨消息负载均衡）；
- `fifoReqs[slot]`：这一行 8 个位置存放"已认领的发送请求"指针，初始全 NULL。

fifo 是二维的 `fifo[64][8]`：64 个 slot（组）× 每组最多 8 个 elem（一次 grouped
irecv 的 n 个 buffer）。接收端 `postFifo` 把一组 elem 一次 RDMA 写到 `fifo[slot]`，
所有 elem 的 `idx` 都写成 `fifoTail+1`。

### 14.2 `ncclIbIsend` 逐段

**(1) 等 credit**（`ib_plugin.c:1736-1740`）：

```c
int slot = comm->fifoHead % MAX_REQUESTS;
uint64_t idx = comm->fifoHead + 1;
if (slots[0].idx != idx) { *request = NULL; return ncclSuccess; }
```

`idx` 是"第 fifoHead 组应有的序号"。对端还没 post 对应 irecv 时 `slots[0].idx`
是旧值，返回 NULL 让 NCCL 重试——流控只是读本地内存（credit 由对端网卡 DMA
进来），零系统调用。

**(2) 自旋等全组到齐**（`ib_plugin.c:1741-1744`）：

```c
nreqs = slots[0].nreqs;
for (int r=1; r<nreqs; r++) while(slots[r].idx != idx);
__sync_synchronize();
```

对端用一条 RDMA_WRITE 写 n 个 elem，但字节陆续落盘，`slots[0]` 到了不代表
`slots[n-1]` 到了。两个设计细节：

- `idx` 是 elem 结构体的**最后一个字段**（offset 40；elem 共 64 字节、32B 对齐，
  即 init 中那组静态断言的目的）。写按序落盘，看到 `idx` 就位即表示前面的
  addr/size/rkeys/tag 已就位；`__sync_synchronize()` 补上 CPU 侧内存序。
- 对 `slots[0]` 是"返回 NULL 重试"，对 1..n-1 却是**死等**：`nreqs` 本身存在
  `slots[0]` 里，`slots[0].idx` 正确后才知道要再等几个；对端 credit 已发出，
  剩余字节马上就到，自旋有界。

**(3) tag 匹配**（`ib_plugin.c:1745-1746`）：

```c
for (int r=0; r<nreqs; r++) {
  if (reqs[r] != NULL || slots[r].tag != tag) continue;
```

grouped recv 一次 post n 个 (buffer, tag)，发送端逐条 isend，两边顺序可以不同。
每次在组内找"未被认领（`reqs[r]==NULL`）且 tag 相等"的 elem；找不到返回 NULL
重试。认领后 `size = min(size, slots[r].size)`（截断语义，§7.3）。

**(4) events 预排**（`ib_plugin.c:1767-1780`，最易混的一段）：

```c
int nEvents = nDataQps;              // 本条消息将经过几个 QP
int qpIndex = comm->base.qpIndex;    // 局部副本！
while (nEvents > 0) {
  ncclIbQp* qp = comm->base.qps + qpIndex;
  ncclIbAddEvent(req, qp->devIndex, &comm->devs[qp->devIndex].base); // events[dev]++
  nEvents--;
  qpIndex = (qpIndex+1) % comm->base.nqps;   // 不动 comm->base.qpIndex
}
```

send 请求的完成条件是"每个承担数据的 QP 都回报一个 signaled CQE"。这里**预先
走一遍** MultiSend 将要使用的 QP 序列并计数。注释明确：此时不能推进
`comm->base.qpIndex`，因为 MultiSend 还要从同一起点走一遍——两处必须严格一致，
否则 events 计在 dev A、CQE 却落在 dev B 的 CQ 上，请求永远无法完成。
（events 按 dev 而非按 QP 计，是因为 CQ 是 per-dev 的；同一 dev 上两个 QP 时
`events[dev]` 自然计 2。）

**(5) 组齐才发**（`ib_plugin.c:1787-1801`）：

```c
*request = reqs[r] = req;
for (int r=0; r<nreqs; r++) if (reqs[r] == NULL) return ncclSuccess;
```

把请求挂进 `fifoReqs[slot][r]` 后检查组内有无空位：有就直接返回。注意此时
`*request` 已交给 NCCL，但**网线上一个包都没发**——该请求将由"填补最后一个
空位的那次 isend"触发的 MultiSend 一并发出、一并完成。因此 test() 一个早已
返回的请求长时间不完成是正常现象。

组齐后：调 MultiSend → `memset(slots[0])` 清首个 elem（复位 nreqs/idx 防旧值
误判，就绪判断只看 `slots[0].idx`）→ 清 `fifoReqs[slot]` → `fifoHead++`。

### 14.3 `ncclIbMultiSend` 逐段

**(1) 组 WR 链 + wr_id 打包**（`ib_plugin.c:1626-1639`）：每个请求一个
`IBV_WR_RDMA_WRITE`（数据），`next` 串链；`wr_id` 把组内各请求在请求池中的
**下标按字节打包**（第 r 字节 = 第 r 个请求下标）。完成时 test() 一条 CQE 即可
解出全组请求逐个 `events[i]--`——这是请求池 ≤256 的原因。

**(2) size 的两种回传**（`ib_plugin.c:1643-1651`）：

- `nreqs==1`：`immData = size`，随 WRITE_WITH_IMM 的 imm_data 送达，接收端
  `sizes[0]` 直接取自 WC；
- `nreqs>1`：imm 只有 32 位不够用。各请求 size 先写入本地 staging
  `remSizesFifo.elems[slot]`，由追加的 WR 一并 RDMA 到**接收端 `sizesFifo[slot]`**
  （建链时其 rkey/addr 已回传）；接收端 irecv 时已把 `req->recv.sizes` 指向
  `sizesFifo[slot]`，DMA 落入后 test 即可读出。此时 imm=0。

**(3) 末尾 WR 的三种形态**（`ib_plugin.c:1653-1671`）：

- **单请求普通**：`wrs[0]` 自身升级为 WRITE_WITH_IMM（数据+imm 一条 WR），
  signaled；
- **多请求**：追加第 nreqs+1 个 WR，负载为 sizes 数组（写对端 sizesFifo），
  opcode WRITE_WITH_IMM，imm=0；
- **AR + 大消息**（> `NCCL_IB_AR_THRESHOLD`，默认 8192）：数据 WR 保持普通
  RDMA_WRITE 且**不 signaled**，追加 0 字节 WRITE_WITH_IMM（imm=size）收尾。
  AR 下包可走不同路径，把"完成通知"独立成小尾巴，接收端按 PSN 顺序产出 CQE
  时数据必然已全部落盘；发送端每个 QP 也只产生一个 CQE。

**(4) 多 QP 切条循环**（`ib_plugin.c:1673-1719`，核心）：

```c
for (int i = 0; i < nqps; i++) {          // 默认 nqps = nDataQps
  ncclIbQp* qp = comm->base.qps + comm->base.qpIndex;
  for (int r=0; r<nreqs; r++) {
    wrs[r].wr.rdma.rkey = slots[r].rkeys[qp->remDevIdx];   // 对端那个 dev 的 rkey
    int chunkSize = DIVUP(DIVUP(size, nqps), 128) * 128;   // 每条 128B 对齐
    int length = MIN(size - offset, chunkSize);            // 末 QP 拿短尾巴
    sges[r] = { addr+offset, length, lkeys[devIndex] };    // 本地这个 dev 的 lkey
  }
  wrap_ibv_post_send(qp->qp, comm->wrs, &bad_wr);          // 整条链挂这个 QP
  for (r...) { offset += chunkSize; sge.addr += chunkSize; remote_addr += chunkSize; }
  comm->base.qpIndex = (qpIndex+1) % comm->base.nqps;      // 下条消息换起始 QP
}
```

每圈改三处后重 post 同一条链：**rkey 按 `remDevIdx` 选**（接收端把同一 buffer
在其每张卡上各注册一遍，credit 携带全部 rkey）、**lkey 按 `devIndex` 选**（本地
同理）、**SGE/远端地址推进一个 chunk**。128B 对齐是因为 NCCL 的 LL/LL128 协议
按固定粒度在数据流里找 flag，条带边界不能切在 flag 中间。`length<=0` 时
`num_sge=0` 仍 post（0 字节 WR 合法，且 nreqs==1 时它本身就是带 imm 的收尾 WR）。

### 14.4 完整算例

设 `nDataQps=2`（QP0 在 dev0、QP1 在 dev1），接收端 grouped irecv 两个 buffer：
tag=11 大小 10000，tag=22 大小 300；设 fifoHead=5（slot=5）。

1. 接收端 postFifo 写 `fifo[5]`：elem0={addrA, 10000, rkeys, tag=11, nreqs=2,
   idx=6}，elem1={addrB, 300, rkeys, tag=22, nreqs=2, idx=6}。
2. `isend(dataX, 300, tag=22)`：`slots[0].idx==6` 通过；自旋等 `elem1.idx==6`；
   tag 命中 elem1 → 建 req_b 放入 `fifoReqs[5][1]`，events 预排（dev0+1、dev1+1）；
   `fifoReqs[5][0]` 仍为 NULL → **返回，未发任何数据**。
3. `isend(dataY, 12000, tag=11)`：命中 elem0，size 截断为 10000；req_a 放入
   `fifoReqs[5][0]`；组齐 → MultiSend：
   - wrs[0]=WRITE(dataY→addrA)，wrs[1]=WRITE(dataX→addrB)，追加 wrs[2]=
     WRITE_WITH_IMM（负载 sizes{10000,300} → 对端 sizesFifo[5]，imm=0，
     signaled），wr_id 两个字节分别为两请求下标；
   - chunkSize(req_a)=DIVUP(DIVUP(10000,2),128)×128=5120；chunkSize(req_b)=
     DIVUP(150,128)×128=256；
   - QP0：req_a 发 [0,5120)，req_b 发 [0,256)，post 链，offset 推进 5120/256；
   - QP1：req_a 发 MIN(10000-5120,5120)=4880 字节 [5120,10000)，req_b 发
     MIN(300-256,256)=44 字节 [256,300)，post 链。
4. 每 QP 仅 wrs[2] signaled → 发送端 2 条 CQE（dev0、dev1 的 CQ 各一）；test 解
   wr_id 两个字节，req_a、req_b 的 events 各递减至全零 → 两请求均 done。
5. 接收端收到 2 条 RECV_RDMA_WITH_IMM（imm=0），sizes 从 `sizesFifo[5]` 读出
   {10000, 300}。

### 14.5 易混点备忘

- **events 预排与 MultiSend 的 QP 序列**：isend 用 `qpIndex` 局部副本预走，
  MultiSend 从 `comm->base.qpIndex` 实走并推进——序列必须一致（§14.2 (4)）。
- **request 早返回 ≠ 已发送**：组内最后一个空位被填补的那次 isend 才真正发射
  全组（§14.2 (5)）。
- **`if (ready == 0)` 出现两次**（`ib_plugin.c:1726-1727`）：第一次 WARN+报错，
  第二次是不可达的历史遗留——现行语义是 NCCL 拿到非 NULL comm 后才准调 isend，
  ready==0 属内部错误。
- **`pHandle` 参数**：v11 给 profiler 的 proxy handle，IB 后端透传未用。
- **`slots[0]` 的 memset 只清一个 elem**：就绪判断只看 `slots[0].idx`，该 slot
  下次复用时对端会整组重写；清零只为防旧值误判与调试。

## 15. 附 D：tag 的含义、产生与匹配机制

### 15.1 tag 匹配指的是什么

插件内部两处配合（`src/ib_plugin.c`）：

- 接收端 post credit 时把 NCCL 给的 tag 写进每个 elem：
  `localElem[i].tag = tags[i]`（`ib_plugin.c:1833`）；
- 发送端 isend 在当前 fifo 组内找"**未被认领且 tag 相等**"的 elem：
  `if (reqs[r] != NULL || slots[r].tag != tag) continue;`（`ib_plugin.c:1746`）。

找不到匹配就返回 `*request = NULL`，NCCL 稍后重试。匹配的作用是去复用
（demux）：**一条 IB 连接上混跑多条逻辑数据流时，tag 决定这条消息该落进哪个
buffer**。插件从头到尾不理解 tag 的语义，只做相等比较。

### 15.2 tag 的含义：发送方 rank

tag 的语义是"**消息的发送者是谁**"，具体值是发送 rank 的 `topParentRank`
（无 PXN 时即为 communicator 内的 rank 号）：

- 发送方给每条消息盖章"我是谁"：`isend(..., resources->tpRank, ...)`
  （NCCL `src/transport/net.cc:1414`）；
- 接收方给每个 buffer 盖章"我等谁"：`tags[subCount] = resources->tpRemoteRank`
  （net.cc:1575）。

两端一致，故能匹配。

### 15.3 tag 是怎么产生的

tag 不是插件产生的，而是 NCCL 核心在三个环节生成传递（以下行号取自 NCCL
master 的 `src/transport/net.cc`）：

1. **建连时**（`sendSetup`/`recvSetup`，net.cc:318/371）：
   `req.tpRank = comm->topParentRanks[myInfo->rank]`（本端 rank）、
   `req.tpRemoteRank = comm->topParentRanks[peerInfo->rank]`（对端 rank），
   存入该连接的 proxy resources。
2. **发送推进时**（`sendProxyProgress`，net.cc:1414）：每次 isend 用本连接的
   `resources->tpRank` 作 tag。
3. **接收推进时**（`recvProxyProgress`，net.cc:1525-1590）：grouped irecv 组装
   一组 sub 请求，**每个 sub 用它自己连接的 `tpRemoteRank`** 填
   `tags[subCount]`，整组一并传给插件。

### 15.4 为什么需要 tag

插件向 NCCL 上报 `maxRecvs=8`，且 NCCL 默认 `NCCL_NET_SHARED_COMMS=1`——同一
节点上多个本地 rank 的逻辑流**共享同一条 sendComm/recvComm 物理连接**（每对
netDev×remoteRank 只建一条）。于是 grouped irecv 的一组 elem 里可能混着"等
rank A 数据的 buffer"和"等 rank B 数据的 buffer"，而共享连接那头发送的消息也
来自不同 rank。若没有 tag，只能按位置顺序匹配，而两端各自的分组顺序相互独立，
必然错配——rank A 的数据可能落进等 rank B 的 buffer。有了 tag（=发送方 rank），
每条消息对号入座。

历史对照：**v3 API 的 isend/irecv 没有 tag 参数**——那时一条连接只承载一条流，
严格 FIFO 对齐即可，插件仅用 `seq != fifoHead` 校验顺序错配（见
`notes/ib_plugin_analysis_hpcx-v2.7.0.md` §5）。tag 是 v8 引入
grouped/multi-recv 与 shared comms 之后的配套机制。

## 16. 附 E：grouped irecv 全链路解析

### 16.1 形态

v8+ 的 irecv 一次 post **n 个**（buffer, size, tag, mhandle），n ≤ `maxRecvs`
（IB 插件上报 `NCCL_NET_IB_MAX_RECVS=8`），产出**一个** request（v3 一次只 post
一个 buffer）。这一组 buffer 在插件里对应 fifo 的一行
`fifo[slot][0..n-1]`——**"组"是调度、发射、完成的基本单位**。

### 16.2 动机

- **shared comms 的去复用**：一个 recvComm 同时承载多条流的接收，需一次挂出
  分属不同流的多个 buffer（配 tag 区分，见附 D）；
- **省 credit 开销**：一组 n 个接收只用**一条 RDMA_WRITE** 把 n 个 credit elem
  推给发送端，而非 n 条——credit 是每步流水线都要发的控制流量，聚合成 1/8 后
  显著降低 QP 压力与延迟；
- **匹配 NCCL 的 group 语义**：`ncclGroup` 中多个 send/recv 在 proxy 层天然成批
  提交，分组接收一次消化。

### 16.3 NCCL 侧：组的形成与下发

（行号取自 NCCL master `src/transport/net.cc` 的 `recvProxyProgress`）

1. **分组形成**（net.cc:1500-1510 附近）：proxy op 被 append 时，同批 subs 打上
   相同 `groupSize`；主循环按组步进
   `for (s = 0; s < args->nsubs; s += args->subs[s].groupSize)`。
2. **组装数组**（net.cc:1519-1583）：遍历组内 subs，只要该 sub 还有 step 未
   post（`posted < nsteps`）且未超流水线深度
   （`posted < done + maxDepth`，`maxDepth = min(NCCL_STEPS,
   NCCL_SHARED_STEPS/nsubs)`），就把当前 step 填入并行数组
   `ptrs/sizes/tags/mhandles/phandles[subCount]`；其中
   `tags[subCount] = resources->tpRemoteRank`（该 sub 所属流的对端 rank）。
3. **一次下发**（net.cc:1590）：`ncclNet->irecv(recvComm, subCount, ptrs, sizes,
   tags, mhandles, phandles, requestPtr)`；`recvComm` 取自组内第一个 sub 的连接
   ——shared comms 保证同组共享同一 recvComm。LL/LL128 且 `subCount==1` 时置
   `NCCL_NET_OPTIONAL_RECV_COMPLETION`，NCCL 可不等完成回执（net.cc:1585-1588）。

### 16.4 插件侧：ncclIbIrecv + ncclIbPostFifo

`ncclIbIrecv`（`ib_plugin.c:1889`）收下一个组后做三件事：

1. **建一个组级请求**：`req->type = RECV, req->nreqs = n`，per-dev `devBases`
   预填。整组共享这一个 request——NCCL test 一次，拿回全部 n 个 size。
2. **挂空 recv WR**（:1916-1925）：在 `nDataQps` 个 QP 上各挂一个
   `num_sge=0` 的 recv WR（轮转 `qpIndex`），每个 QP 给对应 dev 计一个 event。
   这些 WR 不接收数据（数据由发送端 RDMA 直接写入用户 buffer），只承接每个
   数据 QP 收尾那条 `RDMA_WRITE_WITH_IMM` 产生的 CQE。
3. **发整组 credit**（`ncclIbPostFifo`，:1809）：

```c
slot = comm->remFifo.fifoTail % MAX_REQUESTS;  // 组序号取模
req->recv.sizes = comm->sizesFifo[slot];        // sizes 回传指针绑定本组槽位
for (i=0; i<n; i++) req->recv.sizes[i] = 0;
for (i=0; i<n; i++)
  localElem[i] = { addr=data[i], size=sizes[i], tag=tags[i],
                   rkeys[j]=每个本地 dev 的 MR rkey, nreqs=n, idx=fifoTail+1 };
wr = RDMA_WRITE(对端 fifo + slot*8*sizeof(elem), 负载 = n 个 elem 一次写完);
```

- `idx = fifoTail+1`：全组 elem 同一序号，发送端据此判断"这组 credit 有效"
  （`slots[0].idx == fifoHead+1`）；
- 一条 RDMA_WRITE 推 n 个 elem——"省 credit"的落点；
- CTS QP 按 `base.devIndex` 轮转；默认 unsignaled，仅当
  `slot == ctsQp->devIndex` 时升级 signaled 并计入 event——周期性排水，防止
  SQ 被 unsignaled WR 塞满（:1852-1874 的注释）；
- `fifoTail++` 推进到下一组。

**`req->recv.sizes` 的绑定**是关键：接收端无法直接得知每条流实际收到多少字节
（发送端可能截断），发送端会把真实 sizes 数组 RDMA 到接收端 `sizesFifo[slot]`
（n>1 时），req 只是提前把指针指过去；n==1 时 size 走 imm_data，不用它。

### 16.5 发送侧配合（衔接附 C）

组 credit 到达后，发送端逐条 isend 按 tag 认领 elem；**全组 n 个 elem 都被认领
后**才调 MultiSend：每个请求一个数据 `RDMA_WRITE` 串成 WR 链，末尾追加
`RDMA_WRITE_WITH_IMM`（imm=0），其负载是各请求真实 size 数组，写入接收端
`sizesFifo[slot]`；整链在 nDataQps 个 QP 上各发一份（128B 对齐切条），每个 QP
产出一条发送端 signaled CQE 与一条接收端 imm CQE。接收端一个组的总账：
`nDataQps` 条 imm CQE + 可能的 1 条 credit signaled CQE，全部计入
`req->events[]`，归零即完成。

### 16.6 完整时序（n=3，单 QP 简化）

```
接收端                                     发送端
──────                                     ──────
NCCL: 3 个 sub 组一组
irecv(n=3, ptrs, sizes, tags):
  req{nreqs=3, recv.sizes=&sizesFifo[5]}
  post_recv(空WR) × nDataQps
  postFifo: RDMA_WRITE ── elem[0..2]{addr,size,tag,idx=6} ──► fifo[5]
                                       isend(msgB, tag=22): 命中 elem1 → 认领
                                       isend(msgA, tag=11): 命中 elem0 → 认领
                                       isend(msgC, tag=33): 命中 elem2 → 组齐!
                                       MultiSend: WR链[WrA, WrB, WrC,
                                         WrImm(负载sizes{...}→sizesFifo[5],
                                               imm=0, SIGNALED)]
                                         post 到每个数据 QP
数据 RDMA 直接落入 ptrs[0..2] ◄──────────
test: imm CQE × nDataQps (+credit WC)
      → events 归零 → done
      sizes[] = sizesFifo[5] = {实际3个size}
NCCL: 一个 request 完成 → 3 个 sub 的 received 推进
```

### 16.7 边界点

- **n=1 退化**：size 走 imm_data，不用 sizesFifo；末尾 WR 不追加（自身升级为
  WRITE_WITH_IMM）；
- **容量账**：64 个 slot = 最多 64 组在途；每组 ≤8 个接收；请求池 64（recv 每
  组 1 个、send 每元素 1 个）——NCCL 侧 `maxDepth` 保证不超发；
- **组间保序、组内乱序**：发送端严格按 `fifoHead` 顺序消费组，组内按 tag 乱序
  认领——这是 grouped 模型与 tag 匹配的分工；
- **一个 request 代表一整组**：NCCL 不能"部分完成"某个 sub——组内所有流的这步
  step 同生共死，这也是 NCCL 只把"同批 append、共享连接"的 subs 放一组的原因。

## 17. 附 F：容量辨析——"每组 8 个"与单机多 GPU 的关系

疑问：grouped irecv 每组最多 `NCCL_NET_IB_MAX_RECVS=8` 个 buffer，单机 16（或更多）
块 GPU 时是否不够用？答案是否定的——**8 是"单批"上限，不是"总量"上限**。

### 17.1 NCCL 侧自动拆批

`recvProxyProgress` 的分组逻辑（NCCL `src/transport/net.cc:1475-1500`）：

```c
// Initialize subs and group them by same recvComm.
for (int s = 0; s < args->nsubs; s++) {
  if (groupSize == maxRecvs) {
    groupSize = 0;              // 满 8 个就开新组
  } else if (s > 0) {
    // 把后面相同 recvComm 的 sub 交换过来聚拢
    ...
  }
  groupSize++;
  ...
}
```

- 组按 **recvComm 聚类**：不同连接的 sub 不会混进一组；
- 组大小硬上限 = 插件上报的 `maxRecvs`（IB=8）：16 条流共享一条连接时拆成
  8+8 两组、发两个 irecv。插件永远收不到 n>8 的调用（`ib_plugin.c:1895` 的
  报错只是防御）；主循环 `for (s = 0; s < args->nsubs; s += groupSize)` 逐组处理。

### 17.2 插件侧：组是循环复用的流水线槽位

一条连接的真实容量是 **64 组在途**（`MAX_REQUESTS=64` 个 fifo slot + 64 个请求
池槽位），每组最多 8 个 buffer——理论最多 512 个已公告接收、64 组流水。16 卡
共享一条连接只占 2 个 slot。组完成后 slot 与请求立即回收复用，NCCL 侧用
`maxDepth = min(NCCL_STEPS, NCCL_SHARED_STEPS/nsubs)` 控制超前深度。正如 TCP
窗口大小不限制文件总大小，8 只决定"每批多大"，不决定"能承载多少流"。

### 17.3 真实场景中一条连接的流数

- **集合通信（ring/tree）**：每个 rank 只有 1~3 个对端，每条连接一条流，组大小
  为 1，8 根本用不满；
- **p2p 批量（all-to-all、`ncclGroup` 多对端）**：multi-recv 的用武之地。shared
  comms 按 `(netDev, 对端 proxy rank, channelId)` 合并——落到某一条连接上的是
  "本节点若干 rank ↔ 同一对端"的流，受 `p2pnChannels` 限制，通常几条到十几条；
  超 8 条即按 §17.1 拆成多组。

### 17.4 "8" 不构成瓶颈的边界

- 每 vDev 最多融合 4 张物理卡（`NCCL_IB_MAX_DEVS_PER_NIC=4`），16 卡节点是 16
  个独立 vDev 各用各的连接，与 8 无关；
- 物理设备上限 `MAX_IB_DEVS=32`、vDev 上限 `MAX_IB_VDEVS=256`、每连接 QP 上限
  `NCCL_IB_MAX_QPS=128`，均与 8 无关。

8 只是 NCCL 与插件约定的甜点批大小：批越大 credit 聚合越省（一条 RDMA_WRITE
推 8 个），但"组齐才发"的耦合同步也越强。单机 16/32 卡只是让每条连接多拆几组，
功能与性能路径完全一致。

## 18. 附 G：NCCL proxy 的 args/subs 模型

理解 grouped irecv 的上游：插件看到的"组"来自 NCCL proxy 的 `ncclProxyArgs`。
一句话模型：**args 是提交给 proxy 线程的一个"工作包"，sub 是包里的一条流**。
一次 GPU kernel 启动对应的网络工作通常不止一条流，故一个 args 挂多个 sub。

（本节行号取自 NCCL master 的 `src/include/proxy.h`、`src/proxy.cc`。）

### 18.1 数据结构的分工（proxy.h:140-215）

- **args 持有"全组一致"的字段**：`protocol`、`algorithm`、`coll`、`dtype`、
  `redOp`、`sliceSteps/chunkSteps/chunkSize`（流水线切分参数）、`pattern`
  （Send/Recv/Coll）、`nChannels/nPeers`、`progress`（传输层 progress 函数）；
- **每个 sub 持有"一条流各自"的字段**：`connection`（属于哪条 proxy 连接）、
  `peer`、`channelId`、`sendbuff/recvbuff`、`nbytes`、`nsteps`，独立的进度计数器
  `posted/received/transmitted/flushed/done`、`requests[NCCL_STEPS]`（每步的
  net request），以及附 E 中的 `groupSize`。

即：args 决定"这批流按什么协议、什么步调走"，sub 记录"我这条流走到哪了"。

### 18.2 sub 如何挂进 args（proxy.cc:360-470）

上层提交的工作单元是 `ncclProxyOp`（一条连接上的一段工作）。`ProxyAppend`
决定合并还是新起 args：

```c
if (shared && args->opCount == op->opCount) {
  ncclProxyOpToArgs(op, args, args->nsubs);   // 同一批工作 → 追加为下一个 sub
}
```

- `opCount` 是 communicator 内单调递增的启动计数——**同一 opCount = 同一次
  kernel 启动/同一批 group 工作**，满足条件即挂入现有 args 的 `subs[nsubs++]`；
- `ncclProxyOpToArgs` 校验同组 ops 的 `sliceSteps/chunkSteps/protocol/dtype/
  redOp/coll` 必须一致，否则报 `"Proxy append mismatch"`——步调完全相同的流
  才允许同包；
- 上限 `NCCL_PROXY_MAX_SUBS = MAXCHANNELS`（通常 32）。

两条典型来源：

1. **集合通信**：一个 coll 操作按 channel 并行拆分，每个 channel 对其 peer 有
   一条收/发流 → 同一 opCount 的多个 op 合并，nsubs ≈ 该操作用的 channel 数
   （ring：每 channel 1 个 peer；tree：最多 3 个）；
2. **p2p group（`ncclGroup` 批量 send/recv）**：一批提交多个对端的收发 → 同一
   opCount，每个 sub 是一个 (peer, channel, 方向) 流；all-to-all 时一个 args
   可挂很多 sub。

### 18.3 设计意图

- **同生共死**：一个 kernel 的所有 channel/流必须协同推进（kernel 等待全部
  完成），包成一个 args 统一调度、统一回收；
- **批处理效率**：proxy 线程每轮 progress 处理一个 args 的所有 subs，一次唤醒
  推进多条流；
- **为 grouped net 操作供料**：正因一个 args 天然聚了一束流，
  `recvProxyProgress` 才能按 recvComm 聚类、拆出 ≤maxRecvs 的组，调一次
  grouped irecv（附 E 入口 `for (s = 0; s < args->nsubs; s += groupSize)`）。
  **sub 是 NCCL 的调度单位，"组"是插件的发射单位**，这层映射靠 subs 数组完成；
- **进度独立**：每个 sub 五套计数器各记各的，args 级循环"给所有 sub 各推进
  一步"，快流慢流互不阻塞。

### 18.4 直观示例

NCCL 自带的 `printProxyOp`（proxy.cc:266）即按此模型打印：

```
[0-3|12345|Coll 7R/2 5D/2 9G/3]   ← 一个 Coll args，3 个 sub：
                                    对端7 channel2 接收中、对端5 channel2 完成、对端9 channel3 等GPU
[0-4|12346|Recv 12I/0 12I/1]      ← p2p 接收 args：对端12 的 channel0/1 两条流
```

`peer状态/channel` 每段一个 sub。`NCCL_DEBUG=PROXY` 下可见真实 args 调度序列，
与插件侧 `NET/IB` 日志中的 grouped irecv 一一对应。

### 18.5 完整链条

用户 API（coll / p2p group）→ 每 channel/对端一个 proxy op → 同 opCount 合并
为一个 args（多 subs）→ proxy 按 recvComm 把 subs 聚成 ≤8 的组 → 插件
grouped irecv 一次 post 一组 → 一个 request 完成整组。

## 19. 附 H：为什么 8 GPU : 8 NIC 机器上 nreqs / nsubs 几乎恒为 1

现象：在 8 张 GPU 对应 8 张 RDMA 网卡（1:1 rail-optimized）的机器上，插件
isend/irecv 中的 `nreqs` 与 NCCL proxy args 的 `nsubs` 绝大部分时候都是 1。
这是符合预期的，两个观测互为因果。

### 19.1 为什么 nreqs 几乎总是 1

`nreqs > 1` 的前提是一次 grouped irecv 聚进 ≥2 个 sub，而聚组的前提是**多条流
共享同一条 recvComm**（`recvProxyProgress` 按 recvComm 聚类，见附 E/G）。共享
的发生条件（NCCL net.cc）：

```c
// shared comms 的键：netComms[netDev][tpRemoteRank].recvComm[channelId]
if (resources->maxRecvs > 1 && ncclParamNetSharedComms()) { ... 复用连接 ... }
```

即"同一 proxy 进程内，多条流的 (netDev, 对端 rank, channelId) 完全相同"。
8 GPU : 8 NIC 的 1:1 拓扑下每个 rank 独占一张网卡：

- 不同对端 → `tpRemoteRank` 不同 → 不同共享条目；
- 不同 channel → `channelId` 不同 → 不同条目；
- 不同本地 rank → `netDev` 不同（各用各的 NIC）→ 不同条目。

每条流都有独立连接，聚组时每组只有 1 个 sub，故 **nreqs 恒为 1**。共享真正
生效的场景：PXN（多 GPU 流量经同一 rank 的网卡/proxy 转发）、GPU 数多于网卡
数（如 8 GPU 对 2 NIC）、同一 (channel, peer) 建多条连接（connIndex>0）。

### 19.2 为什么 nsubs 几乎总是 1

`nsubs > 1` 需要 `ProxyAppend` 把多个 op 合并进一个 args，合并条件：

```c
if (shared && args->opCount == op->opCount) { 追加为 sub }
```

而连接的 `shared` 标志（net.cc 的 sendSetup/recvSetup）：

```c
shared = (graph || connIndex == 0) ? 0 : (NCCL_NET_SHARED_BUFFERS ?: 1);
```

- **集合通信（ring/tree）graph 非空 → shared 恒 0** → 每个 op 独占一个 args →
  **nsubs 恒 1**。以 all_reduce/all_gather 等集合通信为主（如 nccl-tests 的
  对应 perf 程序）时，这就是全部解释；
- 只有 **p2p 批量**（`ncclGroup` 内多个 send/recv，如 alltoall）且 connIndex>0
  的连接才 shared，同 opCount 的流才合并出多 sub。

### 19.3 什么时候会看到 >1

跑 **alltoall 类 p2p 负载**（nccl-tests 的 `alltoall_perf` 即 ncclGroup 批量
send/recv）并满足共享条件——如开启 PXN、网卡数少于 GPU 数——即可看到
nsubs>1 的 args 与插件侧 nreqs>1 的 grouped irecv。验证方法：

```sh
NCCL_DEBUG=INFO NCCL_DEBUG_SUBSYS=PROXY,NET ./build/alltoall_perf -b 1M -e 128M ...
# PROXY 日志 printProxyOp 一个方括号内出现多段 "peer状态/channel" 即多 sub
```

### 19.4 没有性能损失

nreqs=1 走的是附 C 的**最短路径**：size 随 imm_data 回传、不追加 sizesFifo
WR、一次 isend 独立成组即时发射。grouped 机制在 1:1 拓扑下只是闲置而非开销
——credit 聚合的收益本来就是为了补偿"多流挤一条连接"的场景；一卡一连接时
本来就不需要它。
