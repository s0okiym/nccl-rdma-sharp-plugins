# IB Plugin 深度分析（hpcx-v2.7.0）

本文基于 `hpcx-v2.7.0` tag 对应的代码，分析 `src/ib_plugin.c`（InfiniBand verbs 传输插件）
及其支撑代码（`src/p2p_plugin.c`、`src/ibvwrap.c`、`include/*.h`）的实现、关键机制、
流程与数据结构。文中引用格式为 `文件:行号`。

## 1. 定位与总体架构

本仓库是 NCCL 的外部网络插件（`libnccl-net.so`）。NCCL 运行时通过 `dlopen` 加载它，
查找导出符号 `ncclNetPlugin_v3`（`include/nccl_net.h:76`，`NCCL_PLUGIN_SYMBOL` 宏），
得到一个 `ncclNet_v3_t` 函数表，之后所有网络操作都经由这张表回调到插件。

整体分三层：

```
NCCL 核心
   │  dlopen + ncclNetPlugin_v3 (ncclNet_v3_t 函数表, net API v3)
   ▼
src/p2p_plugin.c   分发层：按 NCCL_PLUGIN_P2P 环境变量选择后端，
   │               并封装 NCCL_PLUGIN_SYMBOL = 选中的插件函数表
   ├──► src/ib_plugin.c     原生 IB verbs 后端（ibPlugin，本文主题）
   ├──► src/ucx_plugin.c    UCX tag 后端      （可选，HAVE_UCX_PLUGIN）
   └──► src/ucx_rma_plugin.c UCX RMA 后端     （可选）
支撑层：src/ibvwrap.c（libibverbs 封装）、include/socket.h（TCP bootstrap）、
        src/utils.c、include/param.h（环境变量）、include/core.h（错误宏、常量）
```

`p2p_plugin.c:40` 先定义一个全 NULL 的 `NCCL_PLUGIN_SYMBOL`，`pluginInit()`
（`p2p_plugin.c:61`）在 NCCL 首次调用 `init` 时读取 `NCCL_PLUGIN_P2P`
（`ib`/`ucx`/`ucx_rma`，默认 `ib`），然后用**结构体整体赋值**把选中的后端拷贝进
`NCCL_PLUGIN_SYMBOL`（`p2p_plugin.c:71-95`），再转发调用后端的 `init()`。

IB 后端自己的函数表在 `ib_plugin.c:703-720`：

```c
ncclNet_t ibPlugin = {
  "IBext",
  ncclIbInit,      ncclIbDevices,  ncclIbGetProperties,
  ncclIbListen,    ncclIbConnect,  ncclIbAccept,
  ncclIbRegMr,     ncclIbDeregMr,
  ncclIbIsend,     ncclIbIrecv,    ncclIbFlush,  ncclIbTest,
  ncclIbCloseSend, ncclIbCloseRecv, ncclIbCloseListen
};
```

net v3 API 的特点（`include/nccl_net.h`）：
- 发送/接收是**异步非阻塞**语义：`isend/irecv` 可以返回 `*request = NULL` 表示
  "现在做不了，稍后再调"，NCCL 会重试；真正的完成靠 `test()` 轮询。
- `flush()` 用于 GDR 场景保证数据对 GPU 可见。
- `listen/connect/accept` 三件套建立连接，`handle`（≤ `NCCL_NET_HANDLE_MAXSIZE`=64 字节）
  由 NCCL 负责在对端之间传递。

## 2. 关键数据结构

### 2.1 设备描述 `nccl_ib_dev_t`（`include/p2p_plugin.h:28`）

