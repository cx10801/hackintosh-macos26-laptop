# Kext 清单

14 个内核扩展，加载顺序有硬性依赖。

| # | Kext | 版本 | 来源 | 作用 | 依赖 |
|---|---|---|---|---|---|
| 1 | `Lilu` | 1.7.x | 开源 | 插件加载器，所有插件的宿主 | — |
| 2 | `VirtualSMC` | 1.3.x | 开源 | 模拟 SMC 芯片 | Lilu |
| 3 | `SMCProcessor` | 1.3.x | 开源 | CPU 温度/功耗传感器 | VirtualSMC |
| 4 | `SMCSuperIO` | 1.3.x | 开源 | 风扇转速/电压 | VirtualSMC |
| 5 | `SMCBatteryManager` | 1.3.x | 开源 | 电池状态 | VirtualSMC |
| 6 | `AMFIPass` | 1.4.1 | 开源 | 放行未签名 kext | Lilu |
| 7 | `AppleALC` | 1.9.8 | 开源 | 音频编解码 | Lilu |
| 8 | `RealtekRTL8111` | 2.5.x | 开源 | 有线网卡 | — |
| 9 | `BlueToolFixup` | 2.7.x | 开源 | 蓝牙协议栈修补 | Lilu |
| 10 | `IntelBTPatcher` | 2.4.x | 开源 | 蓝牙固件加载路径修补 | Lilu, BlueToolFixup |
| 11 | `IntelBluetoothFirmware` | 2.4.x | 开源 | Intel 蓝牙固件 | 上两个 |
| 12 | `itlwm` | 2.3.x | 开源 | Intel 无线网卡 | — |
| 13 | `USBToolBox` | 1.1.x | 开源 | USB 映射加载器 | — |
| 14 | `UTBDefault` | 生成 | 工具生成 | 实际端口映射表 | USBToolBox |

---

## 顺序约束

```
Lilu
 └─ VirtualSMC
     ├─ SMCProcessor
     ├─ SMCSuperIO
     └─ SMCBatteryManager
 └─ AMFIPass
 └─ AppleALC
 └─ BlueToolFixup
     └─ IntelBTPatcher
         └─ IntelBluetoothFirmware

（独立）
RealtekRTL8111
itlwm
USBToolBox
 └─ UTBDefault
```

三条不能违反的规则：

1. **`Lilu` 必须排第一**。它在启动早期挂钩内核，晚了就失效。
2. **`VirtualSMC` 及其子插件必须连续**。中间插入其他 kext 会导致传感器读不到。
3. **`USBToolBox` 与 `UTBDefault` 必须相邻，且顺序固定**。反了映射不生效。

---

## 不在列表里的东西

| 常见但未使用 | 原因 |
|---|---|
| `WhateverGreen` | 用于 AMD/Intel 显卡，本机无可用核显，独显走专用驱动 |
| `VoodooI2C` 系列 | 触控板只做到基础可用，完整方案未落地（见 troubleshooting） |
| `NVMeFix` | 本机 NVMe 原生识别正常 |
| `CPUFriend` | `ProvideCurrentCpuInfo` 已够用 |
| `AirportItlwm` | 与 `itlwm` 二选一，选了后者（更稳定） |
| `USBInjectAll` | 已被 USBToolBox 方案替代 |

---

## 版本管理

kext 版本要在 `config.plist` 里显式声明 `MinKernel` / `MaxKernel` 吗？

**本机没有声明**（留空 = 全版本生效）。理由：这是单机专用配置，不存在多系统兼容需求。

如果是通用 EFI，建议声明，避免系统升级后加载不兼容的 kext 导致启动失败。

---

## 更新策略

| Kext | 更新频率 | 建议 |
|---|---|---|
| Lilu / VirtualSMC | 跟随系统更新 | 系统升级前一起更新 |
| AppleALC | 低 | 除非有音频问题，不动 |
| itlwm | 低 | 同上 |
| USBToolBox 系列 | 极低 | **不要动**，动了要重新做映射 |
| 第三方显卡驱动 | 跟系统 | 关键路径，见 `docs/nvrm.md` |

**原则：能跑就不要更新。** 黑苹果的更新收益远小于风险。
