# 在笔记本上安装 macOS 26：一份完整的踩坑记录

一台没有核显输出的 Intel 游戏本，装上 macOS 26.7.1，用第三方 NVIDIA 驱动点亮 RTX 4060，解决倒挂屏旋转，并在过程中定位到一个可复现的内核 use-after-free。

> **免责声明**：黑苹果违反 Apple 的软件许可协议，且在不支持的硬件上运行未签名内核扩展存在数据丢失与硬件风险。本文仅记录技术过程与排查方法，不构成操作建议。所有序列号、UUID、网卡地址等设备唯一标识均已移除。

---

## 一、为什么值得写下来

黑苹果的教程很多，但绝大多数停留在「照着 config 抄」。真正卡住人的从来不是配置模板，而是**出问题之后怎么定位**：

- 引导卡在进度条，日志里没有任何有用信息；
- OpenCore 明明有 macOS 条目，回车却进了 Windows；
- 系统装好了，触控板、键盘全不工作；
- 显卡驱动加载了，但屏幕是倒的；
- 能进系统，但随机在开机 40 秒时内核崩溃。

这五个问题在这台机器上全遇到了。本文记录每一个的实际排查路径，包括走过的弯路。

---

## 二、硬件

| 项目 | 规格 | 兼容性 |
|---|---|---|
| 机型 | 机械革命 极光X（GM6IQ0C） | — |
| CPU | Intel Core i7-14650HX（Raptor Lake-HX Refresh） | 需 `ProvideCurrentCpuInfo` |
| 核显 | Intel UHD（Raptor Lake，设备 ID 不在 macOS 支持列表） | **不可用**，全程屏蔽 |
| 独显 | NVIDIA RTX 4060 Laptop（AD107，`10DE:28A0`） | 官方不支持，靠第三方驱动 |
| 内存 | 32 GB DDR5-5600（16×2） | 原生支持 |
| 硬盘 | 1 TB NVMe（PCIe 4.0） | 原生支持 |
| 无线 | Intel AX201（Wi-Fi 6 + 蓝牙 5.2） | 需 `itlwm` + `IntelBluetoothFirmware` |
| 有线 | Realtek RTL8168 | 需 `RealtekRTL8111` |
| 声卡 | Conexant/Senary（`14F1:1F87`） | 需 `AppleALC` + layout-id |
| 触控板 | I2C HID（`ACPI\UNIW0001`） | 需 VoodooI2C 系列（本机未完全解决） |
| 屏幕 | 1920×1080 @180Hz，双面板 | 需处理旋转与刷新率 |

**核心矛盾**：这台机器的 Intel 核显不被 macOS 支持，而 NVIDIA 独显也不被官方支持。也就是说，**没有任何一块 GPU 是「合法」可用的**。

这是整个项目难度的根源——不是配置复杂，是根本没有官方支持的显示通路。

---

## 三、最终状态

| 功能 | 状态 | 备注 |
|---|---|---|
| 系统启动 | 可用 | OpenCore 1.0.x + macOS 26.7.1 |
| 显卡加速 | 可用 | 第三方 NVRM 驱动 2.6.5，Metal 可用 |
| 显示输出 | 可用 | 内置屏 + 外接，需旋转 180° |
| 音频 | 可用 | AppleALC，layout-id 需实测匹配 |
| 有线网络 | 可用 | RealtekRTL8111 |
| 无线网络 | 可用 | itlwm（非原生 Airport，部分特性缺失） |
| 蓝牙 | 部分 | 能识别设备，连接稳定性一般 |
| 触控板 | 部分 | 基础移动可用，多点手势不完整 |
| 键盘 | 可用 | |
| 睡眠/唤醒 | 不可用 | 唤醒后黑屏，已禁用睡眠 |
| 隔空投送 / 接力 | 不可用 | 依赖原生无线网卡 |
| iMessage / FaceTime | 不可用 | 需有效序列号，未配置 |

**这是一台「能用」的机器，不是「好用」的机器。** 把它当主力开发机需要接受：不能睡眠、不能隔空投送、触控板手感一般。

---

## 四、技术难点概览

四个问题占了总工作量的 80%：

### 4.1 没有任何官方支持的 GPU

macOS 从 10.14 起就不再支持 NVIDIA 的 Web Driver，而 Raptor Lake 的核显也不在支持列表。

解法是用社区维护的第三方驱动（本文称 NVRM 驱动），它重新实现了 NVIDIA 内核扩展的接口。代价是：不稳定、功能不完整、每次系统更新都可能失效。

详见 [`docs/nvrm.md`](docs/nvrm.md)。

### 4.2 开机 40 秒随机内核崩溃

驱动能用，但大约 35–45% 的开机概率会触发 `kalloc` 堆损坏 panic：

```
[kalloc.type.var6.16]: element modified after free
```

这不是配置问题，是驱动本身的 use-after-free 缺陷。花了两天才确认这一点——中间走过「以为是内存条」「以为是 SMBIOS」「以为是 ACPI 补丁」的弯路。

详见 [`docs/panic-analysis.md`](docs/panic-analysis.md)。

### 4.3 屏幕倒挂

机器装了倒挂的显示面板（或排线定义相反），Windows 下靠驱动旋转 180°，macOS 下没有这个能力。

解法是修改显示相关的 kext，在 EDID 层面做旋转。

详见 [`docs/display-rotation.md`](docs/display-rotation.md)。

### 4.4 OpenCore 识别不到 macOS 分区

现象极具误导性：OpenCore 界面里明明有 macOS 条目，选中回车后却启动了 Windows。

根因是 UEFI 启动项的排序与 OpenCore 的扫描策略冲突。这个问题在论坛上被归因为十几种原因，实际排查要靠 `Misc → Debug` 打开完整日志。

详见 [`docs/journey.md`](docs/journey.md)。

---

## 五、目录

```
.
├── docs/
│   ├── journey.md              # 完整时间线：从评估到可用的 9 天
│   ├── opencore.md             # EFI 配置详解（SMBIOS / ACPI / kext / Quirks）
│   ├── nvrm.md                 # 第三方 NVIDIA 驱动的部署与限制
│   ├── panic-analysis.md       # use-after-free panic 的完整分析
│   ├── display-rotation.md     # 倒挂屏的 180° 旋转方案
│   └── troubleshooting.md      # 排查手册：症状 → 可能原因 → 验证方法
├── files/
│   ├── config-skeleton.plist   # 脱敏的 config.plist 骨架
│   ├── kext-list.md            # kext 清单、加载顺序与作用
│   └── boot-args.md            # 启动参数的演进与含义
└── README.md
```

---

## 六、给后来者的话

1. **先确认有没有显示通路**。没有核显、独显又不被支持时，不要开始——除非你愿意接受第三方驱动的所有不确定性。
2. **OC 的日志比任何教程都有用**。`Misc → Debug → Target = 67` 打开完整日志，很多「玄学问题」在日志里写得很清楚。
3. **一次只改一个变量**。EFI 配置有几十个开关，同时改三个，出问题永远不知道是哪个。
4. **保留每一次可启动的 EFI 备份**。命名为日期 + 状态（`EFI-20261026-bootable`），不要只留「最新版」。
5. **区分「配置错误」和「驱动缺陷」**。前者能改配置解决，后者只能等上游修或绕过。这个判断错了会浪费大量时间。

这台机器现在能日常使用，但它始终是一个**在悬崖边工作的系统**——一次系统更新就可能全部作废。这不是技术问题，是这个方案的固有属性。