```c
typedef struct ncclIbDev {
  int      device;      // ibv_get_device_list 返回数组中的下标
  uint64_t guid;        // sys_image_guid，NCCL 用它区分多 PCI function 的网卡
  uint8_t  port;        // ibv 端口号（新卡上恒为 1）
  uint8_t  link;        // IBV_LINK_LAYER_INFINIBAND / ETHERNET(RoCE)
  uint8_t  isSharpDev;  // 是否 SHARP 交换机卸载 capable（ConnectX-6 IB）
  int      speed;       // Mbps = active_speed × active_width 换算
  struct ibv_context* context;  // verbs 上下文，所有 QP/PD/CQ 都挂在它下面
  char     devName[MAXNAMESIZE];
  char     *pciPath;    // /sys 下 PCI 路径；双口卡合并到同一 PCI 设备
  int      realPort;    // 同一 PCI 设备上的逻辑端口序号（见 3.4）
  int      maxQp;       // 上报给 NCCL 的 maxComms（SHARP 设备被限制为 NCCL_SHARP_MAX_COMMS）
} nccl_ib_dev_t;
```

全局数组 `ncclIbDevs[MAX_IB_DEVS]`（`ib_plugin.c:34`，上限 16，`core.h:18`）与计数
`ncclNIbDevs`（初值 -1 表示"未初始化"）由 `nccl_p2p_ib_init()` 惰性填充。

### 2.2 连接句柄与 QP 交换信息

```c
struct ncclIbHandle {                     // ib_plugin.c:104
  union socketAddress connectAddr;        // 仅仅是一个 TCP 地址(IP:port)
};
```

listen 句柄只是 bootstrap TCP 监听地址，远小于 64 字节上限
（`NCCL_STATIC_ASSERT`，`ib_plugin.c:254`）。真正的 IB 连接信息在 TCP 上另行交换：

```c
struct ncclIbQpInfo {                     // ib_plugin.c:89
  uint32_t lid;                           // IB 子网 LID；RoCE 时为 0
  uint8_t  ib_port;
  uint32_t qpn;                           // 对端的 QP number
  uint64_t spn, iid;                      // RoCE: GID 的 subnet_prefix/interface_id
  enum ibv_mtu mtu;
  uint32_t fifoRkey;                      // 对端 send-fifo 显存的 rkey
  uint64_t fifoAddr;                      // 对端 send-fifo 的地址（RDMA 写 credit 用）
};
```

### 2.3 通信对象（comm）

```c
struct ncclIbVerbs { struct ibv_pd* pd; struct ibv_cq* cq; };   // ib_plugin.c:108

struct ncclIbSendComm {                   // ib_plugin.c:135
  struct ncclIbVerbs verbs;               // 自己的 PD + CQ
  struct ncclIbSendFifo fifo[MAX_REQUESTS]; // 本地"信用"槽位（对端 RDMA 写入）
  struct ncclIbRequest reqs[MAX_REQUESTS];  // 请求池
  uint32_t fifoHead;                      // 下一个待消费槽位（全局序号，单调递增）
  int fd;                                 // bootstrap TCP socket
  int ready;                              // 连接是否已完成（惰性建立，见 4.3）
  struct ibv_qp* qp;
  struct ibv_mr* fifoMr;                  // fifo 的显存注册（对端可 RDMA 写）
};

struct ncclIbGpuFlush {                   // ib_plugin.c:146
  int enabled, hostMem;
  struct ibv_mr* hostMr;
  struct ibv_sge sge;                     // 1 字节 host 缓冲（RDMA_READ 的目的地）
  struct ibv_qp* qp;                      // 自环 QP（连自己），专做 flush
};

struct ncclIbRemFifo {                    // ib_plugin.c:154（接收端持有）
  struct ncclIbSendFifo elems[MAX_REQUESTS]; // 本地 staging：准备写给发送端的 credit
  uint64_t addr; uint32_t rkey;           // 发送端 fifo 的远端地址/rkey
  uint32_t tail;                          // 已发布的 credit 全局序号
  uint32_t flags;                         // 预留给 IBV_SEND_INLINE（当前禁用）
  struct ibv_mr* mr; struct ibv_sge sge;
};

struct ncclIbRecvComm {                   // ib_plugin.c:164
  struct ncclIbVerbs verbs;
  struct ncclIbRemFifo remFifo;
  struct ncclIbRequest reqs[MAX_REQUESTS];
  int fd; int ready;
  struct ibv_qp* qp;
  struct ncclIbGpuFlush gpuFlush;
};
```

