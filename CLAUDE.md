# CLAUDE.md

floo_axi_chimney（pulp-platform FlooNoC 的 AXI4↔NoC 网络接口）的 UVM 验证项目。
DUT 为 vendor 只读快照（FlooNoC v0.8.4），交付物 = UVM 验证环境 + 六类覆盖率 ≥90%。
核心纪律：**厚存储 · 薄读口，机械交脚本 · 语义留 Agent，单一事实源 + 文档守卫**。

## 0. 角色（本项目只有三个）

| 角色 | 职责 | 边界 |
| --- | --- | --- |
| **用户**（DV 主力） | 亲手写 UVM 核心代码：sequence/driver/monitor/scoreboard/covergroup/SVA 的全部核心逻辑 | 本项目是用户的硬核 DV 经历——核心逻辑不代写 |
| **主会话**（orch+基础设施） | 目录/Makefile/脚本/flist 等基础设施；可编译空壳骨架（类文件+TODO 标记）；spec 机械提炼；派 rev 单与回收；记忆系统维护；xverif 协助定位问题 | **不写核心验证逻辑**——骨架只到 phase 方法签名与 TODO 注释为止；给指导用"原理+方向"，不贴实现 |
| **rev**（唯一 subagent，`.claude/agents/rev.md`） | 代码/checker/SVA 审查、spec 审查、仲裁、里程碑签核、书面指导 | 只读分析 + 书面记录（doc/review/），不直接改代码 |

派 rev 前用 skill `/dispatch` 组卡；卡内只给文件路径/章节号/条目 ID，不粘贴推理过程（防共模误读）。

## 1. 仓库结构

```
doc/       spec.md（单一事实源）+ 记忆系统 + testplan.md + feature-matrix.md + bugs.md
           + lint-waivers.md + evidence/ + review/（rev 书面记录）+ archive/（归档件）+ attachment/（图）
vendor/    DUT 与依赖库只读快照（VENDOR.md 记录上游 SHA；禁止未登记的修改）
tb/        uvm/（一类一文件，*_pkg.sv 汇总 include）+ sva/（协议断言，bind 挂接）
sim/       VCS 仿真入口（Makefile、flist/、regress/）
scripts/   机械工作脚本（docs.py / bump.py / regress.py / evidence.py）
.claude/   agents（rev）与 skills（handover/dispatch/evidence/closeout）
```

## 2. 语言与风格

- 注释、文档、commit message 用简体中文；标识符用英文；SystemVerilog 遵循 lowercase_with_underscores。
- UVM 一个类一个文件，由 `*_pkg.sv` 汇总 `include`；DUT 参数配置集中在 `tb/uvm/env/chimney_tb_cfg.sv`（M1 建立后为唯一定义点），禁止在别处硬编码。

## 3. 记忆系统 ★

**接手两步**：`make handover` + `make next`。禁止靠通读文件来接手。

三个滚动文件（doc/ 下）：
1. `status.jsonl` — 首行 = 当前总览（date/version/summary ≤200 字符）；历史快照在下。
2. `log.md` — 交接日志，块头 `## [版本] 日期 标题`，新的在上；仓库内最多 4 块，超限脚本归档。
3. `testplan.md` — 场景真值表，状态位 ✅/❌/⚠️/🔲（✅ 及证据由 evidence.py 回填）。

`make bump` 自动插入 TODO 骨架，agent 只填语义；docs-check 拦截未填的 TODO。归档件统一放 `doc/archive/`（`*-archive.*`），**默认不读**，只在追溯历史时 grep。

**Token 纪律**：grep/rg 定位再精读，不通读大文件；不读归档件；不读已 ✅ 条目细节；spec.md 按章节定位（`grep -n "^#" doc/spec.md` 取目录）；不通读 vendor RTL（按需 grep）。

## 4. 开发工作流（`make next` 告诉你现在该干什么）

1. `make handover` + `make next` 接手。
2. 新组件流转：**用户/主会话在 testplan.md 先登记场景行** → 主会话出可编译空壳骨架 → 用户填核心逻辑 → `make lint` + 仿真 → 派 rev 审查（书面记录）→ PASS 后 `make evidence` 登记。
3. 仿真：`make smoke` / `make run TEST=xx SEED=n [FSDB=1] [COV=1]` / `make regress [COV=1]`。
4. 收尾走 skill `/closeout`：`make bump` → 填 log 四问 + status summary → `make docs-check` → commit → **push**。

### 4.1 版本与里程碑

