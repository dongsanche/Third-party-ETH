# 长距离跨数据中心 RDMA 调研:问题、重传与 QoS 方案、厂商公开材料

> 调研日期:2026-11-25
> 范围:跨数据中心 / 跨域 / 广域链路上的 RDMA(以 RoCEv2 为主),聚焦可靠性和 QoS
> 证据等级:【全文】读过原文(部分由子任务通读后摘要);【摘要】只读了摘要;【二手】经其他论文 related work 或第三方描述确认;【搜索摘要】只到搜索引擎结果摘要,原文未打开;【背景知识】未在本次核对;【粗估】本文估算

---

## 零、结论先行

1. RDMA 的整条设计链路(PFC 无损、ECN 反馈、Go-Back-N、µs 级 RTT)都默认"数据中心内",跨 DCI 后逐环失效,问题是一整串连锁反应。
2. 公开材料里,成熟且已上生产的是"单集群十万卡级"方案(微秒级时延),路线已经收敛:关 PFC 的有损以太网 + SACK 选择性重传 + 乱序落位 + 包裁剪;包喷射 + ECN 自适应负载均衡;多平面 Clos + SRv6;Swift 与 DCQCN 融合;应用层信用与发送窗口。
3. 真正的跨域长距(几十到上千公里),头部厂商只公开了方向性发布和试点(NVIDIA Spectrum-XGS、国内运营商 OTN 试点),没有带细节的成熟方案;厂商与高校合作的研究(SDR-RDMA、LCMP、COMET 等)仍在进行。

---

## 一、背景

- **驱动因素**:单个数据中心受电力和土地约束,预训练走向多集群 / 多数据中心(SDR-RDMA 引用 OpenAI 的 GPT-4.5 训练);此外还有跨集群 PD 分离、联邦学习、跨域存储。
- **链路形态**:园区内(µs 级时延)、同城 / 区域(几百 km,ms 级 RTT)、跨洲(数千 km,几十 ms)。SDR-RDMA 的案例是 3750 km ≈ 25 ms RTT;LCMP 仿真按 200 / 1000 / 2000 km 对应 1 / 5 / 10 ms 单程传播时延。

---

## 二、核心矛盾

| 数据中心内的假设 | 跨数据中心后的现实 |
|---|---|
| PFC 保证无损 | PFC 跨广域不可行(缓冲、稳定性),只能有损运行 |
| ECN 反馈 µs 级到达 | ECN 被传播时延拖慢,控制环路 ms 到 100 ms 级 |
| 丢包用 Go-Back-N 回退,窗口小 | 窗口等于 BDP,丢 1 个包回退整个窗口 |
| 报文有序 | 多路径、TE 切换导致乱序,被 RNIC 当成丢包 |
| 消息远大于 BDP,重传可被流水线掩盖 | 消息大小接近 BDP,每次重传直接暴露 1 个 RTT |

---

## 三、主要问题

### 问题 1:可靠性机制失效(最核心)

- RoCEv2 RC 默认 Go-Back-N,丢 1 个包就从该 PSN 起重传整个窗口【全文,FedRDMA、OptiNIC】。
- 【粗估】400G × 25ms RTT 的 BDP ≈ 1.25 GB ≈ 30 万个 4KB 包;丢包率 1e-6 / 1e-5 / 1e-4 时效率只剩约 86% / 31% / 3%(模型见附录 A)。
- 丢包 → 回退重传 → 链路上数据更多 → 更多丢包,恶性循环直到传输失败【全文,FedRDMA;Mininet 仿真 20 ms RTT、关闭 PFC/ECN 下,≥4 Gbps 且块 ≥2 MB 时基本传不动,丢包接近 100%】。
- 不可靠的 UC 也顶不上:多包消息丢 1 个包整条作废(1 GiB Write 丢一个 4KB 包就要重发 1 GiB);运营商网络乱序会让 UC QP 因 PSN 不匹配整条丢弃【全文,SDR-RDMA §2.3】。

