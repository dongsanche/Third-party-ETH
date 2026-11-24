# UEC 技术深入解析:讨论纪要与 ByteExpress 解法对照

> 承接《byteexpress-vs-uec.md》,本文整理 UEC(Ultra Ethernet Consortium)1.1 规范的深入技术讨论:
> PCM 可编程拥塞控制、INC 在网计算、传输层报文格式、PDC/FEP 概念体系、安全与密钥管理,
> 以及与 RoCE RC 的系统对比、与 ByteExpress 软件解法的逐点对照。
> 依据:UE Specification 1.1(2026.10.12,764 页)+ arXiv:2610.12815v2(ByteExpress)

---

## 目录

- [一、PCM 可编程拥塞控制](#一pcm-可编程拥塞控制)
- [二、INC 在网计算](#二inc-在网计算)
- [三、UEC 特性全景与传输层](#三uec-特性全景与传输层)
- [四、报文格式](#四报文格式)
- [五、PDC / FEP / FEC 概念体系](#五pdc--fep--fec-概念体系)
- [六、安全、密钥与作业管理](#六安全密钥与作业管理)
- [七、与 RoCE RC 的系统对比](#七与-roce-rc-的系统对比)
- [八、ByteExpress 解法与 UEC 方案对照](#八byteexpress-解法与-uec-方案对照)
- [九、常见问题速查](#九常见问题速查)

---

## 一、PCM 可编程拥塞控制

### 1.1 定位

PCM = 把拥塞控制(CC)和负载均衡(LB)算法做成可加载的"程序",运行在 FEP(网卡)端点。
类比:功能上像 QUIC 的可插拔拥塞控制,但执行位置在 NIC 硬件;工程形态像 eBPF——
C 写算法 + YAML 声明资源 + 加载器校验后注入数据面。内置的 NSCC/RCCC 就是"写死的算法",
PCM 把这个机制泛化给用户。

### 1.2 四组件逻辑架构

```
输入信号(Signals) ──► CC 算法(窗口/credit)──► 控制表(Control Table)──► 数据面发送行为
                 └─► LB 算法(熵值/路径选择)──┘
```

### 1.3 编程模型(C + YAML)

- 算法用 C 编写,链接 `uet_pcm.h`,通过函数 API 读写资源(规范刻意选函数调用而非结构体访问,
  便于编译器内联成对特殊地址的 load/store)
- 必选 API:`uet_signal_get`、`uet_set_cwnd`;可选:`uet_set_send_rate`、ACK 合并参数、
  电报 API(`uet_send_tgm_lcl/peer`)、控制包(`uet_send_ctrl_probe/credit`)、
  LB 表(`uet_get/set_ev_field_*`)
- YAML 加载时声明:信号(累加算子/触发器)、控制输出、局部/全局变量、参数、电报、LB 算法
  (ev-table 模式的 hw_algo/ev_fields/ccc_fields)
- 整数类型必选、浮点可选;`parameter` 用于同算法实例化不同"人格"(如一个 NSCC 管 AI、
  一个管存储)

### 1.4 信号与触发器

- 必选信号约 20 个(RTT 128ns 粒度、ECN、trim、NACK、credit 等),可选含 **CSIG 信号**
  (配 `csig_dequantize(t, s)` 反量化),厂商可扩展
- 累加算子:sum/max/min/last(必选)、ewma(可选);同一包的多信号更新原子生效;
  算法调用时取快照
- 触发器在 YAML 声明(比较算子 eq/gt/…,fire 模式 count/time,arm 模式
  DISARMED/ONCE/CONTINUOUS,action INVOKE/RECORD);例:每 2ms 调用一次

### 1.5 运行时契约(算法必须假设)

- 同一 CCC 同时最多一次调用(无并发)
- TX/RX 事件各自按包序,但相互间无顺序保证,且**事件可被跳过**(算法要容忍)
- TIME 触发可靠送达但可延迟;电报按到达序;控制操作按发出序

### 1.6 电报、错误处理、生命周期、eBPF

- **Telegram**:≤30B 载荷 + 8bit type_id 的算法间带外消息,走 PDS Telegram CP(32B 控制包);
  可发本地(netlink/dmesg/厂商守护进程)或对端 CCC,支持 collector 做集中遥测
- **错误处理**:三类致命条件(硬件故障/算法非法操作/算法主动 `uet_ccc_fatal`)→ CCC 原子
  ACTIVE→ERROR,在途包全弃、后续 API 返回 `-ENOTRECOVERABLE`,绝不带病运行;恢复只能靠
  PCM 之外的 reset
- **生命周期**:Load → Instantiate → Configure → Assign(按 UET 实例)→ Activate → Stop →
  Destroy → Unload
- **eBPF 合规(可选)**:有界循环、禁递归、指针可验证、栈 ≤512B、调用深度 ≤8、尾调用 ≤32
- LB 两种模型:per-packet(逐包纯软决策)与 **EV 表**(硬件维护熵值表逐包选路,
  软件算法只调校表项——硬件做决策、软件做策略)

### 1.7 载体:必须在 DPU 上吗?

规范不指定物理载体,只要求运行在 FEP。三种落地:普通 RNIC 固件/硬件引擎(算法离线编译)、
**DPU/智能网卡 ARM 核跑 C/eBPF(最自然,eBPF 合规就是为此设计)**、主机软件参考实现。

---

## 二、INC 在网计算

### 2.1 做什么

把集合通信的计算下沉到交换机:数据边转发边归约。
传统 all-reduce 每端点发 2GB 产 1GB 结果;INC 只发 1GB → 通信量减半,最高 2x 加速;
小集合操作砍掉树形广播的 log n 跳时延。对标 InfiniBand 的 SHArP。

### 2.2 操作与算子

- 操作码:REDUCE/ALLREDUCE/BARRIER(必选)、BCAST/SYNCBCAST(二选一)、EXSCAN、
  厂商自定义 0xF0–0xFF
- 归约算子:MIN/MAX/SUM/SUM_SCALE(广义平均)/PROD/LOR/LAND/BOR/BAND/LXOR/BXOR/
  MINLOC/MAXLOC,编码与 UET 原子操作复用
- 架构:IM 守护进程(gRPC)建 communicator(唯一 CID)→ rank 发 DEFERRABLE_SEND
  (64bit inc_info:ver/rt/ip/fo/op_type/cid/csn/data_type/reduce_op)→ 逻辑 SEP 交换机
  按 JobID/CID 过滤、归约、分发;CSN 12 位 → 最多 4096 个在途操作

### 2.3 两端都要改

| 侧 | 改动 |
|---|---|
| 交换机(SEP) | 解析 UET 语义头与 inc_info、执行归约、保留 ROD PDC(必选);TFC/TSS/YANG(可选) |
| 端点 FEP | DEFERRABLE_SEND 编码 inc_info、保留 PDC、(可选)RTR/TFC/TSS;主机软件:libfabric 扩展(comm_key/集合 API)、IM 部署 |

不能只改一头:INC 信息编码在 UET 头里,普通 RDMA 网卡发不出;普通交换机看不懂也不会算。
INC 是纯可选特性,建组时 IM 经 YANG 判断交换机胜任与否,失败退回普通集合通信,可分阶段部署。

---

## 三、UEC 特性全景与传输层

### 3.1 特性全景(分层)

```
软件层   libfabric / UET Verbs / IM / PCM 算法加载
传输层   UET = SES(语义)+ PDS(交付)+ CMS(拥塞)+ TSS(安全)
网络层   UFH | CSIG | Packet Trimming | DSCP 流量类映射
链路层   LLR | CBFC | Idle Reduction | LLDP 能力协商
物理层   100G→800G、200G/lane | CtlOS | FEC 统计
管理面   Redfish | YANG | SAI | 性能基准规范
```

要点:
- **LLR**:逐跳重传,使 UEC 可跑在尽力而为网络(不强依赖 PFC)
- **CBFC**:credit 制流控,PFC 替代品(同缓冲撑更多无损 VC、credit 占用可指导调度)
- **Packet Trimming**:缓冲耗尽时裁掉 payload、包头换流量类转发——丢包变显式拥塞通知
- **TSS**:端到端零信任加密,PSP 式 KDF 解决百万端点密钥状态爆炸
- profile 套餐:AI Base / AI Full / HPC / Unified Local(1.1 新,scale-up)

### 3.2 传输层新特性(四子层)

**SES**:Deferrable Send(接收方没 post 缓冲可延迟交付)、Rendezvous(协议内建)、
tagged/multi-receive/use-once 缓冲模型(Portals 4 血统)、JobID 四级寻址+授权、
显式内存模型、原子扩展(BF16/FP16/FP8/CAS 变体/MSWAP/INVAL/原子读写)、
Sync Group [1.1]、消息打包、压缩头 12B/8B

**PDS**:交付模式矩阵 RUD/ROD/RUDI/UUD(+RODL [1.1])——可靠性与有序性解耦;
多路径保序(包级熵值喷洒+接收端重排);PDC 动态连接

**CMS**:NSCC/RCCC(1.1 起吃 CSIG)、PCM、TFC、多路径选择/EV 喷洒、ACK 合并可配置

**TSS**:传输内建加密(可选)、可扩展密钥、重放保护/epoch、TSS over UFH_32 [1.1]

### 3.3 显式内存模型(§3.4.9)五维度

| 维度 | 规定 |
|---|---|
| 消息序 | ROD 按流保序(flow = 发起方 PIDonFEP → 目标 {PIDonFEP, RI});RUD/RUDI 无序 |
| 完成序 | DC 位 → 完成通知必须等全局可观测;线协议不提供目标端全序完成 |
| 数据序 | 默认松弛(包内字节可乱序落地);ROD 上 ≤16B 原子可选 RAW/WAW/WAR |
| 一致性/原子性 | 单元素粒度;并发原子必须经典一致;普通 write/read 与原子不保证一致——要一致用 UET_AMO_WRITE/READ |
| 幂等 | ROD/RUD 恰好一次;FETCHING_ATOMIC 必须缓存旧值防重试 |

### 3.4 原子操作扩展

- 操作码:MIN/MAX/SUM/DIFF/PROD/逻辑位运算/**原子 READ/WRITE**(为一致性而生)/
  CSWAP + NE/LE/LT/GE/GT 变体/**MSWAP 掩码交换**/**INVAL 缓存失效**(配 DC 位)/厂商码点
- 数据类型:INT/UINT 8–128、FLOAT/DOUBLE/COMPLEX + **AI 专供:BF16、FP16、
  FP8 E4M3/E5M2(OCP OFP8)**;INC 归约复用同一编码
- 语义控制:UET_AMO_CTRL_CACHEABLE(可缓存)、UET_AMO_CPU_COHERENT(与 CPU 相干,
  可用 PCIe 原子实现)
- 矩阵规则:PROD 全部可选、128 位硬操作可选、无意义组合排除

### 3.5 交付模式矩阵(§3.5.7)

| 模式 | 保证 | 顺序 | 恢复 | 路径 | 目标端状态 |
|---|---|---|---|---|---|
| RUD | 恰好一次 | 无 | 选择性重传 | **多路径喷洒** | PDC 状态 |
| ROD | 恰好一次 | 保序 | GoBackN | **单路径单熵值** | PDC 状态 |
| RODL [1.1] | 恰好一次 | 保序 | 可选关闭 | 单层交换 | 压缩 PDC |
| RUDI | 至少一次(幂等) | 无 | 每包响应+NACK | 任意 | **零状态** |
| UUD | 尽力而为 | 无 | 无 | 任意 | 无 |

不可靠有序模式故意不定义。PDC 规则:一消息一 PDC、消息不交错、PDC 间无序;
profile 映射:AI Base=RUD+UUD,AI Full 加 ROD/RUDI,HPC=RUD/ROD/UUD,
Unified Local=RODL。

### 3.6 跨"连接"乱序问题

- 跨 PDC 顺序**不保证**(刻意设计,顺序与并行不可兼得)——与 IB 相同
- 但 UEC 把"要排序只能挤一个 QP"变成工具箱:
  1. 多 flow 共享一个 ROD PDC(跨逻辑连接全序)
  2. **Sync Group**:组内全部写落地才触发完成/原子——专解"数据+标志"模式
  3. DC 位/全局可观测:看到完成 = 数据一定可见
  4. UET_AMO_WRITE/CSWAP 做标志位(原子一致性)
  5. CQE 完成语义而非内存 flag 轮询
- 单连接多路径乱序才是 UEC 真正解决的:RUD 每包独立熵值 + 接收端按 PSN 重排
- 需要全局全序的场景仍靠上层 barrier(RUDI 的设计哲学)

---

## 四、报文格式

### 4.1 完整头部栈(L2 之上)

```
┌─ Ethernet MAC DA/SA(12B;UFH 时即 UFH 编码)
├─ [可选标签] VLAN / MACsec / CSIG(4B 或 8B)
├─ EtherType:0x0800 / 0x86DD / 0xE790(UET over UFH)
├─ L3 封装三选一:
│  A. IPv4/v6(20/40B,DF=1 禁分片)+ UET 熵值头(4B)
│  B. IPv4/v6 + UDP(8B,src_port 当熵值;此时禁熵值头;checksum=0)
│  C. UFH_32(13B)/ UFH_16(8.5B)+ UFH 熵值头(2B)
├─ [TSS 加密头,可选](pds.type=1)
├─ PDS 头(~12B)
├─ SES 语义头(8~44B)+ 扩展头
├─ Payload(≤ PMTU;Unified Local 可打包 ≤16 条消息)
├─ [UET CRC,可选,紧贴 FCS]
└─ Ethernet FCS
```

硬性规定:FEP 必须同时支持 IPv4/IPv6/UDP 三种封装;**熵值位置刻意对齐 UDP src_port**
(存量交换机 ECMP 哈希零改动);IP 分片禁止。

### 4.2 PDS 头

- Prologue(2B):type(5b,15 种包型)+ next_hdr/ctl_type(4b,指示 SES 头类型)+ flags(7b)
- RUD/ROD Request(~12B):clear_psn_offset(16)+ psn(32)+ spdcid(16)+
  dpdcid/pdc_info(16,syn=1 时 overload)
- ACK_CC:基础 12B + cc_type + mpr(8b,在途窗口上限)+ sack_psn_offset(16)+
  **sack_bitmap(64b,选择性确认位图)** + ack_cc_state(64b)
- ACK_CCX:ack_ccx_state 128b 扩展区(1.1 起装 **CSIG 反射头**——网内遥测捎回源端)
- UUD 仅 4B;RODL over UFH_16:flags 3b、PDCID 12b、**期望 PSN 折进 CRC**

### 4.3 SES 标准头(44B,som=1)

```
|rsvd|opcode(6)|ver|dc|ie|rel|hd|eom|som|      ← 操作码+控制位
|message_id(16)| ri_generation(8)|             ← 消息 ID
|JobID(24)|rsvd|PIDonFEP(12)|rsvd|RI(12)|      ← 四级寻址(JobID 刻意=VXLAN VNI 宽度)
|buffer_offset(64)|                            ← deferrable send 时=重启令牌
|initiator(32)|
|match_bits / memory_key(64)|                  ← 按操作码二义复用
|header_data(64)|                              ← som=0 时换 payload_length+message_offset
|request_length(32)|
```

字段复用巧思:buffer_offset⇄restart token、match_bits⇄RTR 令牌回显;
优化头无 message_id 时用原始 PSN 反查。优化头阶梯:32B(medium)/20B(非匹配)/
12B(Unified Local ac=0)/**8B(ac=1,32b 相对偏移链式压缩)**;响应头 16B/8B/4B。

---

## 五、PDC / FEP / FEC 概念体系

### 5.1 FEP(Fabric Endpoint)

**网卡里"会说 UET 协议的实体"**——UET 协议栈的终点,与物理载体无关(NIC 管线/DPU ARM 核/
纯软件皆可)。管四级寻址 {FA, JobID, PIDonFEP, RI}、授权、内存注册、PDC 表、密钥。

```
┌──────────── 主机 ────────────┐
│ 进程A(Job100) 进程B(Job100) │
└───────┬──────────┬──────────┘
┌───────▼──────────▼──────────┐
│ FEP(UET 栈:SES/PDS/CMS/TSS)│ ← 有自己的地址 FA
└───────┬─────────────────────┘
      以太网(交换机不终结 UET)
```

### 5.2 PDC(Packet Delivery Context)

**FEP 为"向另一个 FEP 可靠送包"维护的账本**:
发送侧 {下一个 PSN、SACK 位图、重传参数} + 接收侧 {期望 PSN、重排缓冲} + 关联 CCC。

- **建连零开销**:首包 syn=1 捎带建立(建连信息藏在第一个数据包里),Close CP 拆除
- 与 QP 的本质区别:QP 是静态、先建后用、语义寻址绑定的重对象;PDC 是按需、毫秒级生灭、
  与"发给谁"无关的交付载体
- PDC 是 **FEP↔FEP** 的,不是 job↔job:同一 PDC 可承载同一对 FEP 间不同 Job 的包
  (给哪个 Job 由 SES 头的 JobID 决定)
- 应用全程无感;特例:INC 保留 PDC 由 IM 预建(管理面为集合通信做的优化)
- 生命周期类比:TCP 握手(1 RTT 纯开销)→ IB CM(更重)→ **UET 首包即数据(零等待)**

### 5.3 FEC(前向纠错)与三层纠错体系

物理层 RS 纠错码(如 RS(544,514)),接收端自己修少量误码,无需重传。
UEC §6.3 用 FEC 纠错计数**预测链路劣化**(纠错数上升 = 光模块老化预警)。

```
第 1 层 FEC(物理层):小误码就地纠正        ← 处理 99% 误码
第 2 层 LLR(链路层):FEC 纠不过 → CRC 错 → 逐跳重传
第 3 层 PDS(传输层):端到端选择性重传      ← 兜底,使尽力而为网络可跑
```

### 5.4 "下两层支持就能联通吗"

- 链路级连通"免费送"(就是以太网);**UEC 语义互通的硬门槛在两端 FEP 的传输层+软件层**
- 中间交换机反而可以什么都不支持:UET over IP 就是普通包,RUD/ROD 自带端到端可靠性
- 增强特性(CSIG/INC/LLR/CBFC/UFH)逐台可选、逐链路 LLDP 协商——不支持只是降级不是断连
- 架构哲学:"把智能推向两端、把网络保持傻瓜"

### 5.5 DPU 必须支持完整协议栈吗?

**不需要**。合规体系是声明式的:最小合规 = 标准以太网 PHY + 一个 profile 的 UET + 软件层。
最小端点 = AI Base(SES 只要 SEND/DATAGRAM/WRITE/原子;PDS 只要 RUD/UUD;CMS 只要 NSCC)。
其余全部可选/可单独声明(合规声明格式如 "AI-BASE, LL-LLR")。
互通三规则:两端同 profile、可选特性取最小公分母、非必选特性必须可关闭。
DPU 的额外甜点:软件层/PCM 算法/IM 可跑在 ARM 核,硬管线面积按 profile 可控。

---

## 六、安全、密钥与作业管理

### 6.1 PDC 与安全(常见误解)

PDC 从来不是安全机制(IB 的 QP 同样不是——QP 号可伪造)。
UET 安全分两层:
1. **SES 授权(常开)**:逐包查 ①JobID 有效 ②内存 key 正确 ③访问合法(五元组
   {FA, JobID, PID, RI, Key} 作用域)——能力令牌式防护,防越权不防中间人
2. **TSS(可选)**:端到端加密+完整性+防重放,交换机零信任,可扩展密钥

反转:PDC/PDS 反而给安全打工——§3.7.3 原话 "replay protection relies on PDS services",
PDC 的判重(恰好一次)= 防重放地基。

### 6.2 key 怎么拿到(四条路径)

| 场景 | 方式 |
|---|---|
| SEND 家族 | 不需要 key(JobID 即授权) |
| Rendezvous 大消息 | **扩展头自带 read 寻址(含 key)**——零预交换 |
| scale-up(Unified Local) | JobID 直接寻址,无独立 key |
| scale-out 通用 RMA | 应用层消息捎带 rkey 交换 / 作业级结构化分发(job-based MR、相对寻址、derived MR) |

真正"提前"建立的只有**作业授权上下文**(作业启动时编排器推表),不是连接。

### 6.3 管理角色分工

```
作业启动器/编排器(特权实体)──UET Control API(netlink)──► FEP 内核驱动 ──► 硬件表
      │                                                        ▲
      └── SDME(安全域管理实体,可内嵌于作业启动服务)────────────┘
```

- **JobID**:MUST 由特权实体分配,下发 {PID, 服务名} → {JobID, UET 地址, 安全绑定} 映射;
  用户态传来的 JobID 必须在硬件上下文校验(防 VM 伪造);支持一进程多 JobID
  (FI_AV_AUTH_KEY 选择);有 fallback 机制
- **SDME 职责**:建安全域/分配 SDI、SSI;分发安全参数(SDK/AN)写入各 FEP 的 SDKDB;
  管加入/退出/epoch/**密钥轮换**(FEP 到调用上限前告警 → 换 AN 双收过渡);
  可与作业启动服务合体;SDME↔FEP 协议规范故意不管
- **KDF 消解 O(N²) 钥匙问题**:Direct 每域 1 条记录;Cluster 按 SSI 派生;
  **Client-Server 模式服务器只存 1 把主钥匙,新客户端零服务器配置**(FaaS 场景)
- 钥匙调用上限 2²⁷–2³⁴·⁵ 包;cluster 模式隐式换钥(计数器位段喂 KDF,零管理流量)

### 6.4 运行时新增访问新 Job

动态多 JobID(§2.2.4)支持运行时增删:
1. B 侧:Job B 启动时已配好映射(或早已在运行)
2. A 侧:特权实体运行时经 UET Control API 给 A 的 {PID, service} **追加** JobID-B 映射
   (netlink 一条消息,即时生效)
3. 应用用 FI_AV_AUTH_KEY 选 JobID-B;key 走 rendezvous/消息交换;PDC 自动
4. 加密场景:SDME 给新成员分 SSI+SDK;client-server 模式服务器零操作

与 IB RC 对照:RC 动态新对端 = 关键路径建连;UET = 管理面一条配置,数据面首包直达。

---

## 七、与 RoCE RC 的系统对比

### 7.1 十项痛点对照

| RoCE RC 的问题 | UEC 的改进 |
|---|---|
| 单路径+队头阻塞(ECMP 按流哈希、丢一包整流停) | RUD 多路径包级喷洒+接收端重排 |
| 依赖 PFC 无损网络(风暴/死锁/级联) | 可靠性上收传输层;可选 LLR/CBFC |
| DCQCN 信号粗反应慢(ECN 1bit、定时器升速) | CSIG 逐跳遥测+NSCC/RCCC+PCM |
| QP 连接扩展性(N² 上下文、provider 锁) | PDC 动态+JobID 寻址解耦 |
| 语义一刀切(不需要顺序也付顺序税) | 交付模式矩阵+4 profile |
| 内存模型模糊(flag 轮询不可信) | 显式内存模型+DC/GO+原子读写 |
| 无端到端安全 | TSS+PSP 式 KDF |
| 头开销大(~74B 固定) | 优化头 8~32B、UFH、消息打包、减 IPG |
| 无遥测/在网计算 | CSIG+INC |
| 拥塞控制写死 | PCM 可编程 |

### 7.2 三大核心改进详解

**改进 3:拥塞控制信号质量**
- DCQCN 缺陷:ECN 1 比特路径级布尔量、位置未知;CNP 反馈环 1 RTT;升速靠定时器爬坡;
  实证:ByteExpress 生产环境纯 DCQCN 下 Push P99(1.12s)比 TCP(1.03s)还差,
  每 30s 一次 PFC 暂停
- UEC 四件套:CSIG(min/max 逐跳聚合+定位元数据+反射)→ NSCC(用 max(Delay) 修公平性、
  min(ABW) 精确抢占)→ RCCC(credit 决策吃 CSIG)→ PCM(算法可换)

**改进 4:连接扩展性**
- RC 缺陷:O(N²) QP + CM 建连仪式 + 寻址绑死连接 + provider 锁(8 线程 64+ QP 消息率崩)
- UEC:PDC 首包捎带建立(零握手)、FEP↔FEP 而非 job↔job、毫秒级生灭、
  JobID 寻址与连接彻底解耦(通信在语义层无连接);1 万进程全互联 = 1 万条表项
  (RC 要 ~5000 万条 QP)

**改进 5:语义与内存模型**
- RC 缺陷:全语义一刀切、意外消息只有 RNR、rendezvous 靠库模拟、内存模型靠猜
- UEC:profile 分档(AI Base 最小硬件)+交付模式矩阵(可靠性/有序性解耦)+
  deferrable send/rendezvous/sync group+显式五维内存模型(排序/一致性/原子性/
  全局可观测/幂等全部写死)

### 7.3 UE 诚实的代价

- RUD 接收端重排窗口是新硬件成本
- 生态从零起步(对比 RoCE 15 年惯性)
- ROD 档位仍是 GoBackN+单路径(没有超越 RC,只给了别的排序工具)
- 交换机智能是可选项——全用普通交换机时性能红利吃不满

---

## 八、ByteExpress 解法与 UEC 方案对照

ByteExpress(BE)是在 UEC 标准化之前,字节跳动用软件在 RoCE 上摸出的"同款答案";
UEC 把这些机制标准化进以太网传输层本身。逐问题对照:

### 8.1 拥塞/在途数据控制

| | ByteExpress | UEC |
|---|---|---|
| 限在途字节 | per-engine **静态软件发送窗口**(4/8/16KB 手工配) | **RCCC** credit 授予(自适应,吃 CSIG 信号)+ NSCC 窗口 |
| 效果 | 窗口 4KB 把触发 PFC 的规模从 63→1218 台;Push P99 1.12s→268ms,PFC 归零 | 规范化协议,信号驱动自适应 |
| 哲学 | "软件窗口管在途量、DCQCN 管速率、PFC 链路兜底"职责分离 | 同样职责分离,但每档都有精确信号(改进 3) |

### 8.2 连接/资源扩展

| | ByteExpress | UEC |
|---|---|---|
| 连接复用 | socket→engine 多路复用(按远端地址共享 RC QP)+ 分片轮转消 HoL | PDC 动态建立+多流复用+JobID 寻址解耦(消灭 QP 概念) |
| provider 锁 | Thread Domain + 每线程 CQ/SRQ + WR 链 doorbell batching | 规范不约束实现;协议级 ACK 合并/SACK 降反馈开销 |
| 本质 | engine 仍是 RC QP,IB 传输语义原封不动(压缩连接数但不换语义) | 传输语义重构,状态收进弹性 PDC |

### 8.3 内存/零拷贝与 key 交换

| | ByteExpress | UEC |
|---|---|---|
| 注册内存 | bemalloc 块级注册摊销(rpmalloc 钩子,稳态 ~2.2GB) | 协议管 MR 语义/授权;分配器是实现自由 |
| 大消息 | SEND/RECV 交换元数据(1 RTT)→ 分片 WRITE → WRITE_WITH_IMM 通知 | **Rendezvous 扩展头自带 read 寻址(含 key)**,eager 部分零等待 |
| flag 可信性 | 用 CQE 完成语义而非 flag 轮询(relaxed ordering 教训) | 显式内存模型+DC 位+sync group+原子读写,规范级答案 |

### 8.4 可用性

| | ByteExpress | UEC |
|---|---|---|
| 故障恢复 | keepalive→换路径→换副本→TCP 回退;黑名单隔离 | 传输层可靠(RUD/ROD)+TSS;服务级恢复留给上层(同样哲学) |

### 8.5 结构性差异

1. **改造层次**:BE 不能动 RNIC/网络,只能 NIC 之上打补丁;UEC 直接重定义 NIC 与交换机行为
2. **拥塞信息**:BE 时代只有 DCQCN 粗反馈,被迫静态窗口;UEC 的 CSIG 使自适应成为可能
   ——正是 BE 论文承认的局限("窗口非自适应、未证明公平性")的标准级补全
3. **互操作 vs 私有**:BE 单厂商软件层收益立竿见影;UEC 多方标准需生态换血
4. 字节跳动本身是 UEC General member——BE 的经验大概率反哺其 UEC 路线
5. 两个补丁与两个标准方案同构:软件发送窗口↔RCCC、receive credit↔CBFC、
   分片轮转↔UET 分片多路径、QP 复用↔PDC、元数据交换↔Rendezvous 头

---

## 九、常见问题速查

**Q:PCM 编程在 DPU 上做吗?**
规范只要求运行在 FEP,DPU 是最自然载体(通用核跑 C/eBPF),但纯硬件 NIC 也可
(算法离线编译成指令/微码)。整数必选浮点可选、eBPF 合规都为硬件铺路。

**Q:INC 只改交换机行吗?**
不行,两端都要改:交换机要解析 inc_info+归约执行单元;端点 NIC 要 UET 的
deferrable send/保留 PDC,主机要 libfabric 扩展+IM。但 INC 纯可选,可分阶段部署。

**Q:支持下两层就联通?**
链路级连通免费(就是以太网);UEC 语义互通要求两端 FEP 支持至少一个 profile 的 UET;
中间交换机可以是普通以太网交换机(性能降级不断连)。

**Q:DPU 要支持完整协议栈?**
不要。声明式合规:最小 = AI Base profile;声明格式如 "Complies to UE Spec v1.1,
AI-BASE, LL-LLR";可选特性必须可关闭。

**Q:PDC 需要预先建立连接吗?**
不需要。首包 syn=1 捎带建立(数据零等待),可能直接复用已有 PDC;
应用全程无感。需要"提前"的只有作业授权上下文(编排器推表)。

**Q:PDC 没有安全?**
安全在 SES 授权(逐包查 JobID+key,常开)和 TSS(可选加密);
PDC 的判重反而是防重放的地基(§3.7.3)。

**Q:作业启动时要配好所有通信对象的密钥?**
不用配"所有对端":每 FEP 一把域钥匙(O(N) 条记录),client-server KDF 模式下
服务器连这一条都不需要(主钥匙现场派生客户端密钥)。运行时新增访问 =
管理面一条 netlink 配置(动态多 JobID + FI_AV_AUTH_KEY)。

**Q:管理谁来做?**
特权实体(作业启动器/调度器)推 JobID 映射;SDME(可内嵌于作业启动服务)
管安全域/密钥/轮换;FEP 驱动+硬件是执行点。规范把管理角色锚定在"谁本来就在管作业"。

---

*整理自对 UE Specification 1.1 全文与 arXiv:2610.12815v2 的研读讨论*
