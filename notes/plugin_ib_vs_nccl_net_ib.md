# 插件 ib_plugin vs NCCL 原生 net_ib：全面对比

> 对比对象
> - **in-tree（原生）**：`nccl/src/transport/net_ib/p2p.cc`（+ `common.h` / `p2p.h`）—— NCCL 自带的 IB verbs 网络传输。
> - **plugin（插件）**：本仓库 `src/ib_plugin.c`（+ `include/p2p_plugin.h`）—— Mellanox 外置 RDMA/SHARP 网络插件。
>
> 数据通路层面的逐行对照见姊妹篇 `net_ib_vs_plugin_data_path.md`；本文在其基础上补全工作方式全景与内存用量核算。

## 一句话结论

插件是原生实现的**早期同源 fork**，核心协议完全一致；原生版已进化为"确定性选路 + 容错 + 因子化完成处理"的现代形态，插件停留在"轮询计数器 + 内联处理 + 无容错"的旧形态，但额外拥有 MR 缓存、vDev 融合、DMA-BUF/data-direct、SHARP 配套等 Mellanox 特性。

---

## 1. 共同的协议基线（工作方式相同之处）

两边都是经典 **CTS（Clear-To-Send）pull 模型**，数据面一字不差：

```
        接收方 Irecv                              发送方 Isend
 ┌───────────────────────────┐          ┌────────────────────────────┐
 │ (a) post 零字节 recv WR    │          │  查自己 FIFO 里有无 CTS     │
 │     (捕获 IMM 用)          │          │  idx == fifoHead+1 ?        │
 │ (b) RDMA-WRITE CTS 元素    │ ───────► │  未就绪 → *request=NULL     │
 │     到发送方 FIFO          │          │  就绪 → MultiSend:          │
 └───────────────────────────┘          │     RDMA-WRITE data         │
                                        │     + RDMA-WRITE-WITH-IMM ─ │ ──┐
                                        └────────────────────────────┘   │
 ┌───────────────────────────┐                                           │
 │ recv WR 被 IMM 消耗       │ ◄─────────────────────────────────────────┘
 │ → 数据到达完成事件         │
 └───────────────────────────┘
```

| 环节 | 共同做法 |
|---|---|
| Irecv | 先 post **零字节 recv WR**（只捕获 IMM），再把 `{addr, rkeys[4], tag, idx}` 打包成 credit 元素 **RDMA-WRITE 到发送方 FIFO**（即 CTS） |
| Isend | 查 FIFO 里 CTS 是否就绪（`slots[r].idx != idx` → 自旋等待）；未就绪返回 `*request=NULL` 由 proxy 重试 |
| 数据 | 就绪后 MultiSend：**RDMA-WRITE 数据 + 最后一条用 `RDMA_WRITE_WITH_IMM`**，IMM 消耗接收方 recv WR 产生完成事件 |
| Flush | 每设备一条 signaled `IBV_WR_RDMA_READ` 拉"写可见性"（GDR 场景） |
| Poll | **无自治 progress 线程**，全部由 NCCL proxy 线程调 `Test()` 时驱动（demand-driven） |
| 批处理 | 仅最后一条 WR 置 `IBV_SEND_SIGNALED`，其余 unsignaled；128B chunk 对齐（服务 LL/LL128 协议） |
| 常量 | `NCCL_NET_IB_MAX_RECVS=8`、`MAX_REQUESTS=8×8=64`、`events[4]` 按 dev 计数——两边一致 |

所有差异都在"如何选 QP、如何传 size、如何记账完成、如何处理多 QP / AR / 容错"。

---

## 2. 工作方式的核心差异

### 2.1 QP 选择与条带化 —— 最大架构分歧

- **插件**：两个独立**轮询计数器**。发数据用 `comm->base.qpIndex`（每 post 一次 `(qpIndex+1)%nqps`，`ib_plugin.c:1678/:1718`），CTS 另用一个 `devIndex` 轮询（`:1820-1821`）。分片靠 per-request 的 `req->send.offset` 累加推进（`:1688/:1712`）。`ncclIbRequest` **没有 id 字段**（`p2p_plugin.h:66`）。
- **原生**：`req->id = fifoHead`，收发两侧用同一个确定性公式 `(id*nQps+qpIndex)%nqps`（基于动态 `activeQps[]`，`common.h:393`）选出**完全相同的 QP 集合**，并有 `sentData[qpIndex]` 选择性重传记账。