### 问题 2:没有普适的替代可靠性方案

- 选择性重传(ConnectX-7/8 已支持)每次丢包至少赔 1 个 RTT,超时触发则是多个 RTT【全文,SDR-RDMA】。
- 400G / 25ms / 丢包 1e-5,消息大小接近 1/P 且不超过 BDP 时,SR 比无损慢 6.5×(均值)、12.2×(p99.9);NACK 把丢包检测缩短到 1 个 RTT,最多改善 4×,但改不掉"每丢一个 chunk 赔一个 RTT"【全文,SDR-RDMA §5.2】。
- 纠删码不等 RTT,但吃带宽和算力:MDS(32,8) 开销 20%;丢包率 >0.1% 时恢复不了;消息 >11 GiB 或丢包率 <1e-6 时反而不如 SR【全文,SDR-RDMA §5.2】。
- 集合通信放大代价:ring allreduce 有 2N−2 个相互依赖的阶段,单阶段损失逐级累加;EC 相对 SR 的加速随丢包率从 3× 涨到 6× 以上【全文,SDR-RDMA §5.3】。

### 问题 3:丢包率既高又不可预测

- 实测一条 350 km 运营商光链路(Lugano–Lausanne,100G):同一包长的丢包率跨 3 天波动 3 个数量级,1 KiB 包 1e-4~1e-2,8 KiB 包 1e-3~>1e-1,疑似运营商侧交换机缓冲拥塞【全文,SDR-RDMA §2.1】。
- 私有光网在以太网 FEC 之前的丢包率可低至 1e-8(微软 15 个数据中心的数据)【全文,SDR-RDMA 引 Zhuo et al. 2017】。
- 结论:可靠性策略只能按链路甚至按连接定制。

### 问题 4:拥塞控制和拥塞信号失灵

- DCQCN 按 µs RTT 设计;长距下 ECN 被传播时延拖慢,瞬时队列分不清突发和持续拥塞【全文,LCMP】。
- 控制器闭环是 100 ms 级(RedTE),追不上 µs 级微突发;ms 级流量工程挡不住 PFC 风暴【全文,LCMP】。
- 公共 HP-WAN 上关键数据面参数不可控,RoCEv2 直接不可用;TCP 也受微突发丢包困扰,要 BBRv1 加入口整形才可预测【摘要,arXiv:2603.12660】。

### 问题 5:多路径带来乱序、路径不对称和羊群效应

- 跨 DC 多路径在时延和容量上高度不对称:UCMP 把流量压到高容量高时延路径(某链路利用率 17%,ECMP 下为 6%),低时延路径闲置【全文,LCMP】。
- 新流同时到达时都选"最便宜"路径,造成碰撞【全文,LCMP】。
- 包喷射、TE 切换、flowlet 引流都会乱序;很多 RNIC 把乱序当丢包,GBN 再把它变成吞吐崩溃【全文,LCMP;COMET】。

### 问题 6:NIC 状态随 BDP 膨胀

- 每连接状态(RoCE 407 B、IRN 596 B、Falcon 350 B)在 4 MB SRAM 下只能开 8K–20K 个 QP【全文,OptiNIC】。
- 乱序传输加纠删码绕开重传,需要 NIC 逐包记录到达 bitmap,bitmap 随 BDP 线性增长:Alveo U280 约 9 MiB BRAM 在广域 RTT 加 Tbps 下不够用【全文,COMET】。

### 问题 7:光层 / OTN 速率不匹配

- 源宿两侧 OTN 速率不协调时,目的侧 OTN 缓存积压;MatchRDMA 做速率匹配后吞吐最高提升 20×,目的侧缓存占用降低 62.7%【摘要,arXiv:2604.23932】。

### 问题 8:上层干脆绕开 RDMA

- 跨 DC PD 分离的 KV Cache 传输直接走 TCP【摘要,PDD、Prefill-as-a-Service】。
- 联邦学习退回 4MB 块的停等式发送【全文,FedRDMA】。
- GeoPipe 靠流水线并行加计算通信重叠,把气泡最多压低 78.91%【摘要,arXiv:2510.12064】。

