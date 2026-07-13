# floo_axi_chimney DUT 行为规格（单一事实源）

本文从上游材料机械提炼：`vendor/floonoc/docs/`（chimneys/flits/route_algos/links.md）、
`vendor/floonoc/hw/floo_pkg.sv`、`vendor/floonoc/hw/floo_axi_chimney.sv`（FlooNoC v0.8.4，SHA 见 vendor/VENDOR.md）。
checker/SVA 的期望值只准从本文推导；本文有缺口/歧义时走 CLAUDE.md §8 修改流程，禁止直接照抄 RTL 行为。

## 0. 本项目验证适配表 ★

| # | 适配项 | 约定 |
| --- | --- | --- |
| 1 | 验证对象 | 单实例 `floo_axi_chimney`（非上游背靠背拓扑）；TB 侧扮演 AXI manager（axi_in）、AXI subordinate（axi_out）与 NoC 对端（floo req/rsp 双链路） |
| 2 | 基线配置（M1/M2） | `AxiCfg='{32,64,1,3,3}`（Addr/Data/User/InId/OutId），`ChimneyCfg=ChimneyDefaultCfg`（含 B/R 均 NoRoB），`RouteCfg`=XYRouting+地址偏移（XYAddrOffsetX=16, XYAddrOffsetY=20，UseIdTable=0），AtopSupport=1，MaxAtomicTxns=4——与上游 floo_test_pkg 对齐 |
| 3 | 配置矩阵（M3/M4） | RoB {NoRoB, SimpleRoB, NormalRoB}（B/R 独立）× MaxUniqueIds {1,>1} × ATOP {on,off}；M4 加 SourceRouting 与 IdTable/SAM |
| 4 | 覆盖率口径 | 六类 line+cond+fsm+tgl+branch+assert，≥90% 合格；DUT 范围 = `floo_axi_chimney` 及其子模块实例 |
| 5 | multicast/collective | v0.8.4 的 collective/multicast 特性不在本项目范围（CollectiveCfg 全 0，user_struct_t 缺省） |
| 6 | 上游语义缺口处理 | 本 spec 未覆盖的行为 → 登记 bugs.md → rev 仲裁补 spec，不得由 checker 现场解释 |

## 1. 概述

`floo_axi_chimney` 是 AXI4（含 ATOP）到 FlooNoC 链路层的网络接口（NI）：
把 AXI 五通道事务打包成 flit 注入 NoC 的 req/rsp 两条物理链路，反向解包；
负责 AXI 保序（RoB/stall）、txnID 重映射、路由头生成。routers 对 AXI 语义不感知，保序责任全在 NI。

数据通路（RTL 结构，hw/floo_axi_chimney.sv）：
AX 输入 spill_register（CutAx）→ flit 打包 + 路由计算（floo_id_translation / route_table）→
req/rsp 各一 wormhole arbiter → 出口 spill_register（CutOup）→ 链路；
入链路 spill_register（CutRsp，作用于 floo_req_i/floo_rsp_i）→ 解包 → B/R 各一 `floo_rob_wrapper`（RoBType 选型）→ AXI 响应端口；
出 NoC 方向 `floo_meta_buffer` 做 txnID 重映射与 src_id 存储；退化端口用 `axi_err_slv` 吞非法访问。

## 2. 参数与端口

### 2.1 参数

| 参数 | 类型 | 语义 |
| --- | --- | --- |
| `AxiCfg` | `axi_cfg_t` | AddrWidth / DataWidth / UserWidth / InIdWidth（manager 侧 txnID 宽）/ OutIdWidth（subordinate 侧 txnID 宽） |
| `ChimneyCfg` | `chimney_cfg_t` | 见 §2.2 |
| `RouteCfg` | `route_cfg_t` | RouteAlgo（XYRouting/YXRouting/IdTable/SourceRouting）、UseIdTable、XYAddrOffsetX/Y、IdAddrOffset、NumSamRules、NumRoutes、CollectiveCfg（本项目恒默认） |
| `AtopSupport` | `bit` | 是否支持 AXI atomic（ATOP）；=0 时收到 ATOP 触发断言错误 |
| `MaxAtomicTxns` | `int unsigned` | 在飞 atomic 事务上限，≤ 2**OutIdWidth-1 |
| `id_t/rob_idx_t/route_t/dst_t/hdr_t/sam_rule_t/Sam/...` | type/value | 由 `FLOO_TYPEDEF_*` 宏族生成；源路由时 `dst_t=route_t`，其余 `dst_t=id_t` |
| `axi_in_req/rsp_t`, `axi_out_req/rsp_t` | type | AXI 结构体（`AXI_TYPEDEF_ALL` 生成） |
| `floo_req_t/floo_rsp_t` | type | 链路结构体：`valid/ready` 握手 + flit union（`FLOO_TYPEDEF_AXI_CHAN_ALL`） |

