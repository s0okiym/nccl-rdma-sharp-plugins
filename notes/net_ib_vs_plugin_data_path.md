# NCCL net_ib (in-tree) vs nccl-rdma-sharp-plugins：send / recv / poll 数据通路对比

> 对比对象
> - **in-tree**：`nccl/src/transport/net_ib/p2p.cc`（+ `common.h` / `p2p.h`）—— NCCL 自带的 IB verbs 网络传输数据通路。
> - **plugin**：`nccl-rdma-sharp-plugins/src/ib_plugin.c`（+ `include/p2p_plugin.h`）—— Mellanox 外置 RDMA/SHARP 网络插件。
>
> 两边实现的插件接口完全对应：`ncclIbIsend` / `ncclIbIrecv` / `ncclIbIflush` / `ncclIbTest` / `ncclIbMultiSend` / `ncclIbPostFifo` / `ncclIbGetRequest`。

## 一句话结论

插件是 NCCL in-tree net_ib 的**早期 fork**，整体协议（CTS pull 模型、RDMA_WRITE_WITH_IMM、按 device 计 events）完全同源。差异不在“用了不同的 verbs”，而在于：

- in-tree 已被重构成 **确定性的 per-request QP 映射 + 双匹配方案 + 容错重传 + 因子化的完成处理**；
- 插件仍停留在 **轮询计数器 + 单一匹配方案 + 无容错 + 内联完成处理** 的旧形态。

---

## 1. 共同的协议基线（理解差异的前提）

两边的数据通路是 NCCL IB 经典的 **“Clear-To-Send” 拉模型**，步骤完全一致：

```
        接收方 Irecv                              发送方 Isend
 ┌───────────────────────────┐          ┌────────────────────────────┐
 │ (a) post 零字节 recv WR    │          │  查自己 FIFO 里有无 CTS     │
 │     (捕获 IMM 用)          │          │  idx == fifoHead+1 ?        │
 │ (b) RDMA-WRITE CTS 元素    │ ───────► │  (addr/rkeys/tag/idx) ─────►│
 │     到发送方 FIFO          │          │  未就绪 → *request=NULL     │
 └───────────────────────────┘          │  就绪 → MultiSend:          │
                                         │     RDMA-WRITE data         │
                                         │     + RDMA-WRITE-WITH-IMM ─ │ ──┐
                                         └────────────────────────────┘   │
 ┌───────────────────────────┐                                           │
 │ recv WR 被 IMM 消耗       │ ◄─────────────────────────────────────────┘
 │ → IBV_WC_RECV_RDMA_WITH_IMM│
 │ 完成 (data 到达)           │
 └───────────────────────────┘
```

- **接收方**：`Irecv` 先 post 一个零/小字节 recv WR（捕获 IMM），再把“接收缓冲 addr / rkeys / tag / idx”打包成 FIFO 元素，**RDMA-WRITE 到发送方的 FIFO**（即 CTS，告知“可发、发到这里”）。
- **发送方**：`Isend` 检查 FIFO 里接收方写来的 CTS（`idx == fifoHead+1` 匹配），**未就绪返回 `*request=NULL`**（由 NCCL proxy 线程重试）；就绪后 **RDMA-WRITE 数据**，并以 `RDMA_WRITE_WITH_IMM` 收尾——IMM 消耗接收方 recv WR，在接收方产生 `IBV_WC_RECV_RDMA_WITH_IMM` 完成。
- **Flush**：两边都用 `IBV_WR_RDMA_READ`（signaled）把写可见性“拉”回来。
- **Poll**：两边都是 **demand-driven**——插件内部**没有**自治 progress 线程，前进只发生在 NCCL proxy 线程调用 `Test()` 时。
- **WR 批处理**：两边都只在**最后一条 WR 置 `IBV_SEND_SIGNALED`**（前面 batch 为 unsignaled），都按 `128B` 对齐 chunk（`IB_WRITE_CHUNK_ALIGNMENT`，服务 LL/LL128 协议）。

所有差异都在“如何选 QP、如何传 size、如何记账完成、如何处理多 QP / AR / 容错”。

---

## 2. 发送（Isend）的触发与推进

### 2.1 触发：都是“接收方 CTS 到达”才触发

