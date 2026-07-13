# REV-001：spec.md v0 审查 + vendor 补丁 P-001 复核

- 审查员：rev
- 日期：2026-07-13
- 卡型：spec 审查 + vendor 补丁复核（两项独立结论）
- 核对依据：`vendor/floonoc/docs/{chimneys,flits,route_algos,links}.md`、`vendor/floonoc/hw/floo_pkg.sv`、`vendor/floonoc/hw/floo_axi_chimney.sv`、`vendor/floonoc/hw/floo_id_translation.sv`、`vendor/floonoc/hw/test/floo_test_pkg.sv`（FlooNoC v0.8.4）

## 一、对象清单

| # | 对象 | 范围 |
| --- | --- | --- |
| 1 | `doc/spec.md` v0 初稿 | 全文（§0–§8）逐条与上游材料核对 |
| 2 | `vendor/VENDOR.md` P-001 及 `vendor/floonoc/hw/floo_axi_chimney.sv` 对应改动 | 行为等价性 + 登记准确性 |

---

## 二、审查对象 1：doc/spec.md — 逐项结论

### 通过项（可指回上游）

| spec 章节 | 结论 | 依据 |
| --- | --- | --- |
| §0 适配表 item2 基线 | 通过 | 与 `floo_test_pkg.sv:28-70` 对齐：`AxiCfg='{32,64,1,3,3}`、`AtopSupport=1`、`MaxAtomicTxns=4`、`XYAddrOffsetX/Y=16/20`、`UseIdTable=0`、`ChimneyCfg=ChimneyDefaultCfg`（`floo_pkg.sv:342-355`，B/R 均 `NoRoB`） |
| §2.1 参数表 | 通过 | 与 `floo_axi_chimney.sv:15-102` 模块头一致；`route_algo_e` 四值 {XYRouting/YXRouting/IdTable/SourceRouting} 见 `floo_pkg.sv:13-42`；`MaxAtomicTxns ≤ 2**OutIdWidth-1` 与断言 `floo_axi_chimney.sv:872` 一致 |
| §2.2 ChimneyCfg 字段 | 通过（含一处引用瑕疵，见 P-3） | 字段与 struct `floo_pkg.sv:279-316` 逐项对应；`CutRsp=入链路` 正确（见下 P-1 反证） |
| §2.3 端口表 | 通过 | 与 `floo_axi_chimney.sv:83-102` 一致；`route_table_i` 宽度 `route_t [RouteTblMsb:0]` 与 P-001 补丁后 RTL 一致 |
| §3.1 header/axi_ch 编码/rsvd | 通过 | `axi_ch_e` 编码 AxiAw=0…AxiR=4 见 `floo_pkg.sv:80-87`；字段语义与 `docs/flits.md`；`rsvd` 打满逻辑与 `get_axi_rsvd_bits`（`floo_pkg.sv:495-498`）一致 |
| §3.2 通道映射 | 通过 | `axi_chan_mapping`（`floo_pkg.sv:375-381`）：AW/W/AR→req，B/R→rsp；与 `docs/links.md` 单 AXI 映射表一致 |
| §5 响应保序（RoB 三型） | 通过 | 与 `docs/chimneys.md` RoBType 表一致；`atop=1` 旁路 RoB 见 `floo_axi_chimney.sv:867-868`（`is_atop_*_rsp`）；RoB 先预留后注入语义与 docs Reordering 段一致 |
| §6 txnID 重映射/ATOP | 通过 | `MaxUniqueIds=1` remap→`'1`、ATOP 区间 `[0,'1)` 与 `docs/chimneys.md` "AXI Transaction ID handling" 段一致；上限断言 `floo_axi_chimney.sv:872` |
| §7 链路层握手/wormhole | 通过 | 与 `docs/links.md`（valid/ready、`last` 位 wormhole）一致 |
| §8 退化行为（err_slv 拓扑） | 通过 | `EnMgrPort=0`→err_slv 挂 `axi_in`（`floo_axi_chimney.sv:268-280`）；`EnSbrPort=0`→err_slv 吞 NoC 请求（`floo_axi_chimney.sv:848-860`，slv_req=`meta_buf_req_in`）——spec §8 两条描述与 RTL 一致 |

### 问题项

**P-1（问题·中）§1 数据通路把 `CutRsp` 误标为出口寄存 —— 与上游冲突且与 §2.2 自相矛盾**
- spec 位置：`doc/spec.md:27` "req/rsp 各一 wormhole arbiter（CutOup/CutRsp 出口寄存）→ 链路"。
- 依据：`CutRsp` 缓冲的是**入链路**——`floo_axi_chimney.sv:289` `gen_rsp_cuts` 对 `floo_req_i`/`floo_rsp_i`（输入侧）插 `spill_register`；`docs/chimneys.md` Configuration 表亦述 `CutRsp="buffer incoming links"`。真正的出口寄存是 `CutOup`（`floo_axi_chimney.sv:691,722` `.Bypass(!ChimneyCfg.CutOup)`）。
- 影响：§1 与 §2.2（正确写"入链路"）互相矛盾；checker/覆盖点若据 §1 会把输入 spill 误当输出。