### 问题 9:落地慢

- 可靠性逻辑固化在 NIC ASIC 里,一轮迭代 3–4 年,新算法要等下一代芯片【全文,SDR-RDMA §2.2】。

---

## 四、重传方案

| 类别 | 方案 | 做法 | 长距下的边界 | 证据 |
|---|---|---|---|---|
| 回退重传 | Go-Back-N(RoCEv2 RC 默认) | 丢 1 个包就从该处重传整个窗口 | 高 BDP 下崩溃,只适合低丢包专纤 | 全文 |
| 协议扩展 | IB 规范的选择性确认 NAK | NAK 附带 7 bit 掩码,标记后续哪些包已收到 | 掩码只有 7 个包,长距窗口动辄上万包 | 背景知识 |
| 选择性重传 | IRN | 接收端缓存乱序包,NAK 带 bitmask 只补丢的包,不依赖 PFC | 每次丢包 ≥1 RTT;每 QP 状态 596 B | 背景知识 + 二手 |
| | SRNIC | 把乱序处理和重传卸载到主机软件线程 | 同上;每 QP 242 B | 二手 |
| | Flor | 在 ConnectX-4/5 上基于 UC 实现选择性重传 | 不支持卸载 | 二手(SDR-RDMA) |
| | ConnectX-7/8 原生 SR | ASIC 内实现 | 比无损慢 6.5×(均值)/ 12.2×(p99.9) | 全文(SDR-RDMA) |
| | Falcon(Google) | 网卡内实现的多路径 RoCEv2 替代;硬件快速重传 + 时延拥塞控制 | 每 QP 350 B,状态多、故障面大 | 二手(MRC、OptiNIC) |
| | MRC(OpenAI + 微软) | 关 PFC、有损以太网;SACK 快速选择性重传;乱序落位;包裁剪触发 NACK | 单集群微秒级时延场景 | 全文 |
| 应用层分块 | FedRDMA | 4MB 块,收到 ACK 再发下一块 | 稳定但停等,牺牲吞吐 | 全文 |
| 前向纠错 | LoWAR、Cloudburst | 面向长距链路的纠删码;编码包分散到多路径提前恢复 | LoWAR 只做到 10G FPGA 原型 | 二手(SDR-RDMA) |
| | SDR-RDMA EC 层、COMET | 随数据一起发校验块,接收端就地恢复 | MDS(32,8) 带宽开销 20%;丢包率 >0.1% 失效;需 NIC 逐包跟踪 | 全文 |
| | OptiNIC | NIC 不重传、不保序,交给 ML 流水线用纠删码兜底 | 只适合容忍丢失的训练 / 推理负载 | 全文 |
| 混合 | SDR-RDMA | 默认 EC,解不出的子消息退回 SR 重传 | 需要可编程 NIC(DPA)做卸载 | 全文 |
| 软件定义 | SDR-RDMA | 把到达 bitmap 暴露给上层,按连接选用 SR 或 EC | 小于 512 KiB 的消息比 RC 慢 | 全文 |

**适用区间与边界**

- SR 与 EC(SDR-RDMA,400G / 25ms):128 KiB–11 GiB 的消息、丢包率 1e-6~1e-2 时 EC 更好;消息 >11 GiB 或丢包率 <1e-6 时 SR 更好;小消息两者接近,SR 更省算力;丢包率 >0.1% 时 EC 失效。
- 编码开销:400G 下 XOR 编码 4 个 CPU 核可以隐藏,MDS 需要两倍;XOR 在丢包率约 1e-3 就要回退 SR,MDS 可以撑到 1e-2 以上。
- SDR-RDMA 卸载性能:400G 线速(512 KiB 消息只需 256 个 DPA 线程中的 20 个);128 线程接近 3.2 Tbit/s。
- COMET:400G 线速(8 路时超过 1.6 Tb/s);0.5 ms 抖动下并发连接数是 SDR-RDMA 的 6 倍、传统 bitmap 的 57 倍。