| 维度 | in-tree `p2p.cc` | plugin `ib_plugin.c` |
|---|---|---|
| CTS FIFO 字段 | `comm->ctsFifo[slot]`（:85） | `comm->fifo[slot]`（:1622） |
| 就绪判定 | `slots[0].idx != idx` → 返回 NULL（:282-285） | `slots[0].idx != idx` → 返回 NULL（:1740） |
| 多 recv 等待 | `while (slots[r].idx != idx);` 自旋（:289） | `while(slots[r].idx != idx);` 自旋（:1743） |
| 内存序 | `std::atomic_thread_fence(seq_cst)`（:291） | `__sync_synchronize()`（:1744） |
| 凑齐 nreqs 判定 | 计数器 `sendReqsCnt[slot] >= nreqs`（:348-350） | 扫描 `reqs[]` 是否全非 NULL（:1790-1792） |

触发逻辑几乎一字不差（连注释都一样）。唯一行为差异：in-tree 用单独计数器 `sendReqsCnt[slot]`，plugin 用“扫描整个 reqs 数组有无 NULL”。

### 2.2 推进：`MultiSend` 的 QP 选择与分片——**最大架构分歧点**

**in-tree（确定性、收发对称的 per-request 映射）：**
- 发送方 `req->id = fifoHead`（:308），接收方 `req->id = fifoHead`（:444）——**两边 id 相同**。
- QP 由 `ncclIbCommBaseGetQpForRequest(base, id, qpIndex)` 选（`common.h:393-400`）：
  ```
  outQpIndex = (id * nQps + qpIndex) % nqps;     // 用 activeQps[]
  ```
  **同一个 request 在 send/recv 两侧算出完全相同的 QP 集合**，分片严格对齐。
- 容错路径有**选择性重传记账** `req->send.sentData[qpIndex]`（:161-170, :252），某 QP 已投递则跳过。
- AR 触发条件：`nreqs>1 || (!(remOooRq && localOooRq) && ar && size>阈值)`（:131-132）——把 OOO 接收队列纳入判断。

**plugin（轮询计数器、非对称）：**
- **`ncclIbRequest` 没有 `id` 字段**（`p2p_plugin.h:66-84`），QP 选择不依赖 request id。
- 用一个**可变轮询计数器** `comm->base.qpIndex`，每 post 一次 `(qpIndex+1)%nqps`（:1678, :1718）；CTS 路径另用一个 `comm->base.devIndex` 轮询（:1820-1821）。
- 分片用 **per-request 的 `req->send.offset`** 累加推进（:1688, :1712），无重记账。
- AR 触发条件：`nreqs>1 || (comm->ar && size>ncclParamIbArThreshold())`（:1654），无 OOO-RQ 概念。

> **影响**：in-tree 的 striping 对任意 request id 都可预测且收发对齐（容错重传、AR 切换路径都依赖此点）；plugin 靠 `qpIndex` / `devIndex` 两个独立计数器在 send/recv 两侧“恰好同步”，更脆弱——换 QP 数或并发投递时存在收发 QP 错配风险（靠 events 计数兜底，但语义上不如前者稳健）。

### 2.3 immData / size 回传——**第二处分歧**

| 维度 | in-tree | plugin |
|---|---|---|
| 匹配方案 | **双方案**，由 `IB_RECEIVER_SIDE_MATCHING_SCHEME` 控制（默认 BY_INDEX，:17, :128） | **单一方案**（index） |
| BY_ID | immData = request id；接收方按 `imm_data % MAX` 取 request（支持完全乱序到达） | — |
| BY_INDEX（nreqs==1） | immData = send size | immData = send size（:1645） |
| nreqs>1 的 size | **RDMA-WRITE 到专门的完成记录** `remCmplsRecords`（:139, :339），接收方读 `req->recv.cmplsRecords->sizes[]` | 通过 `remSizesFifo` / `sizesFifo` 写（:1647-1651）；immData=0 |

plugin 没有 ID 匹配、没有完成记录结构，是 in-tree 的功能子集，但更简单。

---

## 3. 接收（Irecv）的触发与推进

### 3.1 触发：上层调用即立刻“推”出 CTS（两边一致）

- in-tree：`ncclIbPostRecvWorkRequest(qp, &comm->ibRecvWorkRequest)`（:476，复用一个共享 WR）→ `ncclIbPostFifo`（:517）→ `fifoHead++`（:518）。
- plugin：构造局部 `ibv_recv_wr wr`（**`num_sge=0`，纯收 IMM**，:1910-1912）→ `wrap_ibv_post_recv`（:1923）→ `ncclIbPostFifo`（:1930）。