**P-2（问题·中）§4.3 XY 路由 dst_id 提取表述有歧义，checker 写不出唯一期望值**
- spec 位置：`doc/spec.md:99` "dst = 请求地址在 `XYAddrOffsetX/Y`（XY）处的位段"。
- 依据：`floo_id_translation.sv:75-77`——`id_o.x = addr_i[XYAddrOffsetX +: $bits(id_o.x)]`、`id_o.y = addr_i[XYAddrOffsetY +: $bits(id_o.y)]`、`id_o.port_id = '0`。提取**字段宽度**由 `id_t` 坐标子字段位宽决定，且 `port_id` 恒 0；spec 只说"位段"未给宽度，也未说 `port_id=0`。
- 相关缺口：§0 未 pin `id_t` 结构 / `NumX·NumY`（`floo_test_pkg.sv:19-20` NumX=NumY=4），故 checker 无从确定 `+:` 宽度。
- 影响：路由/dst_id checker 无法机械推出唯一期望 dst_id。

**P-3（问题·低）§2.2 表头出处引用不精确**
- spec 位置：`doc/spec.md:45` "（上游 docs/chimneys.md Configuration 表）"。
- 依据：`docs/chimneys.md` Configuration 表只列到 `CutRsp`，**不含 `CutOup`**；`CutOup` 出自 RTL struct（`floo_pkg.sv:313-315`）。字段内容正确，仅出处标注不全。

**P-4（查漏·低）§8 缺配置合法性约束 `NoMgrPortRobType`**
- 依据：`floo_axi_chimney.sv:875-876` 断言 `EnMgrPort=0 ⟹ B/R RoBType 均须 NoRoB`。
- 影响：M3/M4 若在配置矩阵（§0 item3）中令 `EnMgrPort=0` 同时选非 NoRoB，会触发该断言；spec 未登记此约束。基线 `EnMgrPort=1`，故当前不阻塞。

### §0 自洽性（清单 ④）
- §0 内部自洽；item2 基线与 `floo_test_pkg.sv` 对齐；item5（CollectiveCfg 全 0 / user_struct_t 缺省）与 `CollectiveDefaultCfg`（`floo_pkg.sv:335-339`，全 disabled）一致。
- 唯一缺口：坐标/节点 ID 结构未 pin（并入 P-2 处理）。

---

## 三、审查对象 2：vendor 补丁 P-001 复核 — 逐项结论

**行为等价性：通过。**
- `localparam int unsigned RouteTblMsb = floo_iomsb(RouteCfg.NumRoutes)`（`floo_axi_chimney.sv:81`），端口 `input route_t [RouteTblMsb:0] route_table_i`（`floo_axi_chimney.sv:96`）。
- `[RouteTblMsb:0]` 与原始 `[floo_iomsb(RouteCfg.NumRoutes):0]` 逐位等宽，无语义变化；`floo_iomsb` 定义 `floo_pkg.sv:370-372`，经模块顶 `import floo_pkg::*`（`floo_axi_chimney.sv:16`）可见；localparam 置于 parameter_port_list 末尾为合法 SV-2012 写法。判定：位宽计算不变，行为等价。

**登记准确性：通过（一处待主会话回填）。**
- 文件路径、原因（VCS O-2018.09 TCF-ICTC，struct 成员选择+常函数嵌套折叠失败）描述与实际改动自洽；"（2 处）"= localparam 定义（:81）+ 端口引用（:96）两处触点，属实。
- `复核` 列现为"待 rev"；本记录即复核结论，须由主会话将该列回填为 "REV-001 通过"（rev 不改 VENDOR.md）。

**完整性：通过。** 全 vendor 树补丁标记仅出现于 `VENDOR.md` 与 `floo_axi_chimney.sv` 两处（grep `本地补丁/P-001/RouteTblMsb` 无第三处），无未登记的 vendor 修改。

**备注：** 本机 `vcs` 存在，但复现 TCF-ICTC 需回退补丁后编译（越 vendor 只读边界，不做）；等价性已由静态推导确立，无需实跑。

---

## 四、指导意见（原理 + 方向，不贴实现）

1. **P-1**：文档里"时序切割"务必按数据流向归位——`CutAx`=进 AX 输入、`CutRsp`=入链路输入、`CutOup`=出链路输出。§1 的一句话总览要与 §2.2 字段表口径统一，否则覆盖点会照 §1 的错误方位建。修 §1 那句，去掉 `CutRsp` 的"出口"归属即可。
2. **P-2 / §0 缺口**：路由类 checker 的期望值 = "从地址切一个已知宽度的位段、port_id 补 0、再组成 dst_id struct"。要让期望唯一，spec 必须把三件事写死：`id_t` 坐标子字段位宽（=从哪 pin `NumX/NumY`）、`+:` 提取宽度等于该位宽、XY 下 `port_id` 恒 0。建议在 §0 适配表新增一行固定 `id_t` 布局（对齐 `floo_test_pkg` 的 NumX=NumY=4），§4.3 补"提取宽度 = 坐标字段位宽、port_id=0"。走 CLAUDE.md §8 流程（登记 bugs → rev 仲裁 → 主会话补 spec + pin），勿直接改正文。
3. **P-4**：把 `EnMgrPort=0 ⟹ NoRoB` 作为配置矩阵的合法性前置条件登记在 §8（或 §0 item3 的约束脚注），供 M3/M4 组配时排除非法组合。
4. **P-3**：§2.2 表头出处改为"docs/chimneys.md + RTL struct"，或对 `CutOup` 单独标注 RTL 来源。

