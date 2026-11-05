# 排查手册

症状 → 可能原因 → 验证方法。按「开机到进入系统」的时间顺序排列。

---

## 一、OpenCore 阶段

### 1.1 开机直接进 Windows，看不到 OpenCore 菜单

| 可能原因 | 验证方法 | 解决 |
|---|---|---|
| BIOS 启动顺序 | 进 BIOS 看 Boot Order | 把 OpenCore 排第一 |
| 残留的 Windows Boot Manager | `bcdedit /enum firmware` | 删除重复项 |
| 快速启动未关 | BIOS 里看 Fast Boot | 关闭 |
| 安全启动未关 | BIOS 里看 Secure Boot | 关闭 |

**这是本项目最耗时的问题**（详见 `journey.md`）。日志里一切正常，问题在固件层。

### 1.2 看得到 OpenCore 菜单，但没有 macOS 条目

| 可能原因 | 验证方法 | 解决 |
|---|---|---|
| 分区格式不被识别 | 用磁盘工具看分区类型 | APFS，且未加密 |
| `ScanPolicy` 太严 | 临时设为 `0` | 全部扫描 |
| 驱动未加载 | 看 `Drivers/` 目录 | 补 `HfsPlus.efi` 或 `OpenApfs.efi` |
| 硬盘是 RAID / RST 模式 | BIOS 里看 SATA 模式 | 改成 AHCI |

### 1.3 选中条目后黑屏重启

| 可能原因 | 验证方法 | 解决 |
|---|---|---|
| SSDT 冲突 | 逐个禁用 | 找出问题表 |
| Quirks 不匹配 | 对照固件版本 | 调整 Booter Quirks |
| 内存映射问题 | `DevirtualiseMmio` 相关 | 试开/试关 |
| 显卡初始化失败 | 加 `-v` 看卡在哪 | 见下节 |

---

## 二、安装阶段

### 2.1 卡在苹果 logo，进度条不动

| 可能原因 | 验证方法 | 解决 |
|---|---|---|
| **USB 映射错误** | 打开 `Misc → Debug` 看日志 | **重做 USB 映射**（最常见） |
| 存储驱动缺失 | `-v` 看是否卡在 IONVMe | 补 NVMe 补丁 |
| 安装介质损坏 | 校验 SHA256 | 重做 U 盘 |
| 内存不足 | 至少 8 GB | — |

**「卡在 1/5 处不动」几乎总是 USB 映射问题。**

### 2.2 卡在 `Waiting for root device`

| 可能原因 | 验证方法 | 解决 |
|---|---|---|
| USB 端口类型错误 | 换一个 USB 口试 | 用 USB 2.0 口 |
| 存储控制器未识别 | `-v` 看设备枚举 | 加对应 kext |
| 安装介质在 USB 3.0 口上 | — | 换 2.0 口 |

### 2.3 `-v` 输出大量 `AppleUSBHostPort::...` 后卡住

USB 端口映射问题。用 USBToolBox 重新生成。

### 2.4 安装器起来了，但看不到硬盘

| 可能原因 | 验证方法 | 解决 |
|---|---|---|
| NVMe 未被识别 | `-v` 看设备树 | 加 NVMe 补丁 |
| 分区表类型 | 磁盘工具看 | GPT，不是 MBR |
| 硬盘需要专属 kext | 查型号 | 少数型号需额外驱动 |

---

## 三、首次启动阶段

### 3.1 进了系统，但键鼠完全没反应

| 可能原因 | 验证方法 | 解决 |
|---|---|---|
| **USB 端口全被标为 Internal** | 看 USB 映射工具 | **改成 Keyboard / Mouse** |
| I2C 触控板未驱动 | `ioreg` 看设备树 | 装 VoodooI2C 系列 |
| USB 注入冲突 | 禁用 `UTBDefault` | 换用 `USBInjectAll` 测试 |

**这是本项目第二个卡点。** 自动生成的映射把所有端口标成内部接口，而 macOS 对内部端口不加载 HID 驱动。

### 3.2 触控板能移动，但不能点击

| 可能原因 | 验证方法 | 解决 |
|---|---|---|
| 未加载 `VoodooI2CHID` | `kextstat` | 补装 |
| GPIO 中断未配置 | 看 ACPI 补丁 | 加 `SSDT-GPIO` |
| 按键映射错误 | 偏好设置里试 | 改 `VoodooI2C` 配置 |

**本机只做到「移动可用」，多点手势不完整。** 这是已知的未解决问题。

### 3.3 画面颠倒或镜像

见 [`display-rotation.md`](display-rotation.md)。

### 3.4 分辨率不对，只有 1024×768

