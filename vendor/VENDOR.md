# vendor 快照清单

DUT 与依赖库均为上游仓库的只读快照（bender 未部署，手动 vendor）。
**本目录内文件禁止随意修改**；如为工具兼容性必须打补丁，须在下方"本地补丁记录"登记 diff 摘要并经 rev 复核。

## 上游版本锁定

| 目录 | 上游仓库 | tag | commit SHA | 拷贝范围 |
| --- | --- | --- | --- | --- |
| floonoc/ | https://github.com/pulp-platform/FlooNoC | v0.8.4 | 14c253c996fcdc78b793fe28ac18964769b768df | hw/（除 deprecated/）+ docs/floonoc/ |
| axi/ | https://github.com/pulp-platform/axi | v0.39.9 | a256a3b86394fedf19e361047fccfdd7f6ef83e4 | src/ + include/ |
| common_cells/ | https://github.com/pulp-platform/common_cells | v1.39.0 | 9ca8a7655f741e7dd5736669a20a301325194c28 | src/ + include/ |
| common_verification/ | https://github.com/pulp-platform/common_verification | v0.2.5 | fb1885f48ea46164a10568aeff51884389f67ae3 | src/ |
| tech_cells_generic/ | https://github.com/pulp-platform/tech_cells_generic | v0.2.13 | 7968dd6e6180df2c644636bc6d2908a49f2190cf | src/ |

版本依据：FlooNoC v0.8.4 的 Bender.lock（axi 0.39.9 / common_cells 1.39.0 / common_verification 0.2.5 / tech_cells_generic 0.2.13）。
axi_riscv_atomics、idma、cvfpu 未 vendor——chimney 单元验证不需要（它们服务于 mesh/DMA/reduction 场景）。

## 本地补丁记录

| # | 文件 | 改动 | 原因 | 复核 |
| --- | --- | --- | --- | --- |
| P-001 | floonoc/hw/floo_axi_chimney.sv | 端口 `route_table_i` 位宽表达式 `floo_iomsb(RouteCfg.NumRoutes)` 改经参数表 `localparam RouteTblMsb` 中转（2 处，行为等价） | VCS O-2018.09 无法在端口位宽处折叠"struct 参数成员选择+常量函数调用"，报 TCF-ICTC（最小探针已复现：直接成员选择可折叠、localparam 中转可折叠，仅两者嵌套组合失败） | REV-001 通过 |