注意 send/recv comm 都以 `struct ncclIbVerbs verbs` 开头，所以 `regMr/deregMr`
可以把 `void* comm` 直接当 `ncclIbVerbs*` 用（`ib_plugin.c:441`）——两种 comm 共用
同一个 PD，同一段 MR 对收发都有效。

### 2.4 请求与 credit 槽位

```c
struct ncclIbRequest {                    // ib_plugin.c:113
  int used;                               // 槽位是否占用
  int type;                               // 保留（本版本未细分）
  struct ncclIbVerbs* verbs;              // 完成事件要到哪个 CQ 上 poll
  int done;                               // 完成标志（由 test() 置位）
  int size;                               // 完成时记录实际字节数
  int free;                               // 内部请求：完成后立即自动回收
};

struct ncclIbSendFifo {                   // ib_plugin.c:127 —— 一条"接收信用"
  uint64_t addr;                          // 接收端数据缓冲地址
  int      size;                          // 接收端能接收的字节数
  uint32_t seq;                           // 全局序号 = 接收端 remFifo.tail（防错位）
  uint32_t rkey;                          // 接收端缓冲 rkey
  uint32_t ready;                         // 1 = 槽位有效
};
```

请求池是固定 128（`MAX_REQUESTS`，`core.h:16`）的静态数组，`ncclIbGetRequest()`
（`ib_plugin.c:388`）线性扫描找空闲槽，找不到就 `WARN` 并报错。`wr_id` 直接强转为
`ncclIbRequest*`（`ib_plugin.c:485`），完成时从 WC 还原指针——零分配的 fast path。

## 3. 初始化与设备枚举

### 3.1 入口

`ncclIbInit()`（`ib_plugin.c:64`）：
- 若 `NCCL_IBEXT_DISABLE=1` 直接返回错误，让 NCCL 跳过本插件；
- 检查 `NCCL_IB_PCI_RELAXED_ORDERING` 与本机 verbs 是否支持 `IBV_ACCESS_RELAXED_ORDERING`
  （`ibvwrap.h:18-20` 的编译期探测，不支持时该宏定义为 0）；
- 转调 `nccl_p2p_ib_init()`（`p2p_plugin.c:159`）做真正的枚举。

### 3.2 设备枚举流程（`nccl_p2p_ib_init`）

惰性初始化：`ncclNIbDevs == -1` 才执行，全程持 `nccl_p2p_lock` 互斥锁。

1. `wrap_ibv_fork_init()` —— `ibv_fork_init()`，避免 fork 出的子进程踩到 verbs 注册的
   页（配合 `ncclIbMalloc` 的页对齐分配，见 6.1 注释）。
2. `findInterfaces()`（`socket.h:202`）确定 bootstrap TCP 用的 IP 网口：优先
   `NCCL_SOCKET_IFNAME`；否则按 `ib` → 排除 docker/lo → docker → lo 的顺序自动挑。
3. 解析 `NCCL_IB_HCA`：`^` 前缀=黑名单，`=` 前缀=精确匹配，条目形如 `mlx5_0:1`
   （`parseStringList`/`matchIfList`，`utils.c:32-89`）。