| 可能原因 | 验证方法 | 解决 |
|---|---|---|
| framebuffer 未正确初始化 | `nvfb=1` 是否有 | 补参数 |
| EDID 读取失败 | 看系统报告 | 注入 EDID |
| 驱动未加载 | `kextstat` | 见 `nvrm.md` |

---

## 四、运行阶段

### 4.1 开机约 40 秒随机崩溃

见 [`panic-analysis.md`](panic-analysis.md)。**这是驱动缺陷，不是配置问题。**

快速判断：panic 类型是 `element modified after free`，且调用栈含 `NVRM`。

### 4.2 睡眠后唤不醒

| 可能原因 | 验证方法 | 解决 |
|---|---|---|
| 显卡驱动不支持唤醒 | 关掉睡眠测试 | 禁用睡眠 |
| USB 唤醒配置 | `pmset -g` | 关闭 USB 唤醒 |
| ACPI 状态问题 | 看 `pmset -g log` | 调整 SSDT |

本机未解决，直接禁用睡眠：

```bash
sudo pmset -a sleep 0
sudo pmset -a hibernatemode 0
sudo pmset -a disablesleep 1
```

### 4.3 没有声音

| 可能原因 | 验证方法 | 解决 |
|---|---|---|
| layout-id 不对 | 试多个值 | 逐个测 |
| AppleALC 未加载 | `kextstat` | 补装 |
| 设备路径不匹配 | `ioreg` | 注入正确的 `layout-id` |

Conexant 声卡的 layout-id 需要用枚举法试，没有通用值。

### 4.4 Wi-Fi 能连但速度慢

`itlwm` 不是原生 Airport 驱动，性能有差距。这是方案固有限制，不是配置问题。

如果要原生体验，只能换博通网卡（需要硬件改造）。

### 4.5 蓝牙能识别但连不上

| 可能原因 | 验证方法 | 解决 |
|---|---|---|
| 固件未加载 | `log show` 搜 bluetooth | 检查 `IntelBluetoothFirmware` |
| USB 端口被占用 | 看 USB 设备树 | 释放端口 |
| 与 2.4G 干扰 | 换位置试 | — |

---

## 五、通用排查方法

### 5.1 打开完整日志（最重要的一步）

```xml
<key>Misc</key><dict>
  <key>Debug</key><dict>
    <key>Target</key><integer>67</integer>
    <key>AppleDebug</key><true/>
  </dict>
</dict>
```

日志写到 EFI 分区的 `opencore-*.txt`。

**90% 的「玄学问题」在日志里有明确提示。** 前面两天的时间基本浪费在没开日志上。

### 5.2 二分法隔离

改多个参数是排障大忌。正确做法：

```
1. 备份当前可启动的 EFI
2. 只改一个参数
3. 测
4. 不行就回滚，换下一个
```

慢，但每次都有确定的结论。

### 5.3 确认「配置问题」还是「驱动缺陷」

| 特征 | 配置问题 | 驱动缺陷 |
|---|---|---|
| 复现率 | 100% | 随机 |
| 改配置能否解决 | 能 | 不能 |
| 日志指向 | 明确的检查失败 | 内存错误 / 超时 |
| 影响范围 | 特定功能 | 整个系统 |

**判断错了会浪费大量时间。** 如果是驱动缺陷，改配置永远改不好。

### 5.4 保留可启动的 EFI 备份

命名规范：

```
EFI-1026-bootable-with-sound    ← 能启动、声音正常
EFI-1027-no-input               ← 能启动、输入有问题
EFI-1029-final                   ← 最终版本
```

**不要用 `EFI-new`、`EFI-backup` 这种名字。** 一周后你分不清哪个是能用的。

### 5.5 用恢复模式验证

进不去系统时，先进恢复模式（OpenCore 菜单里的 Recovery）：

- 能进恢复模式 → 引导链没问题，问题在驱动或系统
- 进不去恢复模式 → 引导链本身有问题（EFI / ACPI / Quirks）

这个二分能把问题范围直接砍一半。

---

## 六、速查表

| 症状 | 第一嫌疑 |
|---|---|
| 卡进度条 1/5 | USB 映射 |
| `Waiting for root device` | USB 端口类型 |
| 无 macOS 条目 | `ScanPolicy` / 驱动 |
| 选了进 Windows | BIOS 启动顺序 |
| 键鼠全无 | USB 端口标为 Internal |
| 分辨率低 | framebuffer 参数 |
| 画面颠倒 | 需要改 kext |
| 40 秒随机崩 | 驱动 use-after-free |
| 睡不醒 | 驱动 + ACPI |
| 无声音 | layout-id |
| Wi-Fi 慢 | itlwm 固有限制 |
