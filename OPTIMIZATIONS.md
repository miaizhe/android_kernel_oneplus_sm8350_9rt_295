# OnePlus 9RT (MT2110 / OP5154L1) 内核优化记录

- 分支：`perf/eevdf-optimized`
- 基线：`oneplus/sm8350_b_16.0.0_oneplus9rt`
- 内核：5.4.295 QGKI，SM8350 (lahaina)，Android 17 / ColorOS 17 ROM
- 平台：8 核（4×A55 300MHz–1.8GHz / 3×A78 710MHz–2.42GHz / X1 844MHz–2.84GHz），~11.5GB RAM

本文只记录**已经在本机实测验证过**的结论，推测与未完成项单独列在最后一节。

---

## 1. EEVDF 调度器迁移

从 `20260917-R10` 迁入，对齐 Linux 6.6 的 EEVDF。

**明确排除 Linux 7.2 的部分**：7.2 补丁 `6e3ebfbfc4` 只动了 `kernel/sched/fair.c`（+22/-13），且已被 `b762db1bcf`（+13/-22）revert。R10 分支 tip 的 `fair.c` 与 6.6 对齐后的状态 `550683ff75` **字节一致**，因此不含 7.2 改动。

迁移文件（9 个，全部为调度器相关，**未触碰 CPU 调频驱动**）：

```
kernel/sched/fair.c              +328/-197
kernel/sched/core.c               +16/-1
kernel/sched/sched.h               +9/0
kernel/sched/features.h            +4/0
kernel/sched/debug.c               +1/0
include/linux/sched.h              +6/-1
include/uapi/linux/sched.h         +3/-1
include/uapi/linux/sched/types.h   +4/0
arch/arm64/configs/9rt_defconfig   +2/-1
```

**验证**：`/proc/kallsyms` 含 `__pick_eevdf`、`avg_vruntime`；`/proc/sys/kernel/sched_base_slice_ns` 存在；`cat /proc/version` 的版本串含对应 commit。

---

## 2. UX 不公平调度（EEVDF 原生实现）

**背景**：ColorOS 在 `p->ux_state` 中标记前台任务——

```c
#define SCHED_ASSIST_UX_MASK (SA_TYPE_LIGHT|SA_TYPE_HEAVY|SA_TYPE_ANIMATOR|SA_TYPE_LISTPICK)
```

LIGHT（轻前台）/ HEAVY（应用启动）/ ANIMATOR（动画）/ LISTPICK（列表滑动），由 userspace 经 binder 事务下发。但 EEVDF 迁移后**没有任何代码消费这个信号**：所有普通任务的 slice 都是同一个 750µs，唤醒的前台线程对后台工作没有调度优势。

**做法**：在 `entity_slice()` 中对 ux 任务返回更短的 slice。

```
ux 任务 → entity_slice() 返回 base/divisor
        → se->deadline 更早
        → pick_eevdf() 在唤醒时优先选中
        → 前台抢在后台之前拿到 CPU
```

这是 EEVDF 自身的语义（slice → virtual deadline → 选中顺序），**没有重新引入任何 CFS 时代的 vruntime 花招**，也没动 vendor 那套 `place_entity_adjust_ux_task` 之类的钩子。

**运行时调参**：

```bash
su -c 'cat /proc/sys/kernel/sched_ux_slice_divisor'    # 默认 2
su -c 'echo 1 > /proc/sys/kernel/sched_ux_slice_divisor'   # 关闭
su -c 'echo 4 > /proc/sys/kernel/sched_ux_slice_divisor'   # 187.5µs，更激进
```

| divisor | ux slice | 说明 |
| --- | --- | --- |
| 1 | 750.0 µs | 关闭 |
| 2 | 375.0 µs | 默认 |
| 3 | 250.0 µs | |
| 4 | 187.5 µs | |
| ≥8 | 100.0 µs | 有 100µs 下限兜底 |

**安全边界**：`test_task_ux()` 内部有反滥用检查——`total_exec > ux_task_exec_limit()` 时返回 false，长跑任务会自动掉出 ux 资格，**后台不会被饿死**。

**已知副作用**：调度切点变密（750µs → 375µs），理论上略增功耗。这是该改动最可能的负面效果。

---

## 3. 修正 latency_nice → slice 映射的非单调

