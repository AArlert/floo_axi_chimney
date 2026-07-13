# 验证场景真值表

状态位：✅ 通过（证据由 evidence.py 回填）/ ❌ 失败 / ⚠️ 部分 / 🔲 未做。
场景登记先于编码；✅ 必须有 `doc/evidence/` 证据与含 SEED 的复现命令。

| ID | 里程碑 | 场景描述 | 配置 | 状态 | 证据 | 复现 |
| --- | --- | --- | --- | --- | --- | --- |
| M0-01 | M0 | 上游背靠背双 chimney tb 在 VCS 2018 编译+仿真跑通（排雷 struct 参数/宏常量兼容性） | 上游默认（NoRoB+XY） | ✅ | doc/evidence/v0.0.0/M0-01.log | `make run TEST=upstream_sanity SEED=1` |
