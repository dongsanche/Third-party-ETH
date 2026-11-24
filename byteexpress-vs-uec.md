# ByteExpress 与 UEC 的对比分析

> 论文:Deploying RDMA for Distributed Inference and Training At Scale(arXiv:2610.12815v2,字节跳动 + 复旦,2026.10)
> 标准:Ultra Ethernet Consortium Specification 1.1(2026.10.12 发布,764 页)

---

## 一、ByteExpress 论文概览

| 项目 | 内容 |
|------|------|
| 系统 | ByteExpress(BE):叠加在硬件 RNIC 之上的用户态软件通信层 |
| 场景 | 字节跳动参数服务器(PS)架构的推荐在线推理 + 离线训练(抖音/头条) |
| 底层 | Mellanox ConnectX-5,RC 传输 over RoCE,DCQCN + PFC |
| 规模 | 推理:300 PS + 124 ranker;训练:124 PS + 346 trainer;生产运行 2 年以上 |
| 生产收益 | 推理 Fetch P99 -43.7%、CPU -10.69%;训练 Push P99 -74.0%,PFC 归零 |

### 工作层级:应用与硬件之间的整个中间层

```
Archon RPC(应用接口)   ← 恢复/重试/副本选择/TCP 回退留在这层
─────────────────────────
ByteExpress(BE)          ← 论文工作主体
  ├─ 内存层:bemalloc 注册内存分配器
  ├─ 传输/会话层:消息 I/O、QP 复用、credit 流控、发送窗口
  └─ 资源层:线程域、CQ/SRQ 组织
─────────────────────────
RNIC 硬件(RC over RoCE)  ← 可靠传输 + DCQCN + PFC 由硬件负责
```

### 核心论点

RDMA 硬件卸载 ≠ 生产可用。变长消息、海量连接、突发 incast、服务故障四类问题必须靠软件跨层解决,硬件只负责可靠传输与速率控制。

---

## 二、三大问题与做法

### 问题 1:零拷贝落地难(内存层)

**难点**:RDMA 要求内存先注册,但消息变长(KB~MB)、应用要持有/转交/释放缓冲区;固定接收环走不通,逐条注册又太贵。

**做法**:
- **bemalloc**:在 rpmalloc 上嫁接注册钩子,4MB 大块触发 `ibv_reg_mr`,注册按块摊销;`GetMR` O(1);池稳定后(~2.2GB)注册退出稳态路径
- **零拷贝所有权**:RNIC 直写按需分配的注册缓冲,指针直接移交应用线程;发送方同样用 bemalloc 分配载荷
- **大消息传输**:SEND/RECV 交换地址+RKey(1 RTT)→ 分片单边 WRITE → 末片 **WRITE_WITH_IMM** 通知;靠 CQE 完成语义而非 payload flag(relaxed ordering 下 flag 不可信)

### 问题 2:连接与资源扩展(传输层)

**难点**:QP 上下文缓存有限,海量逻辑连接挤爆;多线程下 `ibv_post_send` 内部自旋锁在 64+ QP 后成为瓶颈。

**做法**:
- **QP 多路复用**:socket = 逻辑连接,engine = RC QP;按"远端地址 → engine"共享,调度器协调
- **分片轮转**消 HoL 阻塞:一次只提交一个分片,轮转调度(25G 用 2KB 分片,100G 需 >8KB)
- **WR 链三段式**(in-flight/pending/free)+ doorbell batching,批内仅最后请求要 CQE
- **线程所有权对齐 provider**:per-thread Thread Domain + CQ + 可选 SRQ,engine 无锁
- **receive credit** 防 RNR:SEND/WRITE_WITH_IMM 消耗对端 receive,按 credit 发送,批量(k=d/2)补充

### 问题 3:incast 尾延迟(拥塞层)

**难点**:训练迭代持续 fan-in,聚合流量超过交换机缓冲;DCQCN 反应慢,纯硬件控制下 P99 反而比 TCP 差(Fetch 1.04s vs 650ms),每 30s 出现 1s PFC 暂停。

**做法**(职责分离):
- **per-engine 软件发送窗口**限制在途字节,配合分片;窗口/分片大小按消息尺寸与 incast 规模静态配置
- 三种控制各司其职:软件窗口管**在途数据量**,DCQCN 管**发送速率**,PFC 是**链路层兜底**
- 效果:4KB 窗口把触发 PFC 的规模从 63 台服务器推到 1218 台;生产 Push P99 1.12s → 268ms,PFC/NAK 归零

### 可用性(第 4 层,顺带)

连接 keepalive → 换路径(port-affinity)→ 换 mini-pod 副本 → TCP 回退(1 分钟内恢复);故障节点隔离(5 分钟 30 次连接/50 次请求失败进黑名单)。

---

## 三、与 UEC 的逐项对比