4. 遍历 `ibv_get_device_list` 的每个设备每个端口：
   - 跳过打不开的设备、查询失败的、端口非 `IBV_PORT_ACTIVE` 的；
   - 只收 IB 与 Ethernet（RoCE）两种 link layer；
   - 按 `NCCL_IB_HCA` 白/黑名单过滤；
   - 填充 `ncclIbDevs[]`：guid、端口、link、`speed = nccl_p2p_ib_speed(active_speed)
     × nccl_p2p_ib_width(active_width)`（`p2p_plugin.c:301-318` 的两张查表，
     例如 50000×4 = 200G）、context、devName、maxQp；
   - 通过 sysfs（`/sys/class/infiniband/<dev>/device/{vendor,device}`，
     `readFileNumber`，`utils.c:118`）读 vendor/device id：vendor `0x15b3`
     （Mellanox）且 device `4123/4124`（ConnectX-6）且链路为 IB 时标记
     `isSharpDev=1`，并把 `maxQp` 压到 `NCCL_SHARP_MAX_COMMS`（默认 1）——
     限制 SHARP 网卡上 CollNet 并发通信数；
   - 每个被选中的端口为设备 context 起一个**异步事件线程** `ncclIbAsyncThreadMain`
     （`p2p_plugin.c:135`）：阻塞在 `ibv_get_async_event` 上，把除 `COMM_EST` 外的事件
     打进 WARN 日志（如 QP 进入 error 状态、LID 变更）。
5. 若同时存在 SHARP 与非 SHARP 设备，`qsort` 把 SHARP 设备排到数组前面
   （`devSharpCompare`，`p2p_plugin.c:149`），让 NCCL 优先选它们做 CollNet。
6. 打一行汇总日志：`NET/IB : Using [0]mlx5_0:1/RoCE [1]mlx5_1:1/IB/SHARP ; OOB ib0<ip>`。

### 3.3 PCI 路径合并（`nccl_p2p_ib_pci_path`，`p2p_plugin.c:281`）

对 `/sys/class/infiniband/<dev>/device` 取 `realpath`，然后把路径**最后一个字符改成
`'0'`**——双口 HCA 的两个端口在 sysfs 里表现为 `...:00.0` 和 `...:00.1`（不同 PCI
function），抹掉个位后两个端口归并到同一 `pciPath`。NCCL 的拓扑代码靠 `pciPath`
判断"这两条路径其实是同一张卡"；`realPort` 则按相同 pciPath 已出现的次数递增，
恢复出逻辑端口号。`getProperties` 上报的是 `port + realPort`
（`p2p_plugin.c:129`）。

### 3.4 设备属性上报（`nccl_p2p_ib_get_properties`，`p2p_plugin.c:117`）

- `ptrSupport`：默认 `NCCL_PTR_HOST`；若 `/sys/kernel/mm/memory_peers/nv_mem/version`
  存在（`nccl_p2p_gdr_support`，`p2p_plugin.c:102`，即 nv_peer_mem / GDR 内核模块
  已加载）则追加 `NCCL_PTR_CUDA`，NCCL 才会把 GPU 显存交给插件直接注册。
- `maxComms = maxQp`、`speed`、`port`、`guid`、`name`、`pciPath`。

## 4. 连接建立

### 4.1 三段式 + 惰性就绪

NCCL 的调用序列是 `listen → (handle 经 NCCL bootstrap 传给对端) → connect/accept`。
本插件在 verbs 资源之上叠加了一条 **TCP bootstrap 通道**做带外交换：

```
发送方（connect 侧）                     接收方（listen/accept 侧）
─────────────────────                     ─────────────────────────
listen: 建 TCP 监听，handle={IP:port}  ◄── (handle 由 NCCL 层搬运)
connect(dev, handle):
  TCP connect ────────────────►  accept(): TCP accept
  建 PD/CQ/QP(INIT)
  注册本地 fifo MR
  socketSend(qpInfo{qpn, lid/gid,
       mtu, fifoRkey, fifoAddr}) ──► socketReceive(remQpInfo)
                                    建 PD/CQ/QP(INIT)
                                    mtu = min(双端 mtu)
                                    QP → RTR(remQpInfo) → RTS   ← 立即就绪
                                    注册 remFifo.elems MR
                                    若 GDR：建 gpuFlush 自环 QP → RTR/RTS
                                    socketSend(自己的 qpInfo) ─┐
首个 isend:                                               ◄────┘
  ncclSendCheck: 非阻塞收 qpInfo
    收到后 QP → RTR → RTS，ready=1
    socketSend(ready) ─────────►  首个 irecv:
                                    ncclRecvCheck: 非阻塞收 ready
                                    ready=1
```

