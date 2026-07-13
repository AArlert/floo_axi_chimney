---
name: dispatch
description: 派单卡组装——派 rev 审查/仲裁/签核任务前按固定模板组装输入、过自查。每次派发 rev subagent 之前执行。
---

# 派单流程（本项目唯一 subagent = rev）

## 1. 卡型与必给输入（只放清单内的输入，防共模泄漏）

| 卡型 | 必给输入 | 禁放内容 |
| --- | --- | --- |
| rev 代码/checker 审查 | 待审文件路径清单、对应 spec 章节号、testplan 行 ID、审查清单（正确性/spec 锚定/是否照抄 RTL 行为） | 作者（用户或主会话）对代码正确性的自评与推理过程 |
| rev spec 审查 | doc/spec.md 章节范围、上游依据文件路径（vendor/floonoc/docs/、RTL 文件） | 主会话提炼时的取舍推理 |
| rev 仲裁 | bugs.md 条目 ID（现象/最小复现/spec 依据）、涉及 spec 章节号 | 任何一方的期望裁决方向 |
| rev 里程碑签核 | 里程碑号、`make next` 输出、evidence 目录路径 | 口头结论转述（让 rev 自己读原始材料） |

## 2. 派单前自查

- [ ] 卡内只有文件路径、章节号、条目 ID——没有主会话或用户的推理过程与预设结论。
- [ ] 缺陷仲裁前已在 bugs.md 登记（禁止口头派单）。
- [ ] 任务卡写明交付判据（审查记录路径 doc/review/REV-xxx.md 或 doc/evidence/v0.M.P/review-M<N>.md）。
- [ ] rev 结论必须给依据（spec 章节号/证据路径），回收时缺依据就退回。

## 3. 回收核对

- 审查记录已写入 doc/review/（或里程碑签核入 doc/evidence/）；结论为"通过/不通过+问题清单"。
- rev 指出的问题：TB 代码问题由用户修复；文档/骨架问题由主会话修复；修复后视问题级别决定是否复审。
- `make docs-check` 过一遍再收单。