差异细节：
- in-tree 有 **`prepostReceiveWorkRequests`** 模式：跳过即时 post，完成后再补 post（:471-473, :773-780）；plugin 无此优化。
- in-tree 用 `GetQpForRequest`（:469）选 QP（与发送对称）；plugin 用 `qpIndex` 轮询（:1921, :1924）。

### 3.2 CTS 投递（PostFifo）——QP 选择与 signaled 记账不同

| 维度 | in-tree `ncclIbPostFifo`（:364） | plugin `ncclIbPostFifo`（:1809） |
|---|---|---|
| 选 QP | `ncclIbRecvCommGetQpForCts(comm, req->id, &ctsQp)`（:366，per-request） | `comm->base.qps + comm->base.devIndex`，`devIndex` 轮询（:1820-1821） |
| FIFO rkey / sge | `remDevs[].rkey`、`comm->devs[].sge`（:373-378） | 独立的 `fifoRkey` / `fifoSge`（:1840, :1843）——CTS 用专门 MR |
| signaled 触发 | `slot == ctsQp->devIndex \|\| resiliency`（:409） | `slot == ctsQp->devIndex`（:1875） |
| **是否计入 request events** | **否**（PostFifo 内不调 AddEvent；CTS 完成在 Test 里有独立 RDMA_WRITE 分支，:787） | **是**：signaled 时 `ncclIbAddEvent(req, ...)`（:1879） |

> 这条很关键：**plugin 把 CTS 的 signaled 完成直接计入该 recv request 的 event 计数**，所以 plugin 的 recv request 必须**同时**等到“数据 IMM 完成 + CTS signaled 完成”都 poll 到才算 done；in-tree 则把 CTS 完成单独处理，不混入数据完成计数。两边 `events[]` 语义并不相同。

### 3.3 Flush（Iflush）——基本相同

两边都对**每个 device** post 一条 signaled `IBV_WR_RDMA_READ`（in-tree :550-557；plugin :1960-1965），用 `events[]` 计数。结构几乎一致，仅 devBase 取法不同：
- in-tree `ncclIbAddEvent(req, i)` 内部解析（:560）；
- plugin 显式传 `&comm->devs[i].base`（:1967）。

flush 的 wr_id 偏移：in-tree 用 `NCCL_IB_FLUSH_REQ_WR_ID_OFFSET = 0x1000`（`p2p.h:13`）与普通完成区分；plugin 直接用 `req - base->reqs`（:1954），靠 opcode 区分。

---

## 4. Poll（Test）的触发与推进——分歧最集中

### 4.1 触发：都是 proxy 线程外部驱动，无自治 progress 线程（相同）

### 4.2 推进：循环结构与完成处理差异巨大

**循环骨架：**
- **in-tree（:817-876）**：`do { ... } while (totalWrDone > 0)`——**抽干所有设备当前可得 CQE** 再返回；设备遍历用 `r->base->vProps.ndevs`（:836），对 `events[i]==0 && !resiliency` 或 `devBases[i]==NULL` 的设备**跳过 poll**（:841-843）；入口有容错推进钩子 `ncclIbResiliencyProgress`（:821）。
- **plugin（:1976-2081）**：`while(1){ ... if(totalWrDone==0) return; }`——外层再判一次完成；设备遍历用常量 `NCCL_IB_MAX_DEVS_PER_NIC(=4)`（:2004），仅当 `r->events[i]` 非零才 poll（:2007）。

**完成处理（最根本的结构差异）：**

| 维度 | in-tree | plugin |
|---|---|---|
| 因子化 | 抽成 `ncclIbRequestRetrieveFromCompletion`（:582）+ `ncclIbCompletionEventProcess`（:695） | **全部内联**在 Test 里（:2042-2070） |
| opcode 分派 | 显式 4 路：`RECV_RDMA_WITH_IMM`（BY_ID 按 imm / 否则按 wr_id slot）、`RDMA_READ`（flush）、`RDMA_WRITE`（CTS）、send | 仅粗分“send / 非 send”，靠 `req->events[i]--` 兜底（:2050-2070） |
| request 定位 | 按场景分别用 `imm_data` / `wr_id&0xff` / `wr_id-offset`（:594-610） | 统一 `req = base->reqs[(wc->wr_id & 0xff)]`（:2042），多 send 再解码 `(wr_id>>(j*8))&0xff`（:2052） |
| size 回读 | 从 `cmplsRecords->sizes[]` 或 `aggSize`（:636-637） | 直接 `req->recv.sizes[i]`（:1986） |
| 错误处理 | 有容错：`ncclIbResiliencyHandleCompletionError`（:860）；无容错才 `ncclRemoteError` | **无容错**：任何 CQE 错误立即 `ncclRemoteError`（:2037） |
| prepost 重 arm | 完成后 `ncclIbPostRecvWorkRequest` 补一条（:779） | 无 |

