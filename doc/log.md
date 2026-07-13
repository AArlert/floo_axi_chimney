# 交接日志（新的在上；仓库内最多 4 块，超限 make docs-archive 归档到 log-archive.md）

## [0.0.1] 2026-07-13 M0 基建完成：vendor+脚本体系+VCS 排雷+spec v0

**做了什么**
- vendor 快照：FlooNoC v0.8.4 + axi 0.39.9 / common_cells 1.39.0 / common_verification 0.2.5 / tech_cells_generic 0.2.13，SHA 记录于 vendor/VENDOR.md；flist 由各库 Bender.yml 依赖序生成（sim/flist/{vendor,dut,tb_upstream}.f）。
- 移植 ppa-lite 脚本体系（bump/docs/evidence/regress + 4 skills + githooks），适配三角色制（用户=DV 主力 / 主会话=基础设施 / rev=唯一 subagent）；交付推导改为 tb/ 组件现算。
- VCS 2018 排雷：TCF-ICTC（struct 参数成员选择+常量函数做端口位宽）——最小探针定位，补丁 P-001（localparam 中转，行为等价，REV-001 复核通过）。另 -assert svaext 必需；evidence.py 增加非 UVM log 兜底判定与相对路径修复。
- 上游背靠背 tb 编译+仿真 PASS（2×1000 随机读写 0 Error），M0-01 ✅（doc/evidence/v0.0.0/M0-01.log）。
- spec.md v0 提炼并经 rev 两轮审查（REV-001：4 条问题整改后复核通过）后 pin。

**没做什么**
- 未建 UVM 环境（tb/uvm 空，M1 交付）；regress.list 为空（M1 起生效）；未部署 CI。
- sim/Makefile 的 make run 对非 UVM tb 会停在 $stop 交互提示（M0-01 复现实际用 make smoke）。

**下一步**
- M1：主会话出 UVM 骨架（axi_master/slave_agent、floo_link_monitor、env/test/pkg + chimney_tb_cfg.sv 唯一配置点），用户填核心逻辑；smoke 切换为 UVM 测试。
- REV-001 跟验项：M1 审查时核对 chimney_tb_cfg.sv 的 id_t x/y 位宽与 NumX/NumY=4 基线一致。

**如何验证**
- make handover / make next / make docs-check 全通过；make smoke 重跑上游 sanity（约 15s）；补丁详情 grep P-001 vendor/VENDOR.md。

