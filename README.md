# floo_axi_chimney UVM 验证项目

对 [pulp-platform FlooNoC](https://github.com/pulp-platform/FlooNoC) v0.8.4 的 AXI4↔NoC 网络接口
`floo_axi_chimney` 进行独立 UVM 验证：单 DUT 拓扑（AXI master/slave agent + 主动 floo link agent），
覆盖 flit 打包/解包、wormhole 仲裁、RoB 三模式重排序、txnID 重映射、ATOP、多路由算法。

- **工具链**：Synopsys VCS-MX / Verdi O-2018.09（UVM-1.2），覆盖率口径 line+cond+fsm+tgl+branch+assert ≥90%
- **DUT**：`vendor/` 只读快照（上游 SHA 与本地补丁见 [vendor/VENDOR.md](vendor/VENDOR.md)）
- **规格**：[doc/spec.md](doc/spec.md)（单一事实源，从上游 docs+RTL 提炼、经审查 pin）
- **进度**：`make handover`（接手摘要）/ [doc/testplan.md](doc/testplan.md)（场景真值表与证据链）

## 快速开始

```bash
git config core.hooksPath .githooks   # 启用文档守卫（首次克隆后）
make handover                          # 当前状态一览
make smoke                             # 冒烟仿真
make run TEST=<test> SEED=<n>          # 单测（FSDB=1 抓波形，COV=1 收覆盖率）
make regress && make cov               # 回归 + 覆盖率报告
```

里程碑：M0 基建 → M1 UVM 环境+冒烟 → M2 功能覆盖+错误注入 → M3 多配置回归 → M4 覆盖率收敛（v1.0.0）。
工程约定（角色、证据规则、缺陷闭环）见 [CLAUDE.md](CLAUDE.md)。