要点：

- **不对称的就绪时序**：接收端在 `accept()` 里就把 QP 推到 RTS（`ib_plugin.c:335-336`），
  而发送端把 RTR/RTS 推迟到第一次 `isend` 的 `ncclSendCheck()`（`ib_plugin.c:407`）。
  这是为了契合 NCCL 的非阻塞模型——`connect()` 返回时对端可能还没 `accept()`，
  所以最后一段握手（接收端 qpInfo 的回传）用 `socketProgress`(MSG_DONTWAIT)
  惰性推进，没好就返回 `*request=NULL` 让 NCCL 下次再调。`ready` 标志 + 最后一次
  `socketSend(ready)` 构成双向确认。
- **QP 状态机**：`ncclIbCreateQp`（`ib_plugin.c:186`）建 RC QP 并置 INIT
  （pkey_index=0，port，access flags）；`ncclIbRtrQp`（`ib_plugin.c:208`）置 RTR：
  IB 走 `dlid`，RoCE（lid==0）走 GRH（`dgid = spn+iid`，`sgid_index =
  NCCL_IB_GID_INDEX`，hop_limit=255，`traffic_class = NCCL_IB_TC`），
  `sl = NCCL_IB_SL`，`min_rnr_timer=12`；`ncclIbRtsQp`（`ib_plugin.c:236`）置 RTS：
  `timeout = NCCL_IB_TIMEOUT(14)`、`retry_cnt = NCCL_IB_RETRY_CNT(7)`、
  `rnr_retry = 7`。RQ/SQ PSN 都从 0 开始。
- **MTU 协商**：接收端取 `MIN(对端 mtu, 本端口 active_mtu)`（`ib_plugin.c:331`），
  并以协商值回传给发送端。
- QP 能力：`max_send_wr = max_recv_wr = MAX_REQUESTS(128)`，SGE 上限 1，
  `max_inline_data = 0`（`ib_plugin.c:192-196`）。
- `USE_RDMA_SEND_INLINE` 编译为 0（`ib_plugin.c:29`），credit 不走 inline；
  `remFifo.flags` 因此恒为 0。

### 4.2 bootstrap socket 层（`include/socket.h`）

- 全部 `static` 内联在头文件里；`createListenSocket` 支持 IPv4/IPv6、端口被环境
  强制时开 `SO_REUSEADDR/SO_REUSEPORT`；`connectAddress` 对 `ECONNREFUSED` 做
  1ms×20000 次重试（约 20s），容忍对端 listen 尚未就绪的竞态。
- `socketProgress` 用 `MSG_DONTWAIT` 非阻塞收发并按 offset 累计；`socketWait` 循环到
  收满；`socketSend/socketReceive` 是其阻塞封装。连接建立期的"非阻塞探测 + 补齐"
  模式（`ncclSendCheck/ncclRecvCheck` 中的 progress+wait 组合，`ib_plugin.c:413-415`）
  就建立在这套原语上。

## 5. 数据收发：credit-based RDMA Write 模型

这是整个插件最核心的设计。**数据面完全不使用双边 SEND/RECV**（`USE_RDMA_WRITE=1`，
`ib_plugin.c:28`），而是"接收端下发信用、发送端单边 RDMA 写"：

### 5.1 irecv（`ncclIbIrecv`，`ib_plugin.c:557`）

1. 惰性完成连接握手（`ncclRecvCheck`）。
2. 从请求池取 `req`，`post_recv` 一个 recv WR（sg 指向用户缓冲）。这个 recv WR
   **不用于收数据**，只用于承接 `RDMA_WRITE_WITH_IMM` 在完成队列上产生的
   `IBV_WC_RECV_RDMA_WITH_IMM` 事件（带 immediate 的 RDMA 写必须消耗一个接收 WQE
   才能生成本端 CQE）。