### 2.2 ChimneyCfg 字段（上游 docs/chimneys.md Configuration 表 + floo_pkg.sv `chimney_cfg_t`；CutOup 仅后者有）

| 字段 | 语义 |
| --- | --- |
| `EnMgrPort` | =0 时无 manager 挂接：axi_in 侧退化，非法访问由内部 err_slv 应答（本项目基线 =1） |
| `EnSbrPort` | =0 时无 subordinate 挂接：axi_out 侧退化（基线 =1） |
| `MaxTxns` | NI 可处理的进/出方向 outstanding 事务数上限 |
| `MaxUniqueIds` | 出 NoC 方向可发出的唯一 txnID 数；=1 时全网事务串行化（FIFO 存 src_id），>1 时用 id_queue 支持乱序（§6） |
| `MaxTxnsPerId` | 每 txnID 的 outstanding 上限，仅 NormalRoB 使用 |
| `BRoBType/BRoBSize` | B 响应 RoB 选型/深度（NoRoB 时 Size 无效） |
| `RRoBType/RRoBSize` | R 响应 RoB 选型/深度 |
| `CutAx/CutRsp/CutOup` | 进 AX / 入链路 / 出链路的 spill register 时序切割（不改变协议行为，只加一拍） |

### 2.3 端口

| 端口 | 方向 | 语义 |
| --- | --- | --- |
| `clk_i/rst_ni` | in | 时钟；异步低有效复位 |
| `test_enable_i` | in | DFT，功能验证恒 0 |
| `sram_cfg_i` | in | RoB SRAM 配置透传，功能验证恒 '0 |
| `axi_in_req_i/axi_in_rsp_o` | in/out | AXI slave 口（挂 manager，如 core） |
| `axi_out_req_o/axi_out_rsp_i` | out/in | AXI master 口（挂 subordinate，如 DRAM） |
| `id_i` | in | 本节点 ID/坐标（XY 时为坐标 struct） |
| `route_table_i` | in | 源路由表（`route_t [RouteTblMsb:0]`，仅 SourceRouting 使用） |
| `floo_req_o/floo_req_i` | out/in | req 物理链路（出/入），valid/ready + flit |
| `floo_rsp_o/floo_rsp_i` | out/in | rsp 物理链路（出/入） |

## 3. flit 格式与通道映射

### 3.1 header 字段（docs/flits.md；`FLOO_TYPEDEF_HDR_T`）

| 字段 | 使用者 | 语义 |
| --- | --- | --- |
| `dst_id` | router | 目的 ID；源路由时为整条 route（每跳消费低位并右移） |
| `src_id` | chimney | 源节点 ID，响应侧用它生成回程 dst_id |
| `last` | router | wormhole 锁：=0 时 router 锁方向直到 last=1 的 flit 通过；burst 不被其他端点交织 |
| `axi_ch` | chimney | `axi_ch_e`：AxiAw=0/AxiW=1/AxiAr=2/AxiB=3/AxiR=4 |
| `rob_req` | chimney | 该 flit 的响应需要 RoB 重排 |
| `rob_idx` | chimney | RoB 槽位索引 |
| `atop` | chimney | ATOP flit，旁路重排逻辑 |

flit = `{hdr, payload, rsvd}`；rsvd 为对齐到链路宽度的填充位（`get_axi_rsvd_bits` 计算，各 flit 打满到本链路最宽通道）。

### 3.2 通道映射（floo_pkg::axi_chan_mapping）

- **req 链路**：AW、W、AR 三通道复用，wormhole arbiter 交织仲裁。
- **rsp 链路**：B、R 两通道复用。
- AW 与对应 W 流：AW 先行，其 W beats 跟随于同一 req 链路（SelAw/SelW 状态机保证 AW-W 配对顺序；W burst 靠 last 位维持 wormhole 完整性）。

## 4. 请求路径行为（axi_in → floo_req_o）