---

## 五、QoS 方案

| 层面 | 方案 | 做法 | 长距适配 | 证据 |
|---|---|---|---|---|
| 拥塞控制 | DCQCN | 依赖交换机 ECN 标记 | 反馈被传播时延拖慢;公网不可配 | 全文 |
| | 时延类:Timely、Swift | 用 RTT 上升判断拥塞,不需要交换机配置 | — | 背景知识 |
| | NVIDIA RTTCC | 扩展 DCQCN,硬件反馈环路,周期发时间戳包测 RTT | 无需交换机配置;达到常规 RoCE 的 99%(≤256 连接)和 98%(>16000 连接),但测试是数据中心内 incast | 全文(NVIDIA 博客) |
| | Swift + DCQCN 融合 | RTT 与 ECN 一起用 | 字节 MegaScale 生产在用 | 全文 |
| | 跨 DC 专用:时延 CC(Geng 2023)、LSCC(IWQoS 2024) | LSCC 按链路分段做拥塞控制 | 针对大 RTT、大 BDP 设计,用更细粒度信号 | 二手(LCMP) |
| | UEC NSCC(SMaRTT) | 时延 + ECN + 包裁剪融合,支持多路径 | 相比 Swift、RoCE、MPRDMA 最多好 50% | 摘要 |
| | NVIDIA Spectrum-X | 交换机遥测 + 网卡侧限速 | 比通用以太网高 1.6×,10 万 GPU 以上 95% 效率 | 全文(NVIDIA 官网) |
| 无损保障 | PFC 直通 | 跨长距跑 PFC | 缓冲要按 BDP 配,LCMP 仿真里用了 6 GB | 全文(LCMP) |
| | SWING(2022) | PFC 中继,把无损延伸到长距 | 仍依赖逐跳流控 | 二手(LCMP) |
| | Bifrost(ToN 2024) | 下游驱动的无损流控 | 缓冲预留低、零丢包 | 二手(LCMP) |
| | GeoPipe | 无损 RDMA 的 DC-OTN | 靠专用光层管道 | 摘要 |
| 整形 / 速率匹配 | 出口整形(arXiv:2603.12660) | 入 HP-WAN 前整形 | 抑制微突发丢包,完成时间可预测 | 摘要 |
| | MatchRDMA | 源宿 OTN 速率匹配 + 分段发送 | 吞吐最高 20×,目的 OTN 缓存占用 −62.7% | 摘要 |
| 路由 / 流量工程 | LCMP(EuroSys 2026) | 路径质量分(时延 + 容量)加交换机内拥塞信号,过滤高代价候选后哈希并钉住路径 | 中位 FCT slowdown −36%~−76%,P99 −45%~−77%;测试基于 SoftRoCE | 全文 |
| | RedTE(SIGCOMM 2024) | 多智能体 RL 调整边缘路由器分流比 | 100 ms 环路追不上 µs 级微突发,对 RDMA 退化成静态哈希 | 二手(LCMP) |
| | POP、Teal | WAN 流量工程 | ms 级,对 TCP 有效、对 RDMA 不够 | 二手(LCMP) |
| 多路径 | 包喷射:REPS、Ethereal | 逐包或按流分散到多路径 | 必须搭配乱序传输,否则 RNIC 把乱序当丢包 | 摘要 |
| | MRC + SRv6 | 每 QP 128–256 条路径逐包喷射;ECN 当路径健康信号;静态源路由绕开故障 | 单集群生产 | 全文 |

---

## 六、互联网厂商公开材料

### 6.1 已上生产的共识(单集群内)

