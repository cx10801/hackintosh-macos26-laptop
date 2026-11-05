# EFI 配置详解

本文解释每一项为什么这么配，而不是给一份可以直接抄的模板。抄模板在黑苹果里几乎必然失败——硬件差异太大。

> 完整骨架见 [`../files/config-skeleton.plist`](../files/config-skeleton.plist)，序列号等已清空。

---

## 1. SMBIOS 机型选择

最终选择：**`iMacPro1,1`**

### 为什么

这台机器的关键特征是「**没有可用的核显**」。所以机型必须满足：原生就没有核显输出，或者核显不参与主显示通路。

| 候选机型 | 核显情况 | 结论 |
|---|---|---|
| `iMac19,1` | 有核显，且依赖核显做 QuickSync | 弃（核显不可用会导致硬解、随航失效） |
| `Macmini8,1` | 同上 | 弃 |
| `MacPro7,1` | 无核显，纯独显 | 可用，但机型过新，部分驱动行为不同 |
| **`iMacPro1,1`** | **无核显，纯 AMD 独显** | **选用** |
| `MacBookPro16,2` | 有核显 | 弃（本机曾用过，但在无核显场景下不稳） |

`iMacPro1,1` 是唯一一个「原生设计上就完全依赖独显」的机型。虽然它配的是 AMD 显卡，但 SMBIOS 层面的假设（无核显、纯独显输出）和这台机器一致。

### 需要同步设置的项

```
PlatformInfo → Generic
  SystemProductName    = iMacPro1,1
  SpoofVendor          = true      # 伪装厂商为 Apple
  AdviseFeatures       = false     # 不向系统声明额外特性
  MaxBIOSVersion       = false     # 不限制 BIOS 版本
  SystemMemoryStatus   = Auto
```

`UpdateSMBIOSMode` 设为 `Custom`，配合 `CustomSMBIOSGuid`（Kernel Quirk）——避免写入真实的 SMBIOS 数据，减少与固件冲突的可能。

### 序列号

**必须自己生成，不能用别人的。** 三个字段要配套：

- `SystemSerialNumber`：12 位，需通过 Apple 的格式校验
- `MLB`：17 位主板序列号
- `SystemUUID`：36 位，需与序列号有对应关系

生成工具会给出一组配套值。**不要混用不同来源的序列号和 MLB**——这会导致 iCloud 登录失败或设备被标记。

> 本文不提供任何真实序列号。仓库里的 `config-skeleton.plist` 这几个字段是空的。

---

## 2. ACPI 补丁

三张 SSDT，各有明确目的：

| 文件 | 作用 | 为什么需要 |
|---|---|---|
| `SSDT-PLUG-DRTNIA.aml` | 注入 CPU 电源管理 | 让 XCPM 接管频率调节；不放会导致高频不降、发热严重 |
| `SSDT-EC-USBX-LAPTOP.aml` | 伪造 EC 设备 + USB 供电属性 | macOS 需要 `EC` 设备才加载电池管理；USBX 提供正确的供电参数 |
| `SSDT-PMC.aml` | 伪造 PMC 设备 | 原生 NVRAM 支持；不放会导致 NVRAM 写入失败，启动参数存不住 |

### 顺序有讲究

`SSDT-PLUG` 必须在 `SSDT-EC-USBX` 之前加载。原因是电源管理依赖 EC 提供的温度传感器接口，顺序反了会读不到传感器。

### 没有用的表

试过但删掉的：

- `SSDT-AWAC`：本机 RTC 已经是 AWAC 模式，不需要强制
- `SSDT-RHUB`：本机 USB 控制器不需要重置
- `SSDT-USBW`：和 USBToolBox 的方案冲突

**教训：SSDT 不是越多越好。** 每多一张表，就多一个可能的冲突源。

---

## 3. Kext 加载顺序

顺序不能乱，依赖关系是硬性的：

```
1. Lilu.kext                    ← 所有插件的加载器，必须第一个
2. VirtualSMC.kext              ← 依赖 Lilu
3. ├─ SMCProcessor.kext         ← 依赖 VirtualSMC，CPU 传感器
4. ├─ SMCSuperIO.kext           ← 依赖 VirtualSMC，风扇/电压
5. └─ SMCBatteryManager.kext    ← 依赖 VirtualSMC，电池
6. AMFIPass.kext                ← 依赖 Lilu，允许加载未签名 kext
7. AppleALC.kext                ← 依赖 Lilu，音频
8. RealtekRTL8111.kext          ← 独立，有线网卡
9. BlueToolFixup.kext           ← 依赖 Lilu，蓝牙协议栈修补
10. IntelBTPatcher.kext         ← 依赖 Lilu，蓝牙固件加载
11. IntelBluetoothFirmware.kext ← 依赖上面两个
12. itlwm.kext                  ← 独立，Intel 无线网卡
13. USBToolBox.kext             ← 独立，USB 映射加载器
14. UTBDefault.kext             ← 依赖 USBToolBox，实际映射表
```

**三条硬规则**：

1. `Lilu` 永远第一。所有依赖它的 kext 都必须在它之后。
2. `VirtualSMC` 的四个子插件必须紧跟主插件，中间不能插别的。
3. `USBToolBox` 和 `UTBDefault` 必须相邻，顺序不能反。

### AMFIPass 的作用

这个 kext 是「让 AMFI（Apple Mobile File Integrity）放行未签名内核扩展」的关键。

配合启动参数：