3. 调 `ncclIbPostFifo()`（`ib_plugin.c:527`）向发送端**发布 credit**：
   - 在本地 `remFifo.elems[tail % 128]` 填 `{addr=用户缓冲, rkey, size, seq=tail,
     ready=1}`；
   - 用 `IBV_WR_RDMA_WRITE` 把这个 24 字节的 elem 写到发送端
     `fifo[tail % 128]`（远端地址 `remFifo.addr + (tail%128)*sizeof(elem)`，
     rkey 来自连接建立时交换的 `fifoRkey`）；
   - `tail++`。该 WR 占用一个内部 request（`free=1`），完成即自动回收。

### 5.2 isend（`ncclIbIsend`，`ib_plugin.c:466`）

1. 惰性完成连接握手（`ncclSendCheck`）。
2. **本地轮询 credit 槽**：`slot = fifo[fifoHead % 128]`，若 `slot->ready == 0`
   说明接收端还没 post 对应接收，返回 `*request = NULL`（NCCL 稍后重试）。
   —— 天然实现了与 NCCL 匹配的流控：未经接收端许可绝不在线上发数据。
3. 校验（`ib_plugin.c:503`）：`size > slot->size || slot->size <= 0 || addr==0 ||
   rkey==0 || seq != fifoHead` 任一成立即报 collective mismatch（收发两端
   集合通信次数/大小不一致、或 credit 错位），返回错误。这是调试集合通信
   错配的关键检查。`__sync_synchronize()` 保证 ready 标志与后续字段读取的
   内存序。
4. 构造 WR：`IBV_WR_RDMA_WRITE_WITH_IMM`，`remote_addr = slot->addr`，
   `rkey = slot->rkey`，`imm_data = size`（字节数随 immediate 带给接收端），
   `IBV_SEND_SIGNALED`。零字节消息走 `num_sge=0` 的空写。
5. 清槽（`ready=0`，其余字段清零便于调试），`fifoHead++`，`post_send`。

数据路径因此只有**一次单边 RDMA 写**：发送端网卡直接从本地（可能为 GPU 显存的）
缓冲 DMA，经网络写入接收端缓冲，接收端网卡生成带 imm 的 CQE。无任何 CPU 拷贝、
无逐字节握手。

### 5.3 test（`ncclIbTest`，`ib_plugin.c:623`）

- 先看 `req->done`，已置位则直接返回完成并释放槽位（`used=0`）。
- 否则 `ibv_poll_cq` 一次最多批 4 个 WC（`ib_plugin.c:636-637`），对每个 WC：
  - `status != SUCCESS` → `WARN`（含 vendor_err）并返回 `ncclSystemError`；
  - `wr_id` 还原出 request：`IBV_WC_RECV` 时 `size = wc->byte_len`；
    `IBV_WC_RECV_RDMA_WITH_IMM` 时 `size = wc->imm_data`；置 `done=1`；
  - `free==1` 的内部请求（fifo post）立即回收。
- 循环直到没有新 WC 或本 request 完成。注意 **poll 会顺带推进同一 CQ 上其他人的
  request**——CQ 是 per-comm 共享的，test 任意一个请求都会消费 CQ 上所有已完成 WC。

### 5.4 序号与回绕

`fifoHead`/`remFifo.tail`/`seq` 都是单调递增的 32 位序号，槽位下标取
`% MAX_REQUESTS`。`seq == fifoHead` 校验保证 credit 与发送的一一对应；
128 的槽位深度意味着发送端最多领先接收端 128 个未消费 credit，超出时接收端
的 fifo 写会覆盖未消费槽——但 NCCL 上层保证 in-flight 请求数远小于此
（API 注释 `maxComms`、请求池本身也只有 128），所以安全。

## 6. GPUDirect RDMA 与 flush

### 6.1 GDR 支持判定

`nccl_p2p_gdr_support()`（`p2p_plugin.c:102`）只看
`/sys/kernel/mm/memory_peers/nv_mem/version` 是否存在（结果缓存于 static 变量）。
存在则设备属性带 `NCCL_PTR_CUDA`，NCCL 会把 GPU 显存 buffer 交给 `regMr`。