**影响**：原生的条带对任意 request 都可预测且收发严格对称（容错重传和 AR 切换依赖此性质）；插件靠两个计数器在两端"恰好同步"，多 QP/AR 场景下稳健性弱。

### 2.2 size 回传与完成匹配

- **插件**：单一 index 方案。nreqs==1 时 immData=send size（`:1645`）；nreqs>1 时通过独立的 `remSizesFifo/sizesFifo`（`int[64][8]`）RDMA-WRITE 回传，immData=0（`:1647-1651`）。Test 里统一按 `wr_id & 0xff` 低 8 位定位 request（`:2042`），多 send 再解码 `(wr_id>>(j*8))&0xff`。
- **原生**：双匹配方案（`IB_RECEIVER_SIDE_MATCHING_SCHEME`，默认 BY_INDEX）。BY_ID 用 immData=request id 抗乱序到达；nreqs>1 的 size 写入专门的**完成记录结构** `cmplsRecords->sizes[]`。

### 2.3 Test/Poll 完成处理 —— 分歧最集中

- **插件**（`ib_plugin.c:1976-2081`）：全部**内联**在 Test 里，粗分 send/非-send，`req->events[i]--` 兜底减计数；任何错误 CQE 直接 `ncclRemoteError` **遇错即死**（`:2037`）。
- **原生**（`p2p.cc:817-876`）：因子化为 `RequestRetrieveFromCompletion` + `CompletionEventProcess`，显式 4 路 opcode 分派（RECV_RDMA_WITH_IMM / RDMA_READ / RDMA_WRITE / send）；入口有 `ResiliencyProgress` 容错钩子，支持**选择性重传**；循环抽干所有可得 CQE 才返回，且可跳过无事件的设备。

### 2.4 CTS 完成记账（隐蔽但关键）

- **插件**：`PostFifo` 的 signaled CTS 完成**计入该 recv request 的 events**（`:1879`）→ recv 请求必须等"CTS 完成 + 数据 IMM 完成"两路都到才算 done。
- **原生**：CTS 完成在 Test 中独立分支处理（`:787`），不混入数据完成计数。两边 `events[]` 语义并不相同。

### 2.5 接收 WR 管理

- **插件**：每次 Irecv 构造局部零字节 WR post（`num_sge=0`，`:1910-1923`），无复用。
- **原生**：共享 WR 结构 + `prepostReceiveWorkRequests` 优化（完成后自动补 post，`:779`），减少路径长度。

### 2.6 错误处理与健壮性

| 维度 | 插件 | 原生 |
|---|---|---|
| 异步事件线程 | 有（per-dev，`ncclIbDev` 内） | 有 |
| CQE 错误 | 即刻返回 fatal error | 容错恢复路径（resiliency），无容错才报错 |
| GetRequest 初始化 | **只清 `events[0..1]/devBases[0..1]`**——4 元数组只清一半（`:1463-1465`），已知健壮性瑕疵 | memset 全部 |

### 2.7 插件独有的能力（原生没有或位置不同）

- **vDev 多网卡融合**（v9+ `makeVDevice`）：物理 dev（≤32，持 verbs 资源）+ vDev（≤256，纯账本）；NCCL 拓扑层决定融合策略（默认 PATH_PORT，可 `NCCL_NET_FORCE_MERGE` 显式分组），连接内按成员 dev 条带化、聚合带宽上报拓扑。原生 NCCL 是这套 API 的消费端，自身不实现融合资源管理（详见 master 解析附 I）。
- **MR 缓存**：每物理 dev 一个**页粒度、引用计数、有序槽数组**缓存（`ncclIbRegMrDmaBufInternal2`，`:1482`），新注册区间若被已有槽完全覆盖则 `refs++` 零系统调用复用；flags 恒含 LOCAL_WRITE\|REMOTE_WRITE\|REMOTE_READ\|REMOTE_ATOMIC。原生把注册分离在 `reg.cc`，无 dedup 缓存。
- **DMA-BUF + mlx5 data-direct**：fd≠-1 时普通设备走 `ibv_reg_dmabuf_mr`，mlx5 data-direct 设备走 `mlx5dv_reg_dmabuf_mr`（`mlx5_access=1`）；relaxed ordering 支持（`IBV_ACCESS_RELAXED_ORDERING` + `ibv_reg_mr_iova2` 显式 iova）。
- **ECE**（Enhanced Connection Establishment）、GDR 模块探测（`/sys/kernel/mm/memory_peers/nv_mem{,_nc}/version`）、SHARP CollNet 配套（`sharp_plugin.c`，导出 `ncclCollNetPlugin_v9/v10/v11`，原生无）。
- **多版本 API**：同时服务 v6–v11 六套接口（适配包装链，见 master 解析 §1.2），原生只实现当前版本。

