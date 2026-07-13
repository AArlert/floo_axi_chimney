# 验证场景真值表

状态位：✅ 通过（证据由 evidence.py 回填）/ ❌ 失败 / ⚠️ 部分 / 🔲 未做。
场景登记先于编码；✅ 必须有 `doc/evidence/` 证据与含 SEED 的复现命令。

| ID | 里程碑 | 场景描述 | 配置 | 状态 | 证据 | 复现 |
| --- | --- | --- | --- | --- | --- | --- |
| M0-01 | M0 | 上游背靠背双 chimney tb 在 VCS 2018 编译+仿真跑通（排雷 struct 参数/宏常量兼容性） | 上游默认（NoRoB+XY） | ✅ | doc/evidence/v0.0.0/M0-01.log | `make run TEST=upstream_sanity SEED=1` |
| M2-BP01 | M2 | 背压：floo req 链路对端 ready 随机/长周期拉低（NoC 拥塞）——flit 不丢不重、AXI 侧正确反压、valid 期间 flit 稳定 | 基线 | 🔲 | - | - |
| M2-BP02 | M2 | 背压：floo rsp 链路 ready 背压——响应通路完整、不因 rsp 堵塞死锁 req | 基线 | 🔲 | - | - |
| M2-BP03 | M2 | 背压：axi_out 慢从（AW/W/AR ready 延迟 Fast/Slow/Random）——meta_buffer 满时入口 stall 而非丢事务 | 基线 | 🔲 | - | - |
| M2-BP04 | M2 | 背压：axi_in 慢主（B/R ready 延迟）——RoB/直通路径反压传导正确 | 基线 | 🔲 | - | - |
| M2-BP05 | M2 | 背压极限：全接口背压 + outstanding 打满 MaxTxns——无死锁、无丢包、撤压后吞吐恢复 | 基线 | 🔲 | - | - |