| 系统 | 机构 | 状态 | 重传 | QoS |
|---|---|---|---|---|
| MRC(Multipath RC) | OpenAI + 微软(AMD、Broadcom 参与作者) | 生产:两家最大训练集群,用于训练最新前沿模型 | 关 PFC,有损以太网;SACK 快速选择性重传;乱序落位;包裁剪触发 NACK | 每 QP 128–256 条路径逐包喷射;ECN 做自适应负载均衡(末跳关 ECN);多平面 Clos(两层拓扑支撑 10 万 GPU 以上);SRv6 静态源路由 |
| ByteExpress | 字节跳动 + 复旦 | 生产:PS 架构的推荐推理和离线训练 | RC over RoCE;DCQCN + PFC | 接收信用流控;每引擎发送窗口限制在途字节 |
| MegaScale | 字节跳动 | 生产,万卡以上 | 调 NCCL 重传定时器和重试次数;开网卡 adap_retrans | Swift 与 DCQCN 融合;2×200G 拆分和多轨降低 ECMP 冲突 |
| Spectrum-X | NVIDIA | 产品 | — | 交换机遥测 + 网卡侧限速,无损 |
| EFA / SRD | AWS | 产品 | 文档页未说明 | 自带拥塞控制;流量不能跨可用区或 VPC |
| Falcon | Google | 未核实 | 快速重传 | 时延类拥塞控制、多路径 |

**要点**

- **MRC**(arXiv:2605.04333):对 RoCE RC 的最小扩展,传输层只支持 WRITE 和 WRITE_WITH_IMM;单程时延 5.09 µs(T0 内)、6.54 µs(跨 T1)。关 PFC 的理由:喷射后一条流经几百条路径到达末跳,与 PFC 无法配合;PFC 还会造成集合通信间的队头阻塞。
- **ByteExpress**(arXiv:2610.12815):论文原话是只靠硬件拥塞控制挡不住持续的 PFC 暂停,所以加了软件发送窗口;训练消息均值 / P99 为 20 KB / 50 KB,P99 推送时延比 TCP 低 74.0%,观测期内无 PFC 暂停。详见同目录 `byteexpress-vs-uec.md`。
- **MegaScale**(arXiv:2402.15627):175B 模型在 12,288 卡上 MFU 55.2%。
- **Spectrum-X**:比通用以太网高 1.6×,10 万 GPU 以上 95% 效率。
- 三篇论文(MRC、ByteExpress、MegaScale)都没有跨数据中心的表述,时延都是微秒级,属于单集群(可跨机柜、多平面)方案。

### 6.2 跨域长距的现状

- **NVIDIA Spectrum-XGS**【全文,NVIDIA 官网】:官网称 scale-across,把多个数据中心连成单一 AI 超级工厂,覆盖机柜、楼宇、园区;多站点 NCCL 性能是通用站点间以太网的 1.9 倍。重传和拥塞控制机制没有公开,只看到发布,没看到部署。
- **Google Falcon**:只在 MRC 论文和 OptiNIC 里有二手描述,没查到跨域表述。
- **国内运营商 + 华为**【搜索摘要】:以试点和现网验证为主。
  - 华为官网《RDMA 拉远无损–光传送小百科》(2026-08):拉远后在途数据超出缓存,宿端拥塞丢包,需靠无损传输(PFC 等)。
  - 上海电信 + 华为(2024-10):超 120 km 广域 RDMA 无损传输试点,有效吞吐率 90% 以上,拉远训练算力效率损失小于 5%。
  - 中国联通(2026-10,自媒体):3000 km 长距 RDMA 无损传输"现网验证",端口带宽利用率从 20% 提升到 90% 以上。
  - 上海联通:"OTN+RDMA 长距无损技术"。