```
amfi_get_out_of_my_way=0x1
amfi=0x80
```

**这两个参数和 AMFIPass 是同一件事的两种做法。** 同时用是为了提高成功率——某些系统版本下参数会被忽略，此时靠 kext 兜底。

代价：系统完整性保护被实质性削弱。这是加载第三方驱动的必要代价，不是「顺便关掉」。

---

## 4. Kernel Quirks

| Quirk | 值 | 为什么 |
|---|---|---|
| `AppleXcpmCfgLock` | true | 绕过 CFG Lock 检查（BIOS 里通常锁着） |
| `CustomSMBIOSGuid` | true | 配合 `UpdateSMBIOSMode = Custom` |
| `DisableIoMapper` | true | 关闭 VT-d 映射，避免与驱动冲突 |
| `DisableLinkeditJettison` | true | 防止内核符号被过早释放（`keepsyms=1` 需要） |
| `PanicNoKextDump` | true | panic 时不 dump kext 信息，减少日志噪音 |
| `PowerTimeoutKernelPanic` | true | 电源超时不触发 panic，避免睡眠相关崩溃 |
| `ProvideCurrentCpuInfo` | true | 向系统提供正确的 CPU 信息（Raptor Lake 必需） |
| `SetApfsTrimTimeout` | -1 | 禁用 APFS TRIM 超时（部分 NVMe 兼容性问题） |

### `ProvideCurrentCpuInfo` 是关键

Raptor Lake 是混合架构（P 核 + E 核）。macOS 不认识这种拓扑，不提供正确的 CPU 信息会导致：

- 只能识别部分核心
- 频率调节失效
- 性能异常低

这个 Quirk 会伪造一套 macOS 能理解的 CPU 拓扑。

---

## 5. Booter Quirks

| Quirk | 值 | 为什么 |
|---|---|---|
| `AvoidRuntimeDefrag` | true | 避免固件在运行时整理内存（几乎所有机器都需要） |
| `DevirtualiseMmio` | true | 解除 MMIO 虚拟化，让驱动能直接访问 |
| `EnableSafeModeSlide` | true | 安全模式下也允许 KASLR 偏移 |
| `ProtectUefiServices` | true | 保护 UEFI 服务不被覆盖（新版固件必需） |
| `ProvideCustomSlide` | true | 提供自定义 KASLR 偏移，避免内存冲突 |
| `SetupVirtualMap` | true | 建立虚拟内存映射（大部分固件必需） |
| `SyncRuntimePermissions` | true | 同步运行时内存权限（较新固件必需） |

**这台机器的固件比较新（AMI，2024 年），所以后三条都是必需的。** 老固件可能不需要，开了反而出问题。

---

## 6. boot-args 的演进

启动参数不是一次配好的，是随问题逐个加的：

### 第一版（安装阶段）

```
-v keepsyms=1 debug=0x100 npci=0x2000
```

- `-v`：详细模式，必须开，否则卡住时什么都看不到
- `keepsyms=1`：保留内核符号，panic 时能看到函数名
- `debug=0x100`：panic 时停留在屏幕，不自动重启（关键！否则看不到崩溃信息）
- `npci=0x2000`：解决 PCI 配置空间扫描问题

### 第二版（解决输入设备）

```
-vi2c-force-polling
```

强制 I2C 走轮询模式而不是中断。触控板初始化失败时这是唯一能进系统的方式。代价是 CPU 占用略高。

### 第三版（显卡驱动）

```
nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth gfxrst=4
```

详见 [`nvrm.md`](nvrm.md)。

### 第四版（崩溃缓解）

```
nvrmsettle=10000
```

详见 [`panic-analysis.md`](panic-analysis.md)。

### 最终

```
-v keepsyms=1 debug=0x100 npci=0x2000
-vi2c-force-polling
amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0
nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth gfxrst=4
-ibtcompatbeta revpatch=sbvmm
nvrmsettle=10000
```

**建议**：调试期保留 `-v` 和 `debug=0x100`。系统稳定后再去掉 `-v`，但 `keepsyms=1` 建议留着——下次出问题能省很多时间。

---

## 7. Debug 配置（排障必开）

```xml
<key>Misc</key>
<dict>
  <key>Debug</key>
  <dict>
    <key>AppleDebug</key><true/>
    <key>ApplePanic</key><true/>
    <key>DisableWatchDog</key><false/>
    <key>Target</key><integer>67</integer>
  </dict>
</dict>
```

`Target = 67` 表示同时输出到：

- 屏幕（`1`）
- 文件（`2`）
- 串口（`64`）—— 没有串口时忽略

日志会写到 EFI 分区的 `opencore-YYYY-MM-DD-HHMMSS.txt`。

**这个日志是本项目里最有用的工具。** 前面两天的「玄学问题」，打开日志后大部分都有明确提示。

---

## 8. 不要做的事

| 做法 | 后果 |
|---|---|
| 抄网上的现成 config | 硬件不同，Quirks 和 SSDT 几乎全错 |
| 同时改多个参数 | 出问题无法定位 |
| 用别人的序列号 | iCloud 被锁、设备被标记 |
| 删掉「看起来没用」的 Quirk | 某些 Quirk 是多个功能共同依赖的 |
| 系统更新后直接重启 | 驱动可能失效，先备份 EFI |
| 关闭 SIP 之外还关闭 AMFI 却不用 AMFIPass | 部分 kext 加载失败且无明显报错 |
