---
name: windows-bsod-troubleshoot
description: Windows 蓝屏（BSOD, Blue Screen of Death）排查。读取 C:\Windows\Minidump\ 下的 .dmp 文件，用 WinDbg + !analyze -v 定位肇事驱动或模块，并附常见停止代码（IRQL_NOT_LESS_OR_EQUAL / DPC_WATCHDOG_VIOLATION / WHEA_UNCORRECTABLE_ERROR / MEMORY_MANAGEMENT 等）速查表。触发词：蓝屏、蓝屏了、BSOD、蓝屏分析、蓝屏死机、minidump 解析、WinDbg、停止代码、stop code。
agent_created: true
version: 1.0.1
author: 天工创新坊
license: CC BY 4.0
display_name: "蓝屏排查"
display_name_en: Windows BSOD Troubleshoot
trigger: ["蓝屏", "蓝屏了", "BSOD", "蓝屏分析", "蓝屏死机"]
description_zh: "Windows 蓝屏排查：WinDbg 定位肇事驱动 + 常见停止代码速查"
description_en: "Windows BSOD troubleshoot with WinDbg culprit driver identification"
category: development
---

# Windows 蓝屏（BSOD）排查

**方法要点（承接排错铁律）**：**首要动作是检索停止代码原文**，而非直接分析 dump。多数 BSOD 可在搜索结果前三条中锁定肇事驱动。

---

## 1. 蓝屏后首要动作：记录停止代码

记录以下两项：

- **停止代码**（例如 `DRIVER_IRQL_NOT_LESS_OR_EQUAL`）
- **错误检查码**（例如 `0x000000D1`）

随后检索：

```
"<停止代码>" "<错误检查码>" 驱动故障
"<停止代码>" "<错误检查码>" fix
"<停止代码>" "<错误检查码>" 蓝屏
```

**取前三条高赞结果**，多数情况下可锁定肇事驱动类别。

---

## 2. 加载 WinDbg 解析 minidump（深度分析）

记录错误码后仍未解决，可用 WinDbg 读取 minidump，定位到具体模块。

### 2.1 确认 minidump 在生成