- **百度**【搜索摘要】:专有云 VPC 3.0 承载跨区 RDMA 存储通信。
- **阿里、腾讯、字节**【搜索摘要】:公开的都是机房内方案:阿里 Solar(SIGCOMM '22,计算到存储网络,加速 EBS)、腾讯云 HCC(RoCEv2)、火山引擎 vRDMA(VPC 内,邀测)。"跨 AZ RDMA"只见于非官方博文,未经证实。
- **SDR-RDMA**(ETH + NVIDIA + 微软作者,2025):还在研究按链路在 SR 和纠删码间选择,说明跨域可靠性在生产上没有定论。
- **UEC**:官网没有任何长距 / 跨数据中心的表述【全文,ultraethernet.org】。

### 6.3 本次没查到的

- AWS UltraCluster:如何连接多站点、用什么传输和 QoS。
- Meta 多数据中心集群:论文 "RDMA over Ethernet for Distributed AI Training at Meta" 没拿到原文。
- 微软跨 DC RDMA 的专门公开材料。

---

## 七、选型建议(本文判断,未经实测验证)

- **私有专纤 / OTN,丢包率 <1e-6**:端到端 ECN 或 PFC 加大缓冲即可,重传用原生 SR;重点做速率匹配和整形。
- **私有但丢包率 1e-6~1e-2**:SR 加 NACK,中等大小消息(128 KiB–11 GiB)改用纠删码,出口做速率匹配和整形。
- **不可配置的公共链路**:时延类拥塞控制或出口整形,可靠性靠纠删码加 SR 回退,或者退回 TCP。
- **多路径 DCI**:包喷射必须配乱序传输;否则钉住路径(flow pinning)。
- **设计基线**:可参考 MRC / UET 路线(有损 + SACK + 乱序 + 包裁剪),再叠加纠删码应对高丢包段。

---

## 八、尚未解决的点

- 真实长距链路上的实测数据稀缺,多数结论来自 Mininet、NS-3 或 Python 模型,或 SoftRoCE 测试床。
- 缺少按链路动态选择可靠性策略的控制平面和标准接口。
- 乱序传输与 NIC 状态可扩展性之间的矛盾,目前只有 COMET 的缓存化 bitmap 等早期探索。
- 时延下界受光速约束;WAN 感知的集合通信算法、ZR+ 光模块、空芯光纤是互补方向(arXiv:2606.30278)。
- UEC 的定位里没有长距表述,跨域标准化是空白。

---

## 九、未核实、不要直接引用的条目

- NVIDIA "Adaptive Retransmission":官方技术博客两次搜索都没找到。
- SWING、Bifrost、LSCC、Geng 2023:IEEE 论文,arXiv 上没有,只拿到 LCMP 的一句描述,具体机制和数字需要查原文。
- AWS UltraCluster、Meta 多 DC、Google Falcon 细节:抓取失败。
- 国内厂商条目:只到百度搜索结果摘要级,数字需要复核。
- MatchRDMA、GeoPipe:只看了摘要。
- 背景知识项:IB 规范 7 bit 掩码、IRN / SRNIC 机制、Timely / Swift / HPCC、AWS SRD 特性。

---

## 附录 A:Go-Back-N 效率粗估

**假设**:窗口始终填满、丢包独立、不触发超时、接收端丢弃乱序包。一个窗口内前 j 个包成功、第 j+1 个丢失时,窗口前进 j 个包;全部成功时前进 W 个包。

**结果**:期望效率 η = E[前进] / W = (1 − e^(−pW)) / (pW),只取决于 pW。

| 丢包率 p(W = 30 万) | pW | η(本文模型) | 常见近似 (1−p)/(1+(W−1)p) |
|---|---|---|---|
| 1e-6 | 0.3 | 86% | 77% |
| 1e-5 | 3 | 31% | 25% |
| 1e-4 | 30 | 3% | 3% |

两种算法量级一致,仅用于量级判断。

---

## 附录 B:来源清单

| 名称 | 标识 | 年份 / 出处 | 证据等级 | 本文用途 |
|---|---|---|---|---|
| SDR-RDMA: Software-Defined Reliability Architecture for Planetary Scale RDMA Communication | arXiv:2505.05366 | 2025 | 全文 | 可靠性分析、丢包实测、SR/EC 适用区间 |
| COMET: Scalable Packet Tracking on FPGAs for Erasure-Coded RDMA over Lossy WANs | arXiv:2609.21774 | FPL 2026 | 全文 | 多路径乱序、bitmap 与 BDP |
| LCMP: Distributed Long-Haul Cost-Aware Multi-Path Routing for Inter-Datacenter RDMA Networks | arXiv:2604.07836 | EuroSys 2026 | 全文 | 路径不对称、拥塞信号延迟、相关 work |
| MatchRDMA: A Segmented and Rate-Matched Long-Haul RDMA Scheme for Geo-distributed LLM Training over OTN | arXiv:2604.23932 | 2026 | 摘要 | OTN 速率匹配 |
| FedRDMA: Communication-Efficient Cross-Silo Federated LLM via Chunked RDMA Transmission | arXiv:2403.00881 | 2024 | 全文 | WAN 上 GBN 恶性循环、分块 |
| OptiNIC: A Resilient and Tail-Optimal RDMA NIC for Distributed ML Workloads | arXiv:2512.22743 | 2025 | 全文 | 可靠传输的每 QP 状态、尾时延 |
| Evaluation of TCP Congestion Control for Public High-Performance Wide-Area Networks | arXiv:2603.12660 | HPSR 2026 | 摘要 | 公共 HP-WAN、整形 |
| GeoPipe: a Geo-distributed LLM Training Framework with enhanced Pipeline Parallelism in a Lossless RDMA-enabled Datacenter Optical Transport Network | arXiv:2510.12064 | ACP 2025 | 摘要 | 无损 DC-OTN |
| PDD: Unleashing Economical and Flexible Heterogeneous LLM Inference via Cross-Datacenter Prefill-Decode Disaggregation | arXiv:2609.13161 | 2026 | 摘要 | 跨 DC PD 分离走 TCP |
| Prefill-as-a-Service: KVCache of Next-Generation Models Could Go Cross-Datacenter | arXiv:2604.15039 | 2026 | 摘要 | 同上 |
| LLMs and Optical Networks: A Symbiotic Relationship | arXiv:2606.30278 | 2026 | 摘要 | 光层使能技术 |
| Ultra Ethernet's Design Principles and Architectural Innovations | arXiv:2508.08906 | 2025 | 摘要 | UET 概览 |
| SMaRTT: Sender-based Marked Rapidly-adapting Trimmed & Timed Transport | arXiv:2404.01630 | 2024 | 摘要 | UEC NSCC(时延 + ECN + 包裁剪) |
| REPS: Recycled Entropy Packet Spraying | arXiv:2407.21625 | EuroSys 2026 | 摘要 | 包喷射负载均衡 |
| Ethereal: Divide and Conquer Network Load Balancing | arXiv:2407.00550 | 2024 | 摘要 | 单路径分流 |
| Resilient AI Supercomputer Networking using MRC and SRv6 | arXiv:2605.04333 | 2026 | 全文 | MRC 生产方案 |
| Deploying RDMA for Distributed Inference and Training at Scale(ByteExpress) | arXiv:2610.12815 | 2026 | 全文 | 字节生产方案 |
| MegaScale: Scaling Large Language Model Training to More Than 10,000 GPUs | arXiv:2402.15627 | 2024 | 全文 | 字节融合拥塞控制、adap_retrans |
| NVIDIA networking 官网 | https://www.nvidia.com/en-us/networking/ | — | 全文 | Spectrum-X、Spectrum-XGS |
| NVIDIA RTTCC 博客 | https://developer.nvidia.com/blog/scaling-zero-touch-roce-technology-with-round-trip-time-congestion-control/ | 2021 | 全文 | RTTCC |
| AWS EFA 文档 | https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html | — | 全文 | SRD、跨 AZ 限制 |
| UEC 官网 | https://ultraethernet.org/ | — | 全文 | 长距表述缺失 |

**调研方法说明**

- arXiv 可正常抓取;通用搜索引擎大多超时,百度可用但只到摘要级;github.com 超时。
- MRC、ByteExpress 两篇的细节取自本机 opencode 缓存里之前抓取的全文,本次实时抓取未成功。
- PDF 经 webfetch 会损坏,MatchRDMA 只能读到摘要。
