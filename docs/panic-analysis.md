# 分析一个随机内核崩溃：`kalloc` 堆损坏

> 这是本项目的技术核心。一个看起来像「配置没调好」的问题，实际是第三方驱动的内存安全缺陷。

---

## 1. 现象

### 1.1 崩溃

开机后约 35–45 秒，约 35–45% 的概率触发 panic：

```text
panic(cpu 0 caller 0xffffff8000baa111): [kalloc.type.var6.16]:
  element modified after free (off:8, val:0x0000000000000002, sz:16, ptr:0xffffffa303c6ec50)
    8: 0x0000000000000002
Panicked task 0xffffffa303f2b9a0: 221 threads: pid 0: kernel_task
System uptime in nanoseconds: 40560285949 (~40.5s)
```

### 1.2 另一种表现

当堆检测器没有抓到时，不会 panic，而是**卡死**：

```
NVRM: boot guard: start 1 without a display so far
```

然后 OpenCore 的看门狗检测到系统无响应，强制重启。

**同一个缺陷的两种表现**：抓到内存损坏就 panic，抓不到就挂在初始化等待上。

---

## 2. 读调用栈

自下而上（从触发点到内核入口）：

```text
mach_kernel : _call_continuation
mach_kernel : _kalloc_ext                        ← 分配器
mach_kernel : _kern_os_malloc
com.nullmoth.NVRM : _nv_alloc_pages + 0x286      ← 驱动申请内存
com.nullmoth.NVRM : _osAllocPagesInternal + 0x46a
com.nullmoth.NVRM : _memdescAlloc + 0x1c3
com.nullmoth.NVRM : _gpuCreateRusdMemory_IMPL + 0xa0
com.nullmoth.NVRM : _memmgrStateInitLocked_IMPL + 0x56c
com.nullmoth.NVRM : _gpuStateInit_IMPL + 0xb8
com.nullmoth.NVRM : _gpumgrStateInitGpu + 0x2b
com.nullmoth.NVRM : _RmInitNvDevice + 0x11b
com.nullmoth.NVRM : _RmInitAdapter + 0xe34
com.nullmoth.NVRM : _rm_init_adapter + 0xe2
com.nullmoth.NVRM : __ZN4NVRM2goEj + 0x5f7
com.nullmoth.NVRM : __ZN4NVRM6autoGoEPvi + 0x357
mach_kernel : _call_continuation
```

**关键信息**：

1. 崩溃点在 `NVRM::autoGo` → `NVRM::go`，这是驱动在系统启动后自动初始化 GPU 的路径。
2. 一路走到 `RmInitAdapter`（RM = Resource Manager，NVIDIA 驱动的核心子系统）。
3. 在 `gpuStateInit` → `memmgrStateInit` → `gpuCreateRusdMemory` 这条**内存管理器初始化**链上申请内存。
4. 崩溃发生在 `nv_alloc_pages` 调用 `kern_os_malloc` 的**过程中**。

**注意第 4 点**：panic 不是在「写一个已释放的对象」时直接触发的，而是在**分配器内部**发现「某个 16 字节对象的元数据被改过」。

---

## 3. 什么叫 `element modified after free`

macOS 的 `kalloc` 是一个分区分配器（zone allocator）。开启 zone poisoning 后，对象释放时会把内存填成特定毒值，用于检测释放后写入。

```
1. 对象 A 被 free        → 内存填充毒值 0xDEADBEEF...
2. 某段代码仍持有 A 的指针
3. 该代码写入 A + 8 的位置，值 0x2
4. 分配器再次遍历这个 zone → 发现毒值被改 → panic
```

报告的字段含义：

| 字段 | 值 | 含义 |
|---|---|---|
| `off:8` | 8 | 被改写的位置在对象起始偏移 8 字节处 |
| `val:0x2` | 2 | 写入的值是 2 |
| `sz:16` | 16 | 对象大小 16 字节 |
| `ptr` | 0xffffffa303c6ec50 | 该对象的内核地址 |

**16 字节 + offset 8 + 值 2**，这个组合指向一个典型结构：**链表节点或引用计数对**。前 8 字节是指针（next/prev），后 8 字节是计数或类型标记，被写成了 2。

**推断**：驱动的某个内存描述符链表在并发初始化时，一个线程释放了节点，另一个线程仍在操作它。

---

## 4. 为什么是随机的

如果是确定性的逻辑错误，应该 100% 复现。实际概率 35–45%，说明是**竞态**。

GPU 初始化有两条并发的路径：

```
路径 A（主）: NVRM::go → rm_init_adapter → ... → memmgrStateInit
路径 B（GSP）: 固件加载 → GSP 就绪 → 通知主路径继续
```

驱动需要等 GSP 固件起来才能完成内存管理器初始化。这个「等待 - 通知」之间的时序，决定了谁先碰到那个被释放的节点。

**所以崩溃概率取决于启动时的时序抖动**——CPU 调度、固件加载速度、甚至环境温度都会影响。

这也解释了为什么「有时候能进系统」：不是修好了，是那次没撞上。

---

## 5. `nvrmsettle=10000` 为什么有效（但不根治）

这个启动参数给 GSP 固件的启动加 10 秒延迟。

**它做了什么**：改变了路径 A 和路径 B 的相对时序。主路径在等待之前多睡了 10 秒，GSP 有足够时间完成初始化并稳定下来，竞态窗口被错开。