BE 是"UEC 标准化之前,大型云厂商用软件在 RoCE 上摸出来的同款答案";UEC 1.1 则把这些机制标准化进以太网传输层本身。

| 维度 | ByteExpress(软件层,私有部署) | UEC 1.1(标准规范) |
|------|------|------|
| **定位** | 应用与 RNIC 之间的用户态通信库 | 从 PHY 到软件层的完整开放协议栈,多厂商互操作 |
| **传输语义** | 复用 IB RC verbs,SEND/RECV + WRITE/RKey | 重新定义 UET(SES/PDS/CMS/TSS 四个子层),改进 RDMA 语义(多路径、消息语义现代化) |
| **限制在途数据** | per-engine 软件发送窗口(静态配置,非自适应) | RCCC 接收端 credit 拥塞控制(规范化的 credit 授予);NSCC 窗口式 |
| **接收容量保护** | receive credit 防 RNR,批量补充 k=d/2 | CBFC(基于 credit 的链路层流控)标准化,替代 PFC |
| **拥塞信号** | 不改硬件,依赖 DCQCN(慢)+ 软件窗口兜底 | CSIG 4/8B 带内逐跳遥测(min/max 聚合,精确定位瓶颈),NSCC/RCCC 直接消费;可编程拥塞(PCM) |
| **多路径** | 无(单路径 RC,靠副本选择换路径) | UET 原生多路径 in-order delivery,熵值负载均衡 |
| **连接扩展** | QP 复用(socket→engine 映射)+ 线程域 | UFH + UET 无传统 QP 概念;AI Base profile 最小化实现复杂度 |
| **大消息传输** | 元数据交换 + 分片 WRITE + WRITE_WITH_IMM | UET WRITE/READ(4GB-1),AI Base 同样以 WRITE 为主 |
| **零拷贝/内存** | bemalloc 块级注册摊销,所有权传递 | TSS 加密与 MR 语义由规范定义;libfabric/verbs 映射标准化(UET Verbs) |
| **消息大小优化** | 分片调度缓解 HoL | Unified Local profile 12B/8B 压缩头 + 消息打包(16 条/包)+ UFH + 减 IPG |
| **可用性** | RPC 层重试/副本/TCP 回退 | 传输层可靠(RUD/ROD/RODL)+ 端到端安全;服务级恢复留给上层 |
| **生态** | 字节内部(Archon RPC),封闭 | Linux 基金会开放标准,百余家会员,产品已落地(Broadcom Thor Ultra 800G NIC) |

### 机制层面的对应关系

```
BE 软件发送窗口(限在途字节)      ↔  UET RCCC credit / NSCC 窗口
BE receive credit(防 RNR)       ↔  CBFC credit-based 流控(替代 PFC)
BE 分片 + 轮转调度               ↔  UET 消息分片 + 多路径熵值调度
BE HoL 缓解                      ↔  UET 多路径 in-order delivery
BE 依赖 DCQCN(硬件速率控制)     ↔  CSIG 遥测驱动拥塞控制(信号更精细)
BE QP 复用                       ↔  UEC 抛弃 per-connection QP 模型
BE TCP 回退保可用                ↔  UE 与标准以太网帧共存/网关翻译
```

### 根本差异

1. **改造层次不同**:BE 不能动 RNIC 和网络,只能在 NIC 之上"打补丁";UEC 直接重定义 NIC 与交换机的行为(新传输协议、CSIG 标签、INC 网内归约、UFH)
2. **拥塞信息质量**:BE 时代只有 DCQCN 的粗粒度反馈,被迫用静态软件窗口兜底;UEC 1.1 的 CSIG 提供逐跳、定量、带定位的信号,使自适应拥塞控制(NSCC/RCCC/PCM 可编程)成为可能——这正是 BE 论文中承认的局限("窗口非自适应、未证明公平性")
3. **互操作 vs 私有优化**:BE 是单厂商软件层,收益立竿见影;UEC 是多方标准,需要 1.0→1.1 演进和 OCP/IEEE 协同才能逐步兑现

---

## 四、启示

- BE 的生产数据实证了"纯硬件拥塞控制不够、必须在途数据量受控"(加软件窗口后 Push P99 从比 TCP 差 60% 变为好 74%),这为 UEC 把拥塞控制、credit 流控做进传输层规范提供了工业依据
- 字节跳动本身是 UEC General member,其网络团队(Yibo Zhu 等)长期深耕 RoCE 拥塞控制(DCQCN 作者)——BE 的经验大概率会反哺其在 UEC/超以太生态中的标准化路线
- 对部署者:现有 RoCE + RC 上用 BE 式软件层可立即获得收益;新建 AI 集群则可关注 UEC 兼容硬件(200G/lane、CSIG、INC)的成熟节奏