- 版本 `0.M.P` 存于 `version.json`（M=里程碑编号：M0 基建 / M1 UVM 环境+冒烟 / M2 功能覆盖+主动 floo agent+错误注入 / M3 多配置回归 / M4 覆盖率收敛→v1.0.0）。所有实质性变更都要 bump；里程碑完成打 git tag `v0.M.P`。
- **M 完成判据（三条硬条件，`make next` 机械核对）**：① 该 M 的 feature-matrix 关联场景全 ✅；② `make regress` 100% PASS 且证据归档；③ rev 签核记录存 doc/evidence/。

### 4.2 证据规则 ★（防验证造假）

- **没有仿真 log 就没有 ✅**。证据一律 `make evidence` 机械生成；**禁止手写证据文件**。
- 回归证据 `sim/result_summary.txt` 随里程碑复制入 evidence 目录；里程碑另需覆盖率 summary 摘录与 rev 审查记录。
- 汇报必须与 log 一致；仿真没跑、跑挂了，如实写 ❌/⚠️，不许"应该能过"。

### 4.3 缺陷闭环 ★

1. **登记**：发现 mismatch 先自查激励/检查器；仍疑似缺陷在 `doc/bugs.md` 登记——最小复现（TEST+SEED）、现象、期望及 spec 章节依据。**禁止只在对话里口头传递**。调试过程超一行的开 `doc/bugs/<BUG-ID>.md` 详情页。
2. **归属**：疑似 TB → 用户修复（rev 复核）；疑似 DUT（vendor RTL）→ 派 rev 确认，确认后记录+绕行（vendor 只读，可上报上游 issue），bug 置 WONTFIX 或按 waive 处理；疑似 spec 歧义 → rev 仲裁。
3. **修复→复验**：修复者回填根因与修复 commit（FIX_READY）→ 用登记的 TEST+SEED 复跑 + 相关回归，`make evidence BUG=<ID> ...` 机械关单。**关单人 ≠ 修复人**。

## 5. 仿真环境与工具探测 ★

本地 VM：VCS-MX O-2018.09-SP2 + Verdi 同版（UVM-1.2）。开工先探测：`command -v vcs`。
- **探测到**：必须真跑闭环——编译、lint、仿真、evidence.py 全部实际执行并以真实输出汇报；**有工具却只"声明未跑"视同违规**。
- **探测不到**：只做编码与文档工作，如实声明未编译/未仿真。

入口（根 Makefile 转发 sim/）：`make smoke / run / regress / cov / lint / verdi`。
- 覆盖率口径：**六类** line+cond+fsm+tgl+branch+assert，≥90% 合格。
- 本地另有 **xverif 验证调试工具箱**（用户级 skill，CLI 支持 `--json`）：`xdebug` 波形/设计库因果、`xcov` 覆盖率、`xloc` UVM log 定位、`xbit` 位运算、`xsva` 断言解释。重度 debug 优先用它而非通读 log。

## 6. Git 约定

- 中文 Conventional Commits（`feat:` `fix:` `docs:` `chore:` `test:`）。提交自包含：源码+测试+文档同一提交。
- **每次 `/closeout` 收尾 commit 后立即 `git push`**（用户长期授权动作）；push 失败如实汇报，不静默跳过、不 force push。
- 首次克隆后执行 `git config core.hooksPath .githooks` 启用软门禁。

## 7. 质量门禁

- 本地软门禁（pre-commit）：`docs.py --check` —— TODO 未填、版本失步、✅ 无证据、幽灵引用、缺陷单缺修复 commit/复验证据、spec 被悄改等，任一触发即拦截。
- lint 门禁：TB 代码交付条件——`make lint` 干净（判定范围限 tb/），或告警登记 doc/lint-waivers.md 经 rev 复核。
- 仿真硬门禁：`make regress` 100% PASS 是里程碑完成的必要条件。

## 8. 单一事实源

- `doc/spec.md` 是 DUT 行为规格与单一事实源（由主会话从 vendor/floonoc/docs/ 与 RTL 提炼、rev 审查后 pin）。每次修改必须：① "修改记录"表加条目；② `python3 scripts/docs.py --pin-spec` 重新钉住；③ 同步受影响的 testplan/bugs 条目。
- **修改路径唯一**：发现歧义/新行为 → 登记 bugs.md → rev 仲裁 → 主会话应用 + pin。禁止直改 spec 正文。
- checker/SVA 的期望值只准从 spec 推导，**禁止照抄 RTL 行为**（rev 专查此项）。
- vendor/ 只读：任何补丁须登记 VENDOR.md"本地补丁记录"并经 rev 复核。