**它没做什么**：那个 use-after-free 依然存在。内存管理器的代码没有变，只是在当前时序下不容易撞上。

| 措施 | 触发概率 | 性质 |
|---|---|---|
| 无 | 35–45% | — |
| `nvrmsettle=10000` | 显著降低（实测个位数百分比） | **缓解** |
| 修复驱动 | 0 | 需要上游改代码 |

**这个区分很关键**：缓解意味着「仍然可能崩」，文档必须这么写，否则会误导别人以为问题解决了。

---

## 6. 一个被忽略的疏漏

深入看驱动的安装脚本，发现它确实做了「对特定架构自动加延迟」这件事：

```bash
# 简化后的逻辑
if echo "$DEVICE_ID" | grep -qE '1[eEfF]??|21[89a-fA-F]?'; then
    append_boot_arg "nvrmsettle=10000"
fi
```

这段正则匹配的是：

- `1[eEfF]??` → `1e`、`1f`、`1E` 开头（Turing 的 `1e00`–`1fff` 段）
- `21[89a-fA-F]?` → `2180`–`21af`（Turing 的 TU10x/TU11x）

**Ada Lovelace 的设备 ID 是 `28a0`，不在匹配范围内。**

结果：这台机器**从来没有被自动应用过任何缓解措施**，一直是裸跑。崩溃率高不是巧合，是因为压根没打上这个补丁。

### 修正

手动在 `boot-args` 里加上：

```
nvrmsettle=10000
```

或者改脚本的正则，把 Ada 段加进去：

```bash
# 增加 2[5-9][0-9a-f]{2} 覆盖 Ada（25xx-29xx）
if echo "$DEVICE_ID" | grep -qE '1[eEfF]??|21[89a-fA-F]?|2[5-9][0-9a-f]{2}'; then
    append_boot_arg "nvrmsettle=10000"
fi
```

**这是一个可以提给上游的具体 bug**：自动化脚本的设备 ID 匹配表没有跟上新架构。

---

## 7. 完整的启动参数（本机实际使用）

```
-vi2c-force-polling        # I2C 轮询模式，解决输入设备初始化
amfi_get_out_of_my_way=0x1 # 允许加载未签名 kext
amfi=0x80                  # AMFI 宽松模式
ipc_control_port_options=0 # 解决部分 IPC 兼容问题
nvfb=1                     # 启用 NVIDIA framebuffer
nvaccel=1                  # 启用 GPU 加速
nvfbheads=4                # 4 个显示头
-nvkmsnosmooth             # 关闭 KMS 平滑，减少撕裂
gfxrst=4                   # 图形复位策略
-ibtcompatbeta             # IBT 兼容
revpatch=sbvmm             # 绕过某些机型检查
-v keepsyms=1              # 详细日志 + 保留符号
nvrmsettle=10000           # GPU 初始化延迟 10 秒
```

**前四条是基础兼容性，中间五条是显示驱动，最后一条是崩溃缓解。** 分清楚哪些是「必须」哪些是「缓解」，排障时能少走很多弯路。

---

## 8. 如何复现与取证

### 8.1 提高复现概率

- **冷启动**（完全断电再开），不是重启。重启时 GPU 可能保持已初始化状态，不复现。
- 连续多次冷启动，统计崩溃次数。
- 关闭快速启动（BIOS 里）。

### 8.2 抓 panic 日志

```bash
# 崩溃日志位置
ls -lt /Library/Logs/DiagnosticReports/*.panic | head

# 或者从 NVRAM 里取（下次启动后仍在）
nvram -p | grep -i panic
```

### 8.3 确认是同一个问题

看三条：

1. panic 类型是 `element modified after free`
2. 调用栈里有 `NVRM::autoGo` 或 `NVRM::go`
3. uptime 在 30–50 秒之间

三条都符合，就是同一个缺陷。

---

## 9. 如果要在上游报这个 bug

一个有效的 bug 报告需要包含：

| 项 | 内容 |
|---|---|
| 硬件 | 机型、CPU、GPU（含 PCI 设备 ID `10DE:28A0`） |
| 系统 | macOS 26.7.1 (25G241)，Darwin 25.6.0 |
| 驱动 | 版本号 + build 号 |
| 现象 | 冷启动 40 秒随机 panic，概率约 40% |
| 证据 | 完整 backtrace（含内核符号） |
| 已排除 | 内存、SMBIOS、ACPI 均已交叉验证 |
| 关键线索 | `nvrmsettle` 能缓解 → 指向初始化时序竞态 |
| 脚本缺陷 | 设备 ID 匹配表未覆盖 Ada（`28a0`） |

**最有价值的是最后两条**——它们把「随机崩溃」缩小到了「GSP 初始化时序 + 特定架构未打补丁」这个具体范围。

---

## 10. 从这个案例学到什么

1. **`element modified after free` 是一个明确的信号**，它说明有代码在释放后写入。不要往「内存条坏了」方向查。
2. **随机性 = 竞态**。确定性错误和概率性错误的排查方法完全不同。
3. **缓解不等于修复**。文档里必须区分，否则会误导。
4. **自动化工具有盲区**。脚本的设备 ID 表、正则、默认值，都可能在你的硬件上不成立。**要读脚本，不要只运行脚本。**
5. **backtrace 是唯一可信的线索**。所有「我觉得可能是」的假设，最后都要靠它验证。
