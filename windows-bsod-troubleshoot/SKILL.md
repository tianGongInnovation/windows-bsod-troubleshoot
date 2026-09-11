---
name: windows-bsod-troubleshoot
description: Windows 蓝屏（BSOD, Blue Screen of Death）排查。读取 C:\Windows\Minidump\ 下的 .dmp 文件，用 WinDbg + !analyze -v 找出肇事驱动/模块，常见停止代码（IRQL_NOT_LESS_OR_EQUAL / DPC_WATCHDOG_VIOLATION / WHEA_UNCORRECTABLE_ERROR / MEMORY_MANAGEMENT 等）速查表。触发词：蓝屏、蓝屏了、BSOD、蓝屏分析、蓝屏死机、minidump 解析、WinDbg、停止代码、stop code。
agent_created: true
version: 1.0.0
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

**根本方法论（承接排错铁律）**：**第一动作是 web 搜停止代码原文**，不是直接抓 dump。80% 的 BSOD 都能在搜索结果前 3 条锁定肇事驱动。

---

## 1. 蓝屏后第一动作：拍下停止代码

拍屏幕（或记下）这两条：
- **停止代码**（例如 `DRIVER_IRQL_NOT_LESS_OR_EQUAL`）
- **错误检查码**（例如 `0x000000D1`）

立即 web 搜：
```
"<停止代码>" "<错误检查码>" 驱动故障
"<停止代码>" "<错误检查码>" fix
"<停止代码>" "<错误检查码>" 蓝屏
```

**前 3 个高赞结果就够**，90% 锁定肇事驱动类别。

---

## 2. 加载 WinDbg 解析 minidump（深度分析）

拍下错误码还没解决 → 用 WinDbg 读 minidump 看"3 行定位真凶"。

### 2.1 确认 minidump 在生成