> 一句话：in-tree 的 Test 是“**按 opcode 分派、因子化、容错、可重 arm**”的现代实现；plugin 是“**按 wr_id 低 8 位定位、内联减计数、遇错即死**”的早期实现。

---

## 5. 请求对象与常量层面的小差异

| 维度 | in-tree | plugin |
|---|---|---|
| `events[] / devBases[]` 大小 | `[NCCL_IB_MAX_DEVS_PER_NIC]=4`（`common.h:202`） | 同样 `[4]`（`p2p_plugin.h:70`） |
| `GetRequest` 初始化 | `memset(devBases/events, 0, 全部)`（`p2p.cc:27-28`） | **只清 `events[0..1]`、`devBases[0..1]`**（:1463-1465）——4 元数组只清一半，健壮性瑕疵 |
| `ncclIbAddEvent` 签名 | `AddEvent(req, devIndex)`，内部解析 devBase（`p2p.cc:43`） | `AddEvent(req, devIndex, devBasePtr)`，调用方显式传（:730） |
| request 内字段 | 有 `id`、`send.sentData[]`、`recv.cmplsRecords`、`recv.aggSize`、`rmaProxyCtx`、profiling `pInfo[]` | 无 `id`、有 `send.offset`、`recv.sizes*`，**无容错 / profiling 字段** |
| MR 注册 | 分离在 `reg.cc` | **内联页粒度引用计数 MR cache**（`ncclIbMrCache`，:1482-1539）——plugin 相对早期 NCCL 多出、且就在本文件里 |

常量（两边一致）：`NCCL_NET_IB_MAX_RECVS = 8`；`MAX_REQUESTS = NCCL_NET_MAX_REQUESTS * 8`；`NCCL_IB_MAX_DEVS_PER_NIC = 4`。

---

## 6. 差异总表

| # | 维度 | in-tree `transport/net_ib` | plugin `ib_plugin.c` | 影响 |
|---|---|---|---|---|
| 1 | QP/AR 选路与分片 | `id` 驱动**确定性、收发对称** `(id*nQps+qpIndex)%nqps` + `activeQps` + 重记账 + OOO-RQ | 两个独立**轮询计数器** `qpIndex/devIndex`，无对称保证/无重传/无 OOO | 多 QP/AR 时 plugin 收发对齐保证弱 |
| 2 | size 回传与匹配 | **双方案**（BY_ID / BY_INDEX）+ **完成记录结构**（抗乱序、多 recv size） | **单一** index 方案 + sizes FIFO | plugin 不支持乱序到达匹配 |
| 3 | Poll 完成处理 | **因子化、opcode 分派、容错可恢复、可重 arm** | **内联、按低 8 位定位、遇错即 ncclRemoteError、无容错** | plugin 错误即死，无法恢复 |
| 4 | CTS signaled 记账 | 不计入数据 request events（独立分支） | **计入** recv request events | plugin recv done 需等 CTS+数据两路完成 |
| 5 | `GetRequest` 初始化 | 全清 events/devBases | 只清 `[0..1]`（4 元数组清一半） | plugin 健壮性瑕疵 |
| 6 | recv WR 投递 | 共享 WR + `prepostReceiveWorkRequests` 重 arm 优化 | 局部零字节 WR，无 prepost | plugin 无接收 WR 预投递优化 |
| 7 | MR 管理 | 独立 `reg.cc` | 内联页粒度引用计数 cache | plugin 注册更省、dedup |

**共同点**：触发模型（CTS pull、Isend 未就绪返 NULL）、demand-driven poll、unsignaled-except-last、RDMA_READ flush、128B chunk 对齐、events[4] 计数模型——两者一致。

---

## 7. 结论与工程影响