以上均属"改文档措辞/补缺"，不涉及验证核心逻辑。

---

## 五、总结论（两项独立）

- **对象 1 — `doc/spec.md` v0：有条件通过。** 须改清单（改毕方可 pin）：
  1. P-1 修 §1 `CutRsp` 方位错标（阻塞项：事实错误）。
  2. P-2 补 §4.3 提取宽度 + `port_id=0`，并在 §0 pin `id_t`/坐标位宽（阻塞项：checker 唯一性）。
  3. P-3、P-4 建议同批修（非阻塞）。
- **对象 2 — 补丁 P-001：通过。** 行为等价、登记属实、无遗漏 vendor 改动；仅需主会话把 VENDOR.md `复核` 列回填为 "REV-001 通过"。

问题条数：对象 1 共 4 条（2 阻塞 P-1/P-2 + 2 非阻塞 P-3/P-4）；对象 2 共 0 条。

---

## 六、复核记录（2026-07-13，对照 spec.md 修改记录 #2）

| 原问题 | 复核结论 | 依据 |
| --- | --- | --- |
| P-1 §1 CutRsp 方位错标 | **通过** | `doc/spec.md:26-27` 已改为"wormhole arbiter → 出口 spill_register（CutOup）→ 链路；入链路 spill_register（CutRsp，作用于 floo_req_i/floo_rsp_i）→ 解包"。与 RTL 一致：CutOup 出口 spill 见 `floo_axi_chimney.sv:689-693`（i_req_out_cut）/`:718-722`（i_rsp_out_cut），`.Bypass(!ChimneyCfg.CutOup)`；CutRsp 入链路 spill 见 `:289` gen_rsp_cuts 作用于 `floo_req_i/floo_rsp_i`。§1 与 §2.2 口径已统一，原矛盾消除。 |
| P-2 §4.3 XY dst 提取歧义 | **通过（附 1 条 M1 跟验项）** | `doc/spec.md:99` 已给出精确公式 `dst.x = addr[XYAddrOffsetX +: $bits(id.x)]`、`dst.y = addr[XYAddrOffsetY +: $bits(id.y)]`、`dst.port_id = 0`，IdTable 情形 `dst = addr[IdAddrOffset +: $bits(id)]`——与 `floo_id_translation.sv:75-79` 逐条对应；REV-001 判据三要素（提取宽度=坐标位宽、port_id=0、位宽 pin 位置）均满足。位宽 pin 在 `tb/uvm/env/chimney_tb_cfg.sv`（M1 起唯一定义点）合规——与 CLAUDE.md §2"DUT 参数配置集中于该文件"约定一致，checker 期望值由公式+唯一定义点机械可推。**跟验项（非阻塞）**：当前 M0 该文件尚不存在（属正常，M1 建立）；M1 审查时须核对其中 id_t x/y 位宽实际落地且与所选 NumX/NumY（基线对齐 floo_test_pkg NumX=NumY=4，`floo_test_pkg.sv:19-20`）一致。 |
| P-3 §2.2 出处引用不精确 | **通过** | `doc/spec.md:45` 表头已补注"docs/chimneys.md Configuration 表 + floo_pkg.sv `chimney_cfg_t`；CutOup 仅后者有"，与实际来源（`floo_pkg.sv:313-315`）相符。 |
| P-4 §8 缺配置合法性约束 | **通过** | `doc/spec.md:137` 新增"EnMgrPort=0 时 B/R RoBType 必须为 NoRoB（RTL 断言强制）；M3/M4 配置矩阵须排除该非法组合"，与断言 NoMgrPortRobType（`floo_axi_chimney.sv:875-876`）一致，且明确了对 §0 item3 配置矩阵的排除责任。 |

程序合规性：修改记录 #2 已登记（`doc/spec.md:144`，章节/摘要/依据齐全）；本次整改属 rev 审查驱动的修订路径，符合 CLAUDE.md §8。

**复核总结论：对象 1（doc/spec.md v0）由"有条件通过"升级为通过，两项阻塞项（P-1/P-2）均消除，可执行 `docs.py --pin-spec` 钉住。** 遗留跟验项 1 条（P-2 的 chimney_tb_cfg.sv 位宽落地，挂 M1 审查核对，不阻塞本次 pin）。对象 2（P-001）结论不变：通过（VENDOR.md 复核列仍待主会话回填"REV-001 通过"）。
