# 缺陷登记表

状态集合: OPEN/FIXING/FIX_READY/VERIFYING/CLOSED/TB_BUG/SPEC_CHANGED/WONTFIX。
疑似归属：TB / DUT / spec。DUT 是 vendor 只读快照——真 RTL 缺陷经 rev 确认后记录+绕行（可上报上游）。
调试过程超一行的开 doc/bugs/<BUG-ID>.md 详情页，表格行留摘要+链接。

| ID | 状态 | 疑似归属 | 现象摘要 | 最小复现 | 根因/裁决 | 修复 commit | 复验证据 |
| --- | --- | --- | --- | --- | --- | --- | --- |