蓝屏时 Windows 默认在 `C:\Windows\Minidump\` 写小型转储文件（256 KB 起跳）：

```cmd
:: 在资源管理器地址栏粘贴：
C:\Windows\Minidump\
```

如果目录是空的 → 系统倾印设置被关了，按下面补：

1. `Win + R` → 输入 `sysdm.cpl` → 回车
2. **高级** 标签 → **启动和故障恢复** 区的 **设置**
3. **写入调试信息** 下拉选 **小内存转储(256 KB)**
4. 确认 **小转储目录** 是 `%SystemRoot%\Minidump`
5. 确定 → 重启电脑

⚠️ **此设置对未来 BSOD 才生效，过去已发生的 BSOD 不会有 dump**。如要分析已发生的，只能拍照停止代码搜。

### 2.2 安装 WinDbg（Microsoft Store 版）

⚠️ **2026 年起从 Microsoft Store 装新版 WinDbg**，不要去 Windows SDK 找老版"WinDbg Classic"（界面像 Windows XP 时代）。

- 方法 1（推荐）：打开 Microsoft Store → 搜 "WinDbg" → 找 **Microsoft Corporation** 发布的那一个（绿色虫子 bug 图标）→ 安装
- 方法 2（命令行党）：`winget install Microsoft.WinDbg`

### 2.3 配置符号路径

WinDbg 需要从 Microsoft 服务器下载符号（PDB），否则只能看到十六进制地址。

打开 WinDbg → **File** → **Settings** → **Debugging Settings** → **Default Symbol Path**：

```
srv*C:\Symbols*https://msdl.microsoft.com/download/symbols
```

说明：`srv*` 表示从服务器下载；`C:\Symbols` 是本地缓存（WinDbg 自动创建）。

### 2.4 打开 dump 跑 `!analyze -v`

1. **File** → **Open Crash Dump** → 选 `C:\Windows\Minidump\` 里最新那个 `.dmp` → 打开
2. 第一次会等 1-3 分钟（下载符号），直到右下角不再显示 `*BUSY*`
3. 底部命令栏输入：
   ```
   !analyze -v
   ```
4. 按 Enter，等 10 秒到 1 分钟出报告

### 2.5 速查"3 行定位"（重点中的重点）

`!analyze -v` 输出有几百行，**99% 跳过，只看这 3 行**：

| 字段 | 含义 | 怎么用 |
|---|---|---|
| `MODULE_NAME` | 肇事模块（去掉 .sys 后缀） | 这是**真凶**的函数/驱动名 |
| `IMAGE_NAME` | 真实文件 | 复制文件名到 Google 搜 |
| `FAILURE_BUCKET_ID` | 微软 WER 分类桶 | 格式 `bugcheck代號_子類型_模組名` |

**例**：

```
MODULE_NAME: nvlddmkm
IMAGE_NAME: nvlddmkm.sys
FAILURE_BUCKET_ID: 0x133_ISR_nvlddmkm!unknown_function
```

→ **NVIDIA 显卡驱动**是肇事者。处置：去 NVIDIA 官网装最新稳定版 / 退回上一版。

### 2.6 交叉验证 bugcheck code

报告最上方的"停止代码描述"段：

```
DPC_WATCHDOG_VIOLATION (133)
The DPC watchdog detected a prolonged run time at an IPL of DISPATCH_LEVEL or above.
```

(`133`) 就是 bugcheck code，配合 step 2.5 的 MODULE_NAME，得出完整因果链。

---

## 3. 常见停止代码速查表（背 5 个够用 90%）

| Bugcheck | 名称 | 常见肇事 | 典型处置 |
|---|---|---|---|
| `0xA` | `IRQL_NOT_LESS_OR_EQUAL` | 驱动访问了错误 IRQL 的内存 | 更新/回滚最近装/更新的驱动 |
| `0x50` | `PAGE_FAULT_IN_NONPAGED_AREA` | 驱动坏指针 / RAM 故障 | memtest 跑一遍 + 更新驱动 |
| `0x7E` | `SYSTEM_THREAD_EXCEPTION_NOT_HANDLED` | 系统线程未捕获异常 | 更新故障模块驱动 + 跑 SFC |
| `0x9F` | `DRIVER_POWER_STATE_FAILURE` | 驱动电源状态错乱 | 更新芯片组/显卡/网卡驱动；检查电源设置 |
| `0xD1` | `DRIVER_IRQL_NOT_LESS_OR_EQUAL` | 驱动在错误 IRQL 访问内存 | 更新/重装 !analyze 识别的驱动 |
| `0xEF` | `CRITICAL_PROCESS_DIED` | 系统关键进程挂了 | SFC + DISM + 查杀恶意软件 + 考虑系统还原 |
| `0x124` | `WHEA_UNCORRECTABLE_ERROR` | **纯硬件错误**（CPU/PCIe/RAM） | memtest + 查 CPU 温度 + 拔掉新加的硬件 |
| `0x133` | `DPC_WATCHDOG_VIOLATION` | 驱动 DPC 卡死 | 更新显卡/SSD firmware + 存储驱动 |
| `0x139` | `KERNEL_SECURITY_CHECK_FAILURE` | 驱动 stack buffer overrun / RAM | 更新驱动 + memtest |
| `0x1A` | `MEMORY_MANAGEMENT` | RAM 硬件或页表管理 | memtest 必跑 |
| `0xC000021A` | `STATUS_SYSTEM_PROCESS_TERMINATED` | Winlogon / CSRSS 挂了 | 进安全模式 → SFC + DISM → 考虑系统还原 |

---

## 4. 修复路径决策树

```
BSOD 发生
    │
    ├── 拍了停止代码？
    │   ├── YES → 第 1 步：web 搜停止代码 → 按搜索结果处置
    │   └── NO（只看到蓝屏画面） → 第 2 步：先补 minidump 设置（2.1）→ 等下次 BSOD
    │
    ├── web 搜没解决？
    │   └── YES → 第 2 步：WinDbg !analyze -v → 看 3 行
    │
    ├── 3 行定位到驱动名？
    │   ├── YES → 去设备管理器回滚/更新该驱动
    │   │         (右键设备 → 属性 → 驱动程序 → 回滚驱动程序)
    │   └── NO（指向 ntoskrnl.exe） → 内存/硬件问题
    │                    → 跑 Windows 内存诊断（mdsched.exe）
    │                    → 跑 memtest86
    │                    → 检查温度（HWMonitor / HWiNFO）
    │
    └── 多次 BSOD 反复出现
        ├── 同一驱动 → 换驱动版本（最新 → 上一稳定版）
        ├── 0x124 / 0x1A → 拔掉新装的硬件（RAM 条/SSD/显卡）
        └── 不明 → 干净启动（msconfig → 服务 → 隐藏 Microsoft 全部禁用）
```

---

## 5. 不要做的事

- ❌ **不要"以管理员身份运行"老程序**（包括 WinDbg 早期版本）——WinDbg 新版不需要
- ❌ **不要先 SFC / DISM / 重装系统**——99% 的 BSOD 是驱动问题，1 分钟搜停止代码 + 5 分钟 WinDbg 定位 = 5 分钟搞定
- ❌ **不要把 BSOD 简单归为"硬件坏了"**——75% 是驱动问题，0x124 才是真硬件
- ❌ **不要在蓝屏后立刻重装系统**——先用 WinDbg 看 dump，往往几分钟就知道是哪个驱动
- ❌ **不要清空 `C:\Windows\Minidump\` 后再去找"上次 BSOD 原因"**——清了就什么都没了

---

## 6. 模型选择（按分工铁律）

本技能涉及**微软技术栈深度**（Windows 内核 / WinDbg / 驱动调试）：
- **默认用国外模型**（CODEX / Claude / GPT / Gemini）解析 dump 输出
- 不要用国内免费模型硬上——dump 输出的 3 行需要微软技术栈专业知识
- 如本机已装本地大模型，可走本地免费档；否则用国外模型更省时间

---

## 7. 常见案例（供参考）

> 蓝屏案例随时间沉淀，发现新案例后追加到本节，便于复盘"以前怎么修的"。典型肇事多为显卡/存储驱动，先更新或回滚对应驱动版本。

---

## 8. 沉淀参考

- 完整 WinDbg 教程原文：adersaytech.com 2026 版 + Dell KB 000149411
- 配套技能：wb-health-check（健康巡检，提前发现系统资源枯竭）、troubleshoot-first-rule（排错方法论第一铁律）