原实现锚定在固定 125000ns，0 处靠 `entity_slice()` 特判：

```c
/* 旧 */
return 125000 + (latency_nice + 20) * 73076;   // 0 处算出 1586520 ≠ 750000
// entity_slice(): if (p->latency_nice == 0) return sysctl_sched_base_slice;
```

结果单调性在 0 处断裂：**`latency_nice = -1` 得到 1513444ns，反而是 `0`（750000ns）的两倍** —— 本意「-1 更紧急」，实际更不紧急。

新实现锚定 `sysctl_sched_base_slice`，严格单调（实测数值）：

| latency_nice | slice |
| --- | --- |
| −20 | 93.8 µs (base/8) |
| −10 | 421.9 µs |
| −1 | 717.2 µs |
| 0 | 750.0 µs (base) |
| 1 | 862.5 µs |
| 10 | 1912.5 µs |
| 19 | 3000.0 µs (base×4) |

---

## 4. 其它已合入项

| 提交 | 内容 |
| --- | --- |
| `26865ba866` | PR#1：CAP_CHECKPOINT_RESTORE / clone3 set_tid 系列上游补丁（10 个） |
| （同线） | PR#2：close_range |
| `c02aafd87f` | binder 热路径日志清理与逐字符 sprintf 移除 |
| `5632ea2073` | 9rt_defconfig 行尾统一为 LF |
| `b7f9fd0714` | uxmem_opt：96MB ux pool + 池页计入 MemFree/MemAvailable |
| `b832d7ee03` | 修 uxmem 重复定义（锚点同时匹配 fill 与 refill 尾部） |
| `85c7ab1dca` | EEVDF 迁移 |
| `2e8325bfa4` | CI 接入 ccache：冷编 35.9 分 → 热编 13.0 分 |
| `15354021cb` | 暴露 `sched_base_slice_ns` |
| `aaf8d3fac` | 本文档第 2、3 节 |

---

## 5. 验证方法

**帧统计**（测抖动）：

```bash
adb shell am start -n com.android.settings/.Settings
adb shell dumpsys gfxinfo com.android.settings reset
# 执行可复现动作，例如 10 次 input swipe
adb shell dumpsys gfxinfo com.android.settings | grep -E "Total frames|Janky frames|50th|90th"
```

注意：`dumpsys SurfaceFlinger --latency <layer>` 在 Android 13+ **已废弃**，只返回刷新周期（`16666667`），不要用它。`com.android.launcher` 不走 HWUI 统计路径（`Total frames rendered: 0`），要用 Settings 这类应用。

**确认跑的是哪个内核**：`cat /proc/version`，版本串里的 `g<hex>` 就是 commit 前缀。

**确认 tunable 生效**：

```bash
su -c 'cat /proc/sys/kernel/sched_base_slice_ns'      # 750000
su -c 'cat /proc/sys/kernel/sched_ux_slice_divisor'   # 2
```

---

## 6. 与内核无关的现象（避免误判）

**频率上限由 ColorOS userspace 控制，不是内核。**

实测（`policy4` = A78 / `policy7` = X1，对比 `cpuinfo_max_freq`）：

| 状态 | policy4 | policy7 | 温控冷却设备 |
| --- | --- | --- | --- |
| 高性能模式关闭（空闲） | 1996800 (82.6%) | 2150400 (75.7%) | `cur=0` |
| 高性能模式开启（空闲） | 2227200 (92.1%) | 2496000 (87.9%) | `cur=0` |
| 8 线程满载 | 1670400 (69.1%) | 1785600 (62.8%) | `cur=0` |

开启高性能模式还会把 `scaling_min_freq` 抬高，A55 小核保底 **499200 → 1209600（+142%）**。

关键点：**降频发生了，但内核 thermal 框架全程 `cur=0`，dmesg 也没有 thermal 日志** —— 是 ColorOS 的 `thermal-engine` / `oplus.performance.hal` 直接写 `scaling_max_freq`。因此调内核 thermal 无法解决，也不该归咎于内核改动。

**开机 CPU 饱和**：空载与满载两种状态下都测得开机阶段 CPU 接近 100%，主导进程是 `system_server`、`systemui`、`launcher` 以及 OPLUS 的 `defrag_wrapper`、`deepthinker`、`fallocate`（创建 6GB nandswap 文件）。与基线 R10 对比，**基线爬升更快、峰值更高** —— 该饱和是 Android + OPLUS 开机行为，非本分支引入。