1. AW/AR 接受条件：路由可解析 + 保序资源可用（RoB 槽位或 stall 计数许可）+ meta 信息可登记。
2. 打包：AX/W payload 原样进 flit（txnID 原值随 payload 传输，路由不使用）；header 按 §3.1 生成，`src_id=id_i`。
3. 路由计算（docs/chimneys.md Routing / route_algos.md）：
   - 地址偏移法（UseIdTable=0）：XY 时 `dst.x = addr[XYAddrOffsetX +: $bits(id.x)]`，`dst.y = addr[XYAddrOffsetY +: $bits(id.y)]`，`dst.port_id = 0`（floo_id_translation 语义）；其他算法 `dst = addr[IdAddrOffset +: $bits(id)]`。`id_t` 的 x/y 位宽由 TB 配置基线钉住（M1 起唯一定义于 `tb/uvm/env/chimney_tb_cfg.sv`，与 NumX/NumY 对应）。
   - SAM 表法（UseIdTable=1）：地址区间 → 节点 ID（`Sam`/`NumSamRules`，floo_id_translation 实现）。
   - SourceRouting：先按上述得 dst 节点 ID，再以 dst 为索引查 `route_table_i` 得整条 route 放入 dst_id；src_id 仍为节点 ID。
4. 同一 txnID 到**同一目的地**的多事务可背靠背注入（静态路由保证网络内同源同信道同目的地保序）。
5. 到**不同目的地**：NoRoB 下 stall（等待前序完成）；有 RoB 时分配 rob_idx 后注入（§5）。

## 5. 响应路径保序（floo_rsp_i → axi_in_rsp_o）

AXI 规则：同 txnID 的响应必须按请求序返回。NoC 链路层不保序，由 NI 用两种策略实现：

| RoBType | 行为 |
| --- | --- |
| `NoRoB` | 无缓冲。同 txnID 去不同目的地时 stall 注入侧（计数器跟踪 per-txnID outstanding）；响应直通。 |
| `SimpleRoB` | FIFO 式：同 txnID 响应不重排；不同 txnID 事务被串行化。支持多 outstanding，**不支持 burst**（主要用于单拍 B 响应）。 |
| `NormalRoB` | 完整重排：乱序到达的响应缓存于 RoB，按序释放；保留不同 txnID 间的乱序性；支持 burst 与 MaxTxnsPerId 深度控制。 |

- B 与 R 通道 RoB 独立选型（BRoBType/RRoBType）。
- `atop=1` 的响应旁路 RoB。
- RoB 空间必须在注入请求前预留——响应不可反压 NoC（否则死锁），NI 只在响应可被吞下时才注入请求。

## 6. txnID 重映射与 ATOP（axi_out 方向）

1. 出 NoC 的请求（NoC→subordinate）txnID 重映射（floo_meta_buffer）：
   - `MaxUniqueIds=1`（默认）：全部 remap 为 `'1`（全 1），src_id 存 FIFO，请求/响应同序 → 下游被串行化。
   - `MaxUniqueIds>1`：以 id_queue 分配多个 txnID，允许下游乱序返回，代价更高。
2. 响应返程：用存储的 src_id 生成回程 header（dst_id=原 src_id）。
3. ATOP：以 `[0, '1)` 范围内**全网在飞唯一**的 txnID 发出（数量 ≤ MaxAtomicTxns）；ATOP 带响应（B/R）按 AXI5 ATOP 语义回流。AtopSupport=0 时不支持。

## 7. 链路层握手

- 每条物理链路（req/rsp 各向）为 `valid/ready` 握手 + flit 数据；valid 置起后 flit 保持稳定直到握手完成（AXI 同款语义）。
- wormhole：last=0 的多 flit 包（W burst）在 req 链路上不被其他通道交织；单 flit 包 last=1。

## 8. 复位与退化行为

- `rst_ni` 异步置位、同步释放（common_cells registers.svh 惯例）；复位后所有 valid 输出为 0。
- `EnMgrPort=0`：axi_in 请求由内部 err_slv 以错误响应应答（不进 NoC）；`EnSbrPort=0`：来自 NoC 的请求由 err_slv 应答 SLVERR。
- 断言（RTL 内建，`common_cells/assertions.svh`）：非法 axi_ch 编码、ATOP 越界等触发仿真 fatal/error——TB 不得触发这些作为"正常刺激"。
- 配置合法性约束：`EnMgrPort=0` 时 B/R 的 RoBType 必须为 NoRoB（RTL 断言强制）；M3/M4 配置矩阵须排除该非法组合。

## 修改记录

| # | 日期 | 版本 | 章节 | 摘要 | 依据 |
| --- | --- | --- | --- | --- | --- |
| 1 | 2026-07-13 | 0.0.0 | 全文 | v0 初稿：从上游 docs+RTL 机械提炼 | vendor/floonoc/docs/*、hw/floo_pkg.sv、hw/floo_axi_chimney.sv @ v0.8.4 |
| 2 | 2026-07-13 | 0.0.0 | §1/§2.2/§4.3/§8 | REV-001 整改：CutRsp/CutOup 通路更正；XY dst 提取公式精确化；§2.2 依据补注；补 EnMgrPort=0⟹NoRoB 约束 | doc/review/REV-001.md（P-1..P-4） |