蓝屏时 Windows 默认在 `C:\Windows\Minidump\` 写入小型转储文件（自 256 KB 起）：

```cmd
:: 在资源管理器地址栏粘贴：
C:\Windows\Minidump\
```

若目录为空，说明系统倾印设置未开启，按以下步骤补上：

1. `Win + R` → 输入 `sysdm.cpl` → 回车
2. **高级** 标签 → **启动和故障恢复** 区的 **设置**
3. **写入调试信息** 下拉选 **小内存转储(256 KB)**
4. 确认 **小转储目录** 是 `%SystemRoot%\Minidump`
5. 确定 → 重启电脑

⚠️ **此设置对未来 BSOD 生效，已发生的 BSOD 不会补生成 dump**。若要分析已发生的，只能记录停止代码后检索。

### 2.2 安装 WinDbg（Microsoft Store 版）

⚠️ **2026 年起从 Microsoft Store 安装新版 WinDbg**，不建议从 Windows SDK 找老版"WinDbg Classic"（界面为 Windows XP 时期风格）。

- 方式一：打开 Microsoft Store → 搜 "WinDbg" → 选 **Microsoft Corporation** 发布的那一个（绿色虫子 bug 图标）→ 安装
- 方式二：命令行执行 `winget install Microsoft.WinDbg`

### 2.3 配置符号路径

WinDbg 需从 Microsoft 服务器下载符号（PDB），否则只能看到十六进制地址。

打开 WinDbg → **File** → **Settings** → **Debugging Settings** → **Default Symbol Path**：

```
srv*C:\Symbols*https://msdl.microsoft.com/download/symbols
```

说明：`srv*` 表示从服务器下载；`C:\Symbols` 是本地缓存（WinDbg 自动创建）。

### 2.4 打开 dump 运行 `!analyze -v`

1. **File** → **Open Crash Dump** → 选 `C:\Windows\Minidump\` 中最新的 `.dmp` → 打开
2. 首次运行需等待 1–3 分钟（下载符号），直至右下角不再显示 `*BUSY*`
3. 底部命令栏输入：
   ```
   !analyze -v
   ```
4. 按 Enter，等待 10 秒至 1 分钟生成报告

### 2.5 速查"三行定位"

`!analyze -v` 输出通常有几百行，重点看以下三行：

| 字段 | 含义 | 用途 |
|---|---|---|
| `MODULE_NAME` | 肇事模块（去掉 .sys 后缀） | 该函数/驱动的名称 |
| `IMAGE_NAME` | 真实文件 | 复制文件名到搜索引擎检索 |
| `FAILURE_BUCKET_ID` | 微软 WER 分类桶 | 格式 `bugcheck代号_子类型_模块名` |

**例**：

```
MODULE_NAME: nvlddmkm
IMAGE_NAME: nvlddmkm.sys
FAILURE_BUCKET_ID: 0x133_ISR_nvlddmkm!unknown_function
```

→ 肇事实为 **NVIDIA 显卡驱动**。处置：从 NVIDIA 官网安装最新稳定版，或退回上一版本。

### 2.6 交叉验证 bugcheck code

报告最上方的"停止代码描述"段：

```
DPC_WATCHDOG_VIOLATION (133)
The DPC watchdog detected a prolonged run time at an IPL of DISPATCH_LEVEL or above.
```

其中 `133` 即 bugcheck code，结合 2.5 的 `MODULE_NAME`，可得完整因果链。

---

## 3. 常见停止代码速查表

| Bugcheck | 名称 | 常见肇事 | 典型处置 |
|---|---|---|---|
| `0xA` | `IRQL_NOT_LESS_OR_EQUAL` | 驱动访问了错误 IRQL 的内存 | 更新或回滚最近装/更新的驱动 |
| `0x50` | `PAGE_FAULT_IN_NONPAGED_AREA` | 驱动坏指针 / RAM 故障 | 运行 memtest + 更新驱动 |
| `0x7E` | `SYSTEM_THREAD_EXCEPTION_NOT_HANDLED` | 系统线程未捕获异常 | 更新故障模块驱动 + 运行 SFC |
| `0x9F` | `DRIVER_POWER_STATE_FAILURE` | 驱动电源状态错乱 | 更新芯片组/显卡/网卡驱动；检查电源设置 |
| `0xD1` | `DRIVER_IRQL_NOT_LESS_OR_EQUAL` | 驱动在错误 IRQL 访问内存 | 更新或重装 `!analyze` 识别的驱动 |
| `0xEF` | `CRITICAL_PROCESS_DIED` | 系统关键进程终止 | SFC + DISM + 查杀恶意软件 + 系统还原 |
| `0x124` | `WHEA_UNCORRECTABLE_ERROR` | 多为硬件错误（CPU/PCIe/RAM） | memtest + 查 CPU 温度 + 移除新加的硬件 |
| `0x133` | `DPC_WATCHDOG_VIOLATION` | 驱动 DPC 卡死 | 更新显卡/SSD firmware + 存储驱动 |
| `0x139` | `KERNEL_SECURITY_CHECK_FAILURE` | 驱动 stack buffer overrun / RAM | 更新驱动 + memtest |
| `0x1A` | `MEMORY_MANAGEMENT` | RAM 硬件或页表管理 | 运行 memtest |
| `0xC000021A` | `STATUS_SYSTEM_PROCESS_TERMINATED` | Winlogon / CSRSS 终止 | 进安全模式 → SFC + DISM → 系统还原 |

---

## 4. 修复路径决策树

```
BSOD 发生
    │
    ├── 已记录停止代码？
    │   ├── 是 → 第 1 步：检索停止代码 → 按搜索结果处置
    │   └── 否（仅见蓝屏画面） → 第 2 步：先补 minidump 设置（2.1）→ 等下次 BSOD
    │
    ├── 检索未解决？
    │   └── 是 → 第 2 步：WinDbg !analyze -v → 看三行
    │
    ├── 三行定位到驱动名？
    │   ├── 是 → 到设备管理器回滚或更新该驱动
    │   │        (右键设备 → 属性 → 驱动程序 → 回滚驱动程序)
    │   └── 否（指向 ntoskrnl.exe） → 内存或硬件问题
    │                    → 运行 Windows 内存诊断（mdsched.exe）
    │                    → 运行 memtest86
    │                    → 检查温度（HWMonitor / HWiNFO）
    │
    └── 多次 BSOD 反复出现
        ├── 同一驱动 → 更换驱动版本（最新 → 上一稳定版）
        ├── 0x124 / 0x1A → 移除新装的硬件（RAM 条/SSD/显卡）
        └── 不明 → 干净启动（msconfig → 服务 → 隐藏 Microsoft 全部禁用）
```

---

## 5. 注意事项

- 老程序（含 WinDbg 早期版本）通常无需"以管理员身份运行"；WinDbg 新版亦不需要
- 通常先检索停止代码并用 WinDbg 定位驱动，再考虑 SFC / DISM / 重装系统
- BSOD 多由驱动引起，`0x124` 一类才指向真实硬件问题，不宜直接归因于"硬件损坏"
- 蓝屏后建议先分析 dump，再判断是否需要重装系统
- `C:\Windows\Minidump\` 中的文件在定位原因前不宜清空

---

## 6. 模型选择（按分工约定）

本技能涉及**微软技术栈深度**（Windows 内核 / WinDbg / 驱动调试）：
- **默认使用国外模型**（CODEX / Claude / GPT / Gemini）解析 dump 输出
- dump 输出的三行需微软技术栈专业知识，国内免费模型在解析准确度上有限
- 如本机已装本地大模型，可走本地免费档；否则用国外模型更省时间

---

## 7. 常见案例（供参考）

> 蓝屏案例随时间沉淀，发现新案例后可追加到本节，便于复盘。典型肇事多为显卡或存储驱动，可先更新或回滚对应驱动版本。

---

## 8. 沉淀参考

- 完整 WinDbg 教程原文：adersaytech.com 2026 版 + Dell KB 000149411
- 配套技能：wb-health-check（健康巡检，提前发现系统资源枯竭）、troubleshoot-first-rule（排错方法论第一铁律）
