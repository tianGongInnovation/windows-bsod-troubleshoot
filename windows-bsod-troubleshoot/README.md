# 电脑蓝屏排查（windows-bsod-troubleshoot）

Windows 蓝屏（BSOD，Blue Screen of Death）排查技能：读系统转储文件（minidump），用 WinDbg（微软的蓝屏分析工具）+ `!analyze -v` 找出肇事驱动/模块，附常见停止代码速查表。

由 **天工创新坊** 开发维护，许可 **CC BY 4.0**（署名即可自由使用、修改、分发）。

## 有什么用处？

- 蓝屏后第一动作给对：拍下停止代码（如 `IRQL_NOT_LESS_OR_EQUAL`）立即搜索，80% 的蓝屏在搜索结果前 3 条锁定肇事驱动
- 需要深度分析时：教你在 `C:\Windows\Minidump\` 找转储文件，用 WinDbg 三行定位真凶（模块名、驱动、调用栈）
- 附常见停止代码速查表：`IRQL_NOT_LESS_OR_EQUAL`、`DPC_WATCHDOG_VIOLATION`、`WHEA_UNCORRECTABLE_ERROR`、`MEMORY_MANAGEMENT` 等对应什么部件、先查什么

## 为什么要用？

蓝屏最难的是"不知道从哪查"。网上资料零散，术语吓人。本技能把"拍码 → 搜索 → 解 dump → 定位驱动 → 针对性修复"整成一条普通人也能跟的路线，配合智能体执行，不猜、不盲修。

## 如何用？

1. 本技能是为 WorkBuddy 等智能体（Agent）准备的：安装到智能体的技能目录后，直接说"电脑蓝屏了，帮我排查"，智能体会按技能流程执行
2. 也可人工阅读本仓库 `SKILL.md`，按步骤自己排查
3. 蓝屏正在发生时先拍下停止代码和错误检查码，再按技能流程走

## 有问题怎么办？

- `C:\Windows\Minidump\` 是空的 → 按 SKILL.md 里的方法打开小内存转储设置（此设置只对以后的蓝屏生效，过去的蓝屏只能拍停止代码搜）
- WinDbg 没装/不会装 → 按 SKILL.md 的安装指引操作（微软商店或 SDK 下载均可）
- 定位到肇事驱动后怎么处理 → 常见处置：回滚/更新该驱动、卸载相关软件；不确定时把 WinDbg 输出发给智能体分析
- 问题反馈：本仓库 Issues（优先）或联系邮箱 gouzhongwu@vip.qq.com

## 许可

CC BY 4.0（署名许可）。内容由天工创新坊独立总结编写；如与第三方作品雷同，纯属巧合——提供确切证据即主动撤下。