---

## 3. 内存使用对比

### 3.1 进程级（全局）

| 资源 | 插件 | 说明 |
|---|---|---|
| 物理 dev 表 | `ncclIbDevs[32]`：ibv_context、共享 PD（`pdRefs` 引用计数）、portAttr 缓存、MR 缓存、AR 标志、异步事件线程 | PD 跨连接共享省内存 |
| vDev 表 | `mergedDevs[256]`，仅 `vProps.devs[]` + speed + 拼接名，**零 verbs 资源** | 融合不复制资源 |
| MR cache | 每 dev 有序槽数组，32 起步倍增 | dedup 减少 HCA 侧 MTT/锁表占用（非主机内存） |

### 3.2 每连接（host 固定内存，全部 pinned 注册）

插件侧具体数字（`MAX_REQUESTS = NCCL_NET_MAX_REQUESTS(8) × NCCL_NET_IB_MAX_RECVS(8) = 64`）：

| 结构 | 尺寸 | 大小 |
|---|---|---|
| `reqs[64]`（send/recv comm 各一份） | ~200B/req | ~13KB×2 |
| `qps[128]`（`NCCL_IB_MAX_QPS` 上限，实际用 `max(双端ndevs)×NCCL_IB_QPS_PER_CONNECTION`，默认 1） | 24B/qp | ≤3KB |
| 发送方 `fifo[64][8]` credit 槽（每 elem 64B：addr+size+rkeys[4]+nreqs+tag+idx+padding16） | 512×64B | **32KB** + fifoMr |
| 接收方 `remFifo.elems[64][8]` 同构镜像 | 512×64B | **32KB** + fifoMr |
| `sizesFifo` 双向各一份（`int[64][8]`） | 2KB×2 | 4KB + 各自 MR |
| `fifoReqs[64][8]` 指针表（send 侧，未注册内存） | 4KB | — |
| CQ | `2×MAX_REQUESTS×QpsPerConn` = **128 entries**/conn（每成员 dev 一个） | — |
| QP caps | max_send_wr=`2×MAX_REQUESTS`=128，max_recv_wr=`MAX_REQUESTS`=64（每 QP，`ib_plugin.c:779-780`） | HCA 侧资源 |
| `mhandle` | `mrs[4]`——同一 buffer 在每个成员 dev 的 PD 上各注册一次 | 多轨线性放大 |

原生侧基础结构同构（ctsFifo / 请求池 / QP 池量级相同），差异是：

- **多出**：`cmplsRecords` 完成记录、per-request `sentData[nqps]` 记账、prepost recv WR 池、profiling 字段——固定开销略大；
- **少做**：无 MR dedup 缓存（每次注册直通 `reg.cc`）。

### 3.3 内存使用风格总结

- **插件**：静态预分配池（64 requests、128 QPs、64×8 FIFO 一次性建好），pinned FIFO 约 70KB/连接/方向；好处是数据面零分配、`wr_id` 直接索引 O(1) 定位请求；MR 缓存显著减少重复注册与 HCA 内存。
- **原生**：同为静态池模型，但为容错/乱序/profiling 追加少量 per-request 元数据；MR 管理外置、无 dedup。

---

## 4. 差异总表