- **in-tree 版**面向现代多路径 / AR / 容错 / 大规模训练设计：重传、动态 `activeQps`、ID 匹配抗乱序、完成记录、prepost 重 arm、resiliency 钩子齐备。
- **plugin 版**更老更直白，适合“单路径或简单 AR、不要容错”的场景；因靠轮询计数器选 QP，多 QP/AR 切换时收发对齐保证弱于 in-tree。
- **若要在 plugin 里做 AR 或容错增强**，`MultiSend` / `PostFifo` 的 QP 选择（改为 `id` 驱动的确定性映射）与 `Test` 的完成记账（opcode 分派 + 完成记录）是必须先对齐到 in-tree 模型的两块。

---

## 附：源码定位索引

### in-tree（`nccl/src/transport/net_ib/`）
- 请求生命周期：`ncclIbGetRequest` `p2p.cc:21`、`ncclIbFreeRequest` `:38`、`ncclIbAddEvent` `:43`
- 发送：`ncclIbMultiSend` `:83`；`ncclIbIsend` `:262`（CTS 检查 :282、自旋 :289、fence :291、`sendReqsCnt` :348、`MultiSend` 调用 :353、`fifoHead++` :355）
- CTS：`ncclIbPostFifo` `:364`（`GetQpForCts` :366、signaled :409）
- 接收：`ncclIbIrecv` `:430`（`PostRecvWorkRequest` :476、`GetQpForRequest` :469、`PostFifo` :517）
- Flush：`ncclIbIflush` `:525`（`RDMA_READ` :550）
- Poll：`ncclIbRequestRetrieveFromCompletion` `:582`、`ncclIbRequestIsComplete` `:620`、`ncclIbRequestComplete` `:629`、`ncclIbCompletionEventProcess` `:695`、`ncclIbTest` `:817`（do/while :828/:872、`vProps.ndevs` :836、skip :841、`poll_cq` :845、resiliency :821）
- 结构体/常量：`struct ncclIbRequest` `common.h:192`（events[4] :202、devBases :207、sentData :219、cmplsRecords :222）；`GetQpForRequest` `common.h:393`、`GetNqpsPerRequest` `:379`；flush wr_id offset `NCCL_IB_FLUSH_REQ_WR_ID_OFFSET=0x1000` `p2p.h:13`

### plugin（`nccl-rdma-sharp-plugins/src/ib_plugin.c` + `include/p2p_plugin.h`）
- 请求生命周期：`ncclIbGetRequest` `:1457`（仅清 [0]/[1]）、`ncclIbFreeRequest` `:1475`、`ncclIbAddEvent` `:730`（3 参）
- 发送：`ncclIbMultiSend` `:1620`（chunk :1687、immData :1643、AR :1654、signaled last :1671、qpIndex RR :1678/:1718、`post_send` :1708、offset :1712）；`ncclIbIsend` `:1724`（CTS 检查 :1740、自旋 :1743、sync :1744、`MultiSend` :1795、`fifoHead++` :1800）
- CTS：`ncclIbPostFifo` `:1809`（devIndex RR :1820/:1821、fifoRkey/fifoSge :1840/:1843、signaled :1875、`AddEvent` 计入 :1879）
- 接收：`ncclIbIrecv` `:1889`（零字节 `post_recv` :1910/:1923、qpIndex :1921/:1924、`PostFifo` :1930）
- Flush：`ncclIbIflush` `:1937`（`RDMA_READ` :1960）
- Poll：`ncclIbTest` `:1976`（while(1) :1980、events[0..3] 检查 :1982、`NCCL_IB_MAX_DEVS_PER_NIC` 循环 :2004、`if(events[i])` :2007、`poll_cq` :2008、`reqs[wr_id&0xff]` :2042、多 send 解码 :2052、`events[i]--` :2057/:2069、`ncclRemoteError` :2037）
- 结构体/常量：`struct ncclIbRequest` `p2p_plugin.h:66`（events[4] :70、send.offset :78、recv.sizes :81，**无 id**）；`NCCL_NET_IB_MAX_RECVS=8` `:25`、`MAX_REQUESTS` `:27`、`NCCL_IB_MAX_DEVS_PER_NIC=4` `:54`；MR cache `ncclIbMrCache` `p2p_plugin.h:49`、`ncclIbRegMrDmaBufInternal2` `ib_plugin.c:1482`
