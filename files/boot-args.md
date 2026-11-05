# 启动参数

每一项都是为解决一个具体问题加的。附带「为什么需要」和「去掉会怎样」。

---

## 完整参数

```
-v keepsyms=1 debug=0x100 npci=0x2000
-vi2c-force-polling
amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0
nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth gfxrst=4
-ibtcompatbeta revpatch=sbvmm
nvrmsettle=10000
```

---

## 逐条说明

### 调试类

| 参数 | 作用 | 去掉会怎样 |
|---|---|---|
| `-v` | 详细模式，显示内核日志 | 卡住时只有苹果 logo，无法定位 |
| `keepsyms=1` | 保留内核符号 | panic 时全是地址，看不到函数名 |
| `debug=0x100` | panic 后停留在屏幕 | 自动重启，看不到崩溃信息 |

**`debug=0x100` 最容易被忽略但最有价值。** 没有它，崩溃信息一闪而过，等于没有。

系统稳定后可以去掉 `-v`，但建议保留 `keepsyms=1` 和 `debug=0x100`。

### PCI / 内存

| 参数 | 作用 |
|---|---|
| `npci=0x2000` | 解决 PCI 配置空间扫描冲突 |

不加的表现：卡在 `PCI configuration begin` 或相关位置。

### 输入设备

| 参数 | 作用 | 代价 |
|---|---|---|
| `-vi2c-force-polling` | I2C 走轮询而非中断 | CPU 占用略高（约 1–2%） |

**这是本机能进系统的关键参数之一。** 不加的话，触控板初始化失败会导致整个输入子系统挂起，表现为「键鼠全无」。

### AMFI / 签名

| 参数 | 作用 |
|---|---|
| `amfi_get_out_of_my_way=0x1` | 允许加载未签名 kext |
| `amfi=0x80` | AMFI 宽松模式 |

两个同时用是冗余设计——某些系统版本会忽略其中一个。

配合 `AMFIPass.kext` 使用，三重保险。

**代价**：系统完整性保护被实质性削弱。这是加载第三方驱动的必要成本。

### 图形驱动

| 参数 | 作用 | 建议值 |
|---|---|---|
| `nvfb=1` | 启用 framebuffer（显示输出） | 必开 |
| `nvaccel=1` | 启用 GPU 加速 | 必开 |
| `nvfbheads=4` | 显示头数量 | 笔记本通常 4 |
| `-nvkmsnosmooth` | 关闭 KMS 平滑，减少撕裂 | 建议 |
| `gfxrst=4` | 图形复位策略 | 4 最稳 |

`nvfbheads` 不是越大越好。设太大浪费显存并可能导致初始化失败，设太小则外接显示器不亮。

### 兼容性

| 参数 | 作用 |
|---|---|
| `ipc_control_port_options=0` | 解决部分 IPC 兼容问题 |
| `-ibtcompatbeta` | 允许在测试版系统上加载 IBT |
| `revpatch=sbvmm` | 绕过某些机型检查 |

### 崩溃缓解

| 参数 | 作用 |
|---|---|
| `nvrmsettle=10000` | GPU 初始化前延迟 10 秒 |

**这是缓解不是修复。** 详见 `docs/panic-analysis.md`。

代价：开机会多等 10 秒。如果对启动速度敏感，可以试 5000，但触发概率会上升。

---

## 按问题归类

```
看到崩溃信息       -v keepsyms=1 debug=0x100
PCI 卡住          npci=0x2000
键鼠全无           -vi2c-force-polling
kext 加载不了      amfi_get_out_of_my_way=0x1 amfi=0x80
显示不亮           nvfb=1 nvfbheads=4
画面卡顿           nvaccel=1 -nvkmsnosmooth
画面撕裂           gfxrst=4
随机崩溃           nvrmsettle=10000
```

---

## 怎么改

三种方式，优先级从高到低：

1. **`config.plist` 的 `NVRAM → Add → boot-args`** — 推荐，跟着 EFI 走
2. **OpenCore 引导菜单里临时输入** — 调试用，重启即失效
3. **`sudo nvram boot-args="..."`** — 在系统里改，但可能被 config 覆盖

**调试时用第 2 种**：不改配置、立即生效、重启还原。确认有效再写进 config。

---

## 注意

- 参数之间用**空格**分隔，不要用逗号
- 参数里不要有中文标点
- 整串写在 `<string>` 里，不要换行
- 改完必须重启（`nvram` 写入后也需要）