| # | 维度 | in-tree `transport/net_ib` | plugin `ib_plugin.c` | 影响 |
|---|---|---|---|---|
| 1 | QP/AR 选路与分片 | id 驱动确定性、收发对称 `(id*nQps+qpIndex)%nqps` + `activeQps` + 重传记账 + OOO-RQ | 两个独立轮询计数器 `qpIndex/devIndex`，无对称保证/无重传/无 OOO | 多 QP/AR 时插件收发对齐保证弱 |
| 2 | size 回传与匹配 | 双方案（BY_ID/BY_INDEX）+ 完成记录结构（抗乱序） | 单一 index 方案 + sizes FIFO | 插件不支持乱序到达匹配 |
| 3 | Poll 完成处理 | 因子化、opcode 分派、容错可恢复、可重 arm | 内联、按 wr_id 低 8 位定位、遇错即 ncclRemoteError | 插件错误即死，无法恢复 |
| 4 | CTS signaled 记账 | 不计入数据 request events（独立分支） | 计入 recv request events | 插件 recv done 需等两路完成 |
| 5 | GetRequest 初始化 | 全清 events/devBases | 只清 `[0..1]`（4 元数组清一半） | 插件健壮性瑕疵 |
| 6 | recv WR 投递 | 共享 WR + prepost 重 arm 优化 | 局部零字节 WR，无 prepost | 插件无预投递优化 |
| 7 | MR 管理 | 独立 `reg.cc`，无缓存 | 内联页粒度引用计数 cache | 插件注册更省、dedup |
| 8 | 多网卡融合 | makeVDevice API 的定义者/消费端，策略在 topo.cc | 融合执行与资源管理在插件（物理 dev + vDev 两级模型） | 关注点分离 |
| 9 | Mellanox 特性 | 无 | MR 缓存、DMA-BUF/data-direct、relaxed ordering、ECE、SHARP 配套 | 插件生态增强 |
| 10 | API 版本 | 仅当前版本 | 同时导出 v6–v11 六套表 + 适配包装链 | 兼容多代 NCCL |

---

## 5. 结论与工程影响

- **in-tree 版**面向现代多路径 / AR / 容错 / 大规模训练设计：重传、动态 `activeQps`、ID 匹配抗乱序、完成记录、prepost 重 arm、resiliency 钩子齐备。
- **plugin 版**更老更直白，适合"单路径或简单 AR、不要容错"的场景；因靠轮询计数器选 QP，多 QP/AR 切换时收发对齐保证弱于 in-tree；但凭借 MR dedup、vDev 融合、DMA-BUF 与 SHARP，在生产 Mellanox/HPC-X 环境中功能更强。
- **若要在插件里做 AR 或容错增强**，`MultiSend` / `PostFifo` 的 QP 选择（改为 id 驱动的确定性映射）与 `Test` 的完成记账（opcode 分派 + 完成记录）是必须先对齐到 in-tree 模型的两块。

---

## 附：源码定位索引

### in-tree（`nccl/src/transport/net_ib/`）
- 请求生命周期：`GetRequest p2p.cc:21`、`FreeRequest :38`、`AddEvent :43`
- 发送：`MultiSend :83`；`Isend :262`（CTS 检查 :282、自旋 :289、fence :291、`sendReqsCnt` :348）
- CTS：`PostFifo :364`（signaled :409，不计入 events）
- 接收：`Irecv :430`（共享 WR :476、prepost 重 arm :779）
- Flush：`Iflush :525`（RDMA_READ :550，flush wr_id offset `0x1000` `p2p.h:13`）
- Poll：`RetrieveFromCompletion :582`、`CompletionEventProcess :695`、`Test :817`（do/while :828、resiliency :821）
- 结构：`struct ncclIbRequest common.h:192`（id、sentData、cmplsRecords）；`GetQpForRequest common.h:393`

### plugin（`src/ib_plugin.c` + `include/p2p_plugin.h`）
- 请求生命周期：`GetRequest :1457`（仅清 [0]/[1]）、`AddEvent :730`（3 参）
- 发送：`MultiSend :1620`（qpIndex RR :1678/:1718、AR :1654、signaled last :1671）；`Isend :1724`（CTS 检查 :1740）
- CTS：`PostFifo :1809`（devIndex RR :1820、signaled :1875、AddEvent 计入 :1879）
- 接收：`Irecv :1889`（零字节 WR :1910-1923）；Flush：`Iflush :1937`（RDMA_READ :1960）
- Poll：`Test :1976`（while(1) :1980、`poll_cq` :2008、`reqs[wr_id&0xff]` :2042、遇错即死 :2037）
- 结构：`struct ncclIbRequest p2p_plugin.h:66`（无 id）；`fifo[MAX_REQUESTS][8] ib_plugin.c:643`；MR cache `:1482`；QP caps `:779-780`；CQ 深度 `:752`