---

## 7. 未完成 / 待验证（**不要当成已完成**）

- **UX 不公平调度的 A/B 定量验证尚未跑完**。测量方法已就绪（第 5 节），但 `divisor=1` vs `2` 的 jank% 对照数据还没有。在拿到数据之前，**不要声称这个改动改善了卡顿**。
- **vendor 模块加载失败未排查**。早前一次 dmesg 采样中看到 3183 行 `disagrees about version of symbol`，`module_layout` 出现 280 次，涉及 `haptic` / `bolero_cdc_dlkm` / `wcd9xxx_dlkm` / `q6_dlkm` / `swr_dlkm` / `mbhc_dlkm`。dmesg 已回绕（现为 0 行），**需重启复现**。这首先是功能问题（震动、音频 codec 可能未加载），其次才是开机开销。
- **binder 日志刷屏未处理**：`oplus_binder_stats binder_stats_driver_ioc` 的 `pr_info` 开机出现 396 次。
- **续航优化已列选项但全部未实施**。内核功耗基础本身已较好：`NO_HZ_IDLE`、`ENERGY_MODEL`(EAS)、`CPU_IDLE_GOV_MENU`、`schedutil`、`RCU_FAST_NO_HZ` + `RCU_NOCB_CPU`、`LTO_CLANG` + `CFI_CLANG` 均已启用。zram 使用 `zstd`，swappiness=60。
- **开机时的一次 WARN**：`kernel/sched/fair.c` 的 `place_entity()` 中 mainline 自带的 `WARN_ON_ONCE(!load)`（`load = cfs_rq->avg_load` 在 PID 1 首次入队时为 0），触发一次后代码自行恢复（`load = 1`），判断为无害。

---

## 8. 已知的一处结构性风险（未证实为缺陷）

本内核同时编入了**三套调度器**（kallsyms 实测）：

- EEVDF（`__pick_eevdf`、`avg_vruntime`）
- OPLUS sched_assist（`sched_assist_pick_next_entity`、`oplus_check_preempt_wakeup_in_list`、`should_ux_task_skip_cpu`、`sched_assist_task_misfit` 等）
- 高通 WALT（110 个符号）

其中 `sched_assist_pick_next_entity()` 挂在 `pick_next_entity()` 中，能覆盖 EEVDF 已选出的实体：

```c
se = pick_eevdf(cfs_rq);
#ifdef CONFIG_OPLUS_FEATURE_AUDIO_OPT
    if (sched_assist_pick_next_entity(cfs_rq, &se))
        goto oplus_pick;          /* vendor 覆盖 */
#endif
```

不过实测其实现**范围很窄**，只在唤醒伙伴被标记为 `im_small`（即时消息/音频场景）时才返回 true：

```c
if (cfs_rq->next && oplus_entity_is_task(cfs_rq->next)) {
    struct task_struct *tsk = task_of(cfs_rq->next);
    if (tsk->oplus_task_info.im_small) { *se = cfs_rq->next; return true; }
}
return false;
```

**结论**：这是一个值得留意的结构风险点（`CONFIG_OPLUS_FEATURE_SCHED_ASSIST=y`、`CONFIG_OPLUS_FEATURE_AUDIO_OPT=y` 均已开启，vendor 代码当初按 CFS 语义编写，而它钩住的 `pick_next_entity()`/`check_preempt_wakeup()` 正是 EEVDF 重写过的函数），但**没有实测证据表明它造成缺陷**。若要验证，需构建一版关闭 `CONFIG_OPLUS_FEATURE_AUDIO_OPT` 做 A/B。

vendor 源码不在本仓库内：`include/linux/sched_assist` 是软链接，指向 `../../../../vendor/oplus/kernel/oplus_performance/sched_assist/`，该 `vendor/` 树由 CI 从 `JackA1ltman/android_kernel_modules_and_devicetree_oneplus_sm8350`（分支 `oneplus/sm8350_b_16.0.0_oneplus9rt`）检出后 `mv` 到 workspace 根，正好补上这 4 层软链接。

---

## 9. 回滚

```bash
fastboot flash boot_a <boot.img>
```

各阶段 boot 镜像备份保存在本地，未入库。