### 6.2 为什么需要 flush

RDMA 写完成（CQE 生成）只说明数据到达了接收端**网卡/主机内存子系统**，对 GPU
显存的写可能还滞留在 PCIe 路径上，GPU kernel 立即可读不到。因此 NCCL 在
接收端收到 CUDA buffer 后会调 `flush()` 做栅栏。

### 6.3 flush 实现（`ncclIbFlush`，`ib_plugin.c:592`）

- `accept()` 时若 GDR 可用且 `NCCL_GDR_FLUSH_DISABLE=0`，额外建一个 **自环 QP**
  （localQpInfo 填自己的 lid/gid/qpn，`RTR` 指向自己，`ib_plugin.c:360-370`），
  并注册一个 4 字节 host buffer。
- flush 时在该自环 QP 上发一个 **1 字节 `IBV_WR_RDMA_READ`**：`remote_addr` 是 GPU
  buffer（用户传入的 mhandle rkey），读进 host buffer。RDMA READ 的响应包会迫使
  此前到达的 PCIe 写先对 GPU 可见（PCIe 序保证 read completion 不会越过先前的
  posted write），随后用 `ncclIbTest` **忙等**该请求完成（`ib_plugin.c:615-618`）。
- 用独立 QP 而不是数据 QP，避免 flush 的 RDMA READ 与数据 WR 在同一 QP 上排队
  相互阻塞；`size==0` 或未启用 GDR 时 flush 直接返回。

### 6.4 内存注册（`ncclIbRegMr`，`ib_plugin.c:440`）

- 将 `[addr, addr+size)` 向下/向上对齐到 4KB（`REG_ALIGN`）再 `ibv_reg_mr`——
  verbs 注册按页进行，且 `ncclIbMalloc`（`utils.c:21`）的注释说明：注册过的页
  会被标 `DONTFORK`，若与其他数据共享页会导致子进程崩溃，所以内部通信对象
  一律用页对齐的 `posix_memalign`。
- flags 恒含 `LOCAL_WRITE|REMOTE_WRITE|REMOTE_READ`（REMOTE_READ 是 flush 的
  RDMA READ 所需）；`NCCL_IB_PCI_RELAXED_ORDERING=1` 时追加
  `IBV_ACCESS_RELAXED_ORDERING`。
- `mhandle` 就是 `struct ibv_mr*` 本体。

## 7. 资源释放与错误处理

- `ncclIbCloseSend/Recv/Listen`（`ib_plugin.c:666-701`）：关 TCP fd → destroy QP
  （recv 侧还包括 gpuFlush QP/hostMr）→ dereg fifo/remFifo MR → destroy CQ →
  dealloc PD → free。注意 QP 必须先于 CQ/PD 销毁。
- 错误处理统一走 `core.h` 宏：`NCCLCHECK` 透传 `ncclResult_t` 并在 INFO 日志留下
  `文件:行 -> 错误码` 的"回溯链"，`SYSCHECK*` 处理系统调用（对
  EINTR/EAGAIN/EWOULDBLOCK 自动重试）。`ibvwrap.c` 用 `IBV_*_CHECK` 系列宏把
  每个 verbs 调用的失败转成 `WARN + ncclSystemError`；`post_send/post_recv/
  poll_cq` 走 `context->ops.*` 直接派发到驱动 fast path（`ibvwrap.c:138`、
  `ibvwrap.h:43`）。

## 8. 线程模型与并发

- **主线程（NCCL 调用线程）**：所有 API 回调。数据面无锁——每个 comm 的
  fifoHead/tail、request 池都只被单个 NCCL 线程触碰（NCCL 保证 per-comm 串行
  调用）；credit 槽用 `volatile` 读（`ib_plugin.c:474-475`）+ `__sync_synchronize`
  应对网卡 DMA 写入。
- **异步事件线程**：每设备一个，只打日志，不触碰数据面。
- **初始化锁** `nccl_p2p_lock`（`p2p_plugin.c:26`）保护设备枚举的一次性初始化；
  `ib_plugin.c:36` 的 `ncclIbLock` 是遗留变量，实际未使用。
- 环境变量参数（`param.h` 的 `NCCL_PARAM` 宏）首次读取时上自己的互斥锁并缓存，
  之后无锁。

## 9. 可调环境变量速查

| 变量 | 默认 | 作用 |
|---|---|---|
| `NCCL_PLUGIN_P2P` | `ib` | 选择 p2p 后端：ib / ucx / ucx_rma |
| `NCCL_IBEXT_DISABLE` | 0 | 置 1 禁用本 IB 插件 |
| `NCCL_IB_HCA` | （全收） | 网卡:端口过滤，如 `mlx5_0:1,mlx5_1:1`、`^mlx5_2`、`=mlx5_0:1` |
| `NCCL_SOCKET_IFNAME` / `NCCL_SOCKET_FAMILY` | 自动 | bootstrap TCP 的网口 / 地址族 |
| `NCCL_IB_GID_INDEX` | 0 | RoCE 的 GID 下标 |
| `NCCL_IB_TIMEOUT` | 14 | QP RTS 的 ACK 超时（4.096µs×2^t） |
| `NCCL_IB_RETRY_CNT` | 7 | QP 重传次数 |
| `NCCL_IB_SL` / `NCCL_IB_TC` | 0 | IB Service Level / RoCE Traffic Class |
| `NCCL_IB_PCI_RELAXED_ORDERING` | 0 | MR 注册带 relaxed ordering 标志 |
| `NCCL_GDR_FLUSH_DISABLE` | 0 | 置 1 关掉 GDR flush 自环 QP |
| `NCCL_SHARP_MAX_COMMS` | 1 | SHARP 设备上报的 maxComms |

## 10. 设计要点小结

1. **零拷贝单边语义**：接收端 post credit → 发送端 RDMA_WRITE_WITH_IMM，数据路径
   只有一次 RDMA 写；immediate 同时充当"到达通知 + 字节数"，省掉单独的控制消息。
2. **credit FIFO 即流控也是同步**：发送端只在本地内存上 `volatile` 轮询槽位，
   无系统调用、无网络往返；`seq` 校验把集合通信错配变成明确的错误而非数据损坏。
3. **静态资源池**：128 深度的 request 池、fifo 槽、QP 容量全部编译期固定，
   数据面零 malloc；request 与 WR 通过 `wr_id` 指针直译，O(1) 映射。
4. **惰性连接就绪**：connect/accept 只做能无阻塞完成的部分，最后一段握手摊到
   首次 isend/irecv 里非阻塞推进，契合 NCCL net API 的"返回 NULL 即重试"模型。
5. **独立自环 QP 做 GDR flush**：用 1 字节 RDMA READ 借 PCIe 序实现 GPU 可见性
   栅栏，与数据 QP 隔离。
6. **与 NCCL 拓扑的接口**：`pciPath`（双口归并）、`guid`、`speed`、
   `port+realPort`、`maxComms`（SHARP 限流）这些属性是 NCCL 建图/选路的依据，
   插件通过设备枚举阶段的 sysfs 读取与换算把它们填准。

## 附：一次完整 send/recv 的时序

```
接收端                                  发送端
──────                                  ──────
irecv(buf,size):
  post_recv WR (占位接 imm)
  RDMA_WRITE ──credit{addr,rkey,seq,ready=1}──► 本地 fifo[head%128]
                                        isend(data,size):
                                          轮询 slot->ready==1
                                          校验 seq/size
                                          RDMA_WRITE_WITH_IMM ──数据──► buf
                                          (imm_data = size)
                                        test(): poll_cq → SEND 完成 → done
test(): poll_cq → RECV_RDMA_WITH_IMM
      size=imm_data → done
flush(): (GDR) 自环 QP RDMA_READ 1B，忙等完成 → GPU 可见
```
