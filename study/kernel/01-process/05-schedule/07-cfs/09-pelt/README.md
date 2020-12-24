https://lwn.net/Articles/639543/
81907478c

# 1 PELT 3.8@2012 per-entity load-tracking
-------

| 时间  | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:------:|:---:|
| 2012/8/23 | [per-entity load-tracking](https://lore.kernel.org/patchwork/cover/322242) | per-task 的负载跟踪首次引入内核 | V1 ☑3.8 | [LWN](https://lwn.net/Articles/531853), [PatchWork](https://lore.kernel.org/patchwork/cover/322242), [lkml](https://lkml.org/lkml/2012/8/23/267) |

第一个版本的 PELT 在 3.8 的时候合入主线.



```cpp
e9c84cb8d5f1 sched: Describe CFS load-balancer
f4e26b120b9d sched: Introduce temporary FAIR_GROUP_SCHED dependency for load-tracking
5b51f2f80b3b sched: Make __update_entity_runnable_avg() fast
f269ae0469fc sched: Update_cfs_shares at period edge
48a1675323fa sched: Refactor update_shares_cpu() -> update_blocked_avgs()
82958366cfea sched: Replace update_shares weight distribution with per-entity computation
f1b17280efbd sched: Maintain runnable averages across throttled periods
bb17f65571e9 sched: Normalize tg load contributions against runnable time
8165e145ceb6 sched: Compute load contribution by a group entity
c566e8e9e44b sched: Aggregate total task_group load
aff3e4988444 sched: Account for blocked load waking back up
0a74bef8bed1 sched: Add an rq migration call-back to sched_class
9ee474f55664 sched: Maintain the load contribution of blocked entities
2dac754e10a5 sched: Aggregate load contributed by task entities on parenting cfs_rq
18bf2805d9b3 sched: Maintain per-rq runnable averages
9d85f21c94f7 sched: Track the runnable average on a per-task entity basis
```
当时整个 sched_avg 结构体如下所示:

```cpp
struct sched_avg {
       /*
        * These sums represent an infinite geometric series and so are bound
        * above by 1024/(1-y).  Thus we only need a u32 to store them for for all
        * choices of y < 1-2^(-32)*1024.
        */
       u32 runnable_avg_sum, runnable_avg_period;
       u64 last_runnable_update;
       unsigned long load_avg_contrib;
};
```

| 字段 | 描述 |
|:-------:|:------:|
| runnable_avg_sum | 调度实体 sched_entity 在就绪队列上(on_rq) 累计负载 |
| runnable_avg_period | 调度实体 sched_entity 自创建至今如果持续运行, 所应该达到的累计负载 |
| last_runnable_update | 上次更新 sched_avg 的时间 |
| load_avg_contrib | 进程的负载贡献, running_avg_sum * load_weight / avg_period, 参见 [\_\_update_entity_load_avg_contrib](https://elixir.bootlin.com/linux/v3.8/source/kernel/sched/fair.c#L1402) |


```cpp
# https://elixir.bootlin.com/linux/v3.8/source/kernel/sched/fair.c#L1402
static inline void __update_task_entity_contrib(struct sched_entity *se)
{
    u32 contrib;

    /* avoid overflowing a 32-bit type w/ SCHED_LOAD_SCALE */
    contrib = se->avg.runnable_avg_sum * scale_load_down(se->load.weight);
    contrib /= (se->avg.runnable_avg_period + 1);
    se->avg.load_avg_contrib = scale_load(contrib);
}
```

cfs_rq 上记录的负载信息, 如下所示:

```cpp
# https://elixir.bootlin.com/linux/v3.8/source/kernel/sched/sched.h#L227
#ifdef CONFIG_SMP
/*
 * Load-tracking only depends on SMP, FAIR_GROUP_SCHED dependency below may be
 * removed when useful for applications beyond shares distribution (e.g.
 * load-balance).
 */
#ifdef CONFIG_FAIR_GROUP_SCHED
    /*
     * CFS Load tracking
     * Under CFS, load is tracked on a per-entity basis and aggregated up.
     * This allows for the description of both thread and group usage (in
     * the FAIR_GROUP_SCHED case).
     */
    u64 runnable_load_avg, blocked_load_avg;
    atomic64_t decay_counter, removed_load;
    u64 last_decay;
#endif /* CONFIG_FAIR_GROUP_SCHED */
/* These always depend on CONFIG_FAIR_GROUP_SCHED */
#ifdef CONFIG_FAIR_GROUP_SCHED
    u32 tg_runnable_contrib;
    u64 tg_load_contrib;
#endif /* CONFIG_FAIR_GROUP_SCHED */

    /*
     *   h_load = weight * f(tg)
     *
     * Where f(tg) is the recursive weight fraction assigned to
     * this group.
     */
    unsigned long h_load;
#endif /* CONFIG_SMP */
```

# 2 PELT 4.1@2015 consolidation of CPU capacity and usage
-------


| 时间  | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:------:|:---:|
| 2012/8/23 | [consolidation of CPU capacity and usage](https://lore.kernel.org/patchwork/cover/322242/) | CPU 调频会导致 capacity 的变化, 在支持 DVFS 的系统还用最大 capacity 计算负载是不合理的, 因此 PELT 感知 capacity 的变化  | V1 ☑ 4.1 | [PatchWork](https://lore.kernel.org/patchwork/cover/545867), [lkml](https://lkml.org/lkml/2015/2/27/309) |


这组补丁为 PELT 引入了 `frequency scale invariance` 的特性, 当前的系统都是支持 DVFS 的, CPU 处于不同的频率对应的 capacity 肯定是不同的, 特别的对于 big.LITTLE 等架构, 不同 cluster 的能效也是不同的. 因此如果两个相同的进程在不同的频率运行相同的时间, 那么其利用率理论上应该是不一样的.


```cpp
dfbca41f3479 sched: Optimize freq invariant accounting
1aaf90a4b88a sched: Move CFS tasks to CPUs with higher capacity
caff37ef96ea sched: Add SD_PREFER_SIBLING for SMT level
dc7ff76eadb4 sched: Remove unused struct sched_group_capacity::capacity_orig
ea67821b9a3e sched: Replace capacity_factor by usage
8bb5b00c2f90 sched: Calculate CPU's usage statistic and put it into struct sg_lb_stats::group_usage
ca6d75e6908e sched: Add struct rq::cpu_capacity_orig
b5b4860d1d61 sched: Make scale_rt invariant with frequency
0c1dc6b27dac sched: Make sched entity usage tracking scale-invariant
a8faa8f55d48 sched: Remove frequency scaling from cpu_capacity
21f4486630b0 sched: Track group sched_entity usage contributions
36ee28e45df5 sched: Add sched_avg::utilization_avg_contrib
```



| 字段 | 描述 |
|:-------:|:------:|
| runnable_avg_sum | 调度实体 sched_entity 在就绪队列上(on_rq) 累计负载 |
| running_avg_sum | 调度实体 sched_entity 实际在 CPU 上运行(on_cpu) 的累计负载 |
| avg_period  | 调度实体 sched_entity 自创建至今如果持续运行, 所应该达到的累计负载, 等同于原来的 runnable_avg_period, 只是由于这个负载其实跟进程是不是 runnable 没关系, 因此改名 |
| last_runnable_update | 上次更新 sched_avg 的时间 |
| decay_count | 用于计算当前 SE  blocked 状态时的待衰减周期, 每次调度实体出队时, 保存当前 cfs_rq 的 decay_counter, 下次入队更新时, 两个的差值就是已经经历的周期|
| load_avg_contrib | 进程的负载贡献, running_avg_sum * load_weight / avg_period, 参见 [\_\_update_entity_load_avg_contrib](https://elixir.bootlin.com/linux/v4.1/source/kernel/sched/fair.c#L2718) |
| utilization_avg_contrib | 进程的利用率, running_avg_sum * SCHED_LOAD_SCALE / avg_period, 参见 [\_\_update_task_entity_utilization](https://elixir.bootlin.com/linux/v4.1/source/kernel/sched/fair.c#L2744) |


这组补丁为了支持了 `frequency scale invariance`, 因此引入了利用率 utilization 的概念. 之前计算的 runnable_avg_sum 以及 load_avg_contrib 包含了调度实体 runnable 和 running 的负载和. 并不能很好的体现利用率. 利用率更在意的是它真正的运行. 因此 sched_avg 中引入了 running_avg_sum 和 utilization_avg_contrib, 分别表示其占用 CPU 的累计负载和利用率.

在不支持 `frequency scale invariance` 之前, 那么每个 1MS(1024us) 窗口, 其如果处于 R 状态, 那么当前窗口贡献的负载值就是 1024. 感知了频率变化之后, `__update_entity_runnable_avg` 每次在更新负载的时候, delta 会根据当前频率对应的 capacity 进行一个缩放.

```cpp
# https://elixir.bootlin.com/linux/v4.1/source/kernel/sched/fair.c#L2587
# https://elixir.bootlin.com/linux/v4.1/source/kernel/sched/fair.c#L2597
static __always_inline int __update_entity_runnable_avg(u64 now, int cpu,
                            struct sched_avg *sa,
                            int runnable,
                            int running)
{
    // ......
    if (delta + delta_w >= 1024) {
        // ......

        /* Efficiently calculate \sum (1..n_period) 1024*y^i */
        runnable_contrib = __compute_runnable_contrib(periods);
        if (runnable)
            sa->runnable_avg_sum += runnable_contrib;
        if (running)
            sa->running_avg_sum += runnable_contrib * scale_freq
                >> SCHED_CAPACITY_SHIFT;
        sa->avg_period += runnable_contrib;
    }

    /* Remainder of delta accrued against u_0` */
    if (runnable)
        sa->runnable_avg_sum += delta;
    if (running)
        sa->running_avg_sum += delta * scale_freq
            >> SCHED_CAPACITY_SHIFT;
    sa->avg_period += delta;

    return decayed;
}
```

这组补丁的理念中 load 是一个与频率无关的概念, 但是 utilization 利用率从并不是. 如果我们在两个有不同计算能力的 CPU 上运行两个 nice 0 的死循环程序, 那么他们的负载应该是相差不多的, 因为他们的所期望的计算需求是一样的. 因此在计算的时候只有 running_avg_sum 按照 `DELTA * scale_freq / 1024` 的方式缩放了, 而 runnable_avg_sum 并没有. 这样 [\_\_update_task_entity_utilization](https://elixir.bootlin.com/linux/v4.1/source/kernel/sched/fair.c#L2744) 在根据  running_avg_sum 计算 utilization_avg_contrib 的时候, 就会感知到每次 scale_freq 的变化.

# 3 PELT 4.3@2015 Rewrite runnable load and utilization average tracking
-------

| 时间  | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:------:|:---:|
| 2012/8/23 | [Rewrite runnable load and utilization average tracking](https://lore.kernel.org/patchwork/cover/579066) | 重构组调度的 PELT 跟踪, 在每次更新平均负载的时候, 更新整个 CFS_RQ 的平均负载| v10 ☑4.3 | [LWN](https://lwn.net/Articles/579066), [PatchWork](https://lore.kernel.org/patchwork/cover/590249), [lkml](https://lkml.org/lkml/2015/7/15/159) |

```cpp
7ea241afbf49 sched/fair: Clean up load average references
139622343ef3 sched/fair: Provide runnable_load_avg back to cfs_rq
1269557889b4 sched/fair: Remove task and group entity load when they are dead
540247fb5ddf sched/fair: Init cfs_rq's sched_entity load average
6c1d47c08273 sched/fair: Implement update_blocked_averages() for CONFIG_FAIR_GROUP_SCHED=n
9d89c257dfb9 sched/fair: Rewrite runnable load and utilization average tracking
cd126afe838d sched/fair: Remove rq's runnable avg
```

该补丁合入之后, sched_avg 结构体又发生了重大的变化.

```cpp
# https://elixir.bootlin.com/linux/v4.3/source/include/linux/sched.h#L1204
/*
 * The load_avg/util_avg accumulates an infinite geometric series.
 * 1) load_avg factors the amount of time that a sched_entity is
 * runnable on a rq into its weight. For cfs_rq, it is the aggregated
 * such weights of all runnable and blocked sched_entities.
 * 2) util_avg factors frequency scaling into the amount of time
 * that a sched_entity is running on a CPU, in the range [0..SCHED_LOAD_SCALE].
 * For cfs_rq, it is the aggregated such times of all runnable and
 * blocked sched_entities.
 * The 64 bit load_sum can:
 * 1) for cfs_rq, afford 4353082796 (=2^64/47742/88761) entities with
 * the highest weight (=88761) always runnable, we should not overflow
 * 2) for entity, support any load.weight always runnable
 */
struct sched_avg {
    u64 last_update_time, load_sum;
    u32 util_sum, period_contrib;
    unsigned long load_avg, util_avg;
};
```

| 字段 | 描述 |
|:-------:|:------:|
| last_update_time | 代替原来的 last_runnable_update, 记录上次更新 sched_avg 的时间 |
| load_sum | 接替原来的 runnable_avg_sum, 调度实体 sched_entity 在就绪队列上(on_rq) 累计负载之和, 带权重加权 |
| util_sum | 接替原来的 running_avg_sum, 调度实体 sched_entity 实际在 CPU 上运行(on_cpu) 的累计负载之和 |
| load_avg | 接替原来的 load_avg_contrib, 作为进程的负载贡献 sa->load_sum / LOAD_AVG_MAX, 参见 [`\_\_update_load_avg`](https://elixir.bootlin.com/linux/v4.3/source/kernel/sched/fair.c#L2518) |
| util_avg| 接替 utilization_avg_contrib, 成为进程的利用率 sa->util_sum << SCHED_LOAD_SHIFT) / LOAD_AVG_MAX; |
| period_contrib | 记录了当前进程最后运行的未满一个窗口(1024us) 的剩余时间 |
| ~~avg_period~~ | ~~调度实体 sched_entity 自创建至今如果持续运行, 所应该达到的累计负载,曾用名 runnable_avg_period~~, 用 LOAD_AVG_MAX 替代 |
| ~~decay_count~~ | ~~之前用于计算当前 SE  blocked 状态时的待衰减周期,  实现了 update_blocked_average 之后~~, 新的算法不需要此结构 |

cfs_rq 中记录的负载信息如下所示:

```cpp
# https://elixir.bootlin.com/linux/v4.3/source/kernel/sched/sched.h#L367
# https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9d89c257dfb9c51a532d69397f6eed75e5168c35
struct cfs_rq {
......
#ifdef CONFIG_SMP
    /*
     * CFS load tracking
     */
    struct sched_avg avg;
    u64 runnable_load_sum;
    unsigned long runnable_load_avg;
#ifdef CONFIG_FAIR_GROUP_SCHED
    unsigned long tg_load_avg_contrib;
#endif
    atomic_long_t removed_load_avg, removed_util_avg;
#ifndef CONFIG_64BIT
    u64 load_last_update_time_copy;
#endif
};
```

# 4 PELT 4.4@2015 Compute capacity invariant load/utilization tracking
-------


| 时间  | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:------:|:---:|
| 2015/8/14 | [Compute capacity invariant load/utilization tracking](https://lore.kernel.org/patchwork/cover/590249) | PELT 支持 Capacity Invariant, 对之前, 对 frequency scale invariance 的进一步优化 | V1 ☑4.4 | [LWN](https://lwn.net/Articles/531853), [PatchWork](https://lore.kernel.org/patchwork/cover/590249), [lkml](https://lkml.org/lkml/2015/8/14/296) |

这组补丁 Morten Rasmussen 自打 2014 年就开始推. 当前, 每个实体的负载跟踪仅对利用率跟踪的频率缩放进行补偿. 这个补丁集也扩展了这种补偿, 并增加了计算能力(不同的微架构和/或最大频率/P-state)的不变性. 前者防止在 cpu 以不同频率运行时做出次优负载平衡决策, 而后者确保可以跨 cpu 比较利用率(sched_avg.util_avg), 并且可以直接将利用率与 cpu 容量进行比较, 以确定cpu是否过载.


```cpp
98d8fd812667 sched/fair: Initialize task load and utilization before placing task on rq
231678b768da sched/fair: Get rid of scaling utilization by capacity_orig
9e91d61d9b0c sched/fair: Name utilization related data and functions consistently
e3279a2e6d69 sched/fair: Make utilization tracking CPU scale-invariant
8cd5601c5060 sched/fair: Convert arch_scale_cpu_capacity() from weak function to #define
e0f5f3afd2cf sched/fair: Make load tracking frequency scale-invariant
```



# 5 PELT 4.15@2017 A bit of a cgroup/PELT overhaul
-------

Peter 后来进行了一些优化

| 时间  | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:------:|:---:|
| 2017/09/01 | [ A bit of a cgroup/PELT overhaul](https://lore.kernel.org/patchwork/cover/827575) | | v2, 4.15 | [PatchWork](https://lore.kernel.org/patchwork/cover/827575), [LKML](https://lkml.org/lkml/2017/9/1/331) |


```cpp
17de4ee04ca9 sched/fair: Update calc_group_*() comments
2c8e4dce7963 sched/fair: Calculate runnable_weight slightly differently
9a2dd585b2c4 sched/fair: Implement more accurate async detach
f207934fb79d sched/fair: Align PELT windows between cfs_rq and its se
144d8487bc6e sched/fair: Implement synchonous PELT detach on load-balance migrate
1ea6c46a23f1 sched/fair: Propagate an effective runnable_load_avg
0e2d2aaaae52 sched/fair: Rewrite PELT migration propagation
2a2f5d4e44ed sched/fair: Rewrite cfs_rq->removed_*avg
9059393e4ec1 sched/fair: Use reweight_entity() for set_user_nice()
840c5abca499 sched/fair: More accurate reweight_entity()
8d5b9025f9b4 sched/fair: Introduce {en,de}queue_load_avg()
b5b3e35f4149 sched/fair: Rename {en,de}queue_entity_load_avg()
b382a531b9fe sched/fair: Move enqueue migrate handling
88c0616ee729 sched/fair: Change update_load_avg() arguments
c7b50216818e sched/fair: Remove se->load.weight from se->avg.load_sum
3d4b60d3e3dd sched/fair: Cure calc_cfs_shares() vs. reweight_entity()
cef27403cbe9 sched/fair: Add comment to calc_cfs_shares()
7c80cfc99b7b sched/fair: Clean up calc_cfs_shares()
```

这个版本 sched_avg 结构体变更如下

```cpp
# https://elixir.bootlin.com/linux/v4.15/source/include/linux/sched.h#L278
struct sched_avg {
    u64             last_update_time;
    u64             load_sum;
    u64             runnable_load_sum;
    u32             util_sum;
    u32             period_contrib;
    unsigned long           load_avg;
    unsigned long           runnable_load_avg;
    unsigned long           util_avg;
};
```

| 字段 | 描述 |
|:-------:|:------:|
| last_update_time | 代替原来的 last_runnable_update, 记录上次更新 sched_avg 的时间 |
| load_sum | 接替原来的 runnable_load_sum, 调度实体 sched_entity 在就绪队列上(on_rq) 累计负载 |
| util_sum | 接替原来的 running_avg_sum, 调度实体 sched_entity 实际在 CPU 上运行(on_cpu) 的累计负载 |
| load_avg | 接替原来的 load_avg_contrib, 作为进程的负载贡献 sa->load_sum / LOAD_AVG_MAX, 参见 [`\_\_update_load_avg`](https://elixir.bootlin.com/linux/v4.3/source/kernel/sched/fair.c#L2518) |
| util_avg| 接替 utilization_avg_contrib, 成为进程的利用率 sa->util_sum << SCHED_LOAD_SHIFT) / LOAD_AVG_MAX; |
| runnable_load_sum | |
| runnable_load_avg | |
| period_contrib | |

进程投入运行至今, 如果一直运行那么能达到的负载最大值是多少呢? 

最早的 PELT 3.8 版本是在 sched_avg 中存储了一个字段 runnable_avg_period, 用来表示进程自投入运行至今, 假设一直运行, 所能达到的最大负载, 每次 `__update_entity_runnable_avg` 都会更新, 参见 [9d85f21c94f7 sched: Track the runnable average on a per-task entity basis](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9d85f21c94f7f7a84d0ba686c58aa6d9da58fdbb). 此时 runnable_avg_period 的计算公式如下所示:

```cpp
runnable_avg_period(t) = \Sum 1024 * y^i

load_avg_contrib = runnable_load_sum * load.weight / (runnable_avg_period + 1);

```

接着的 PELT 4.1 版本, 引入了利用率, utilization_avg_contrib 的计算跟 load_avg_contrib 类似, 只不过只计算了 running 状态的负载. 这里由于 runnable_avg_period 不再只是跟 runnable 的负载有关系, 因此改名为 avg_period. 参见 [36ee28e45df5 sched: Add sched_avg::utilization_avg_contrib](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=36ee28e45df50c2c8624b978335516e42d84ae1f).

```cpp
# https://elixir.bootlin.com/linux/v4.1/source/include/sched/sched.h#L2718
load_avg_contrib = runnable_avg_sum * load.weight / (avg_period + 1);

# https://elixir.bootlin.com/linux/v4.1/source/include/sched/sched.h#L2744
utilization_avg_contrib = running_avg_sum * SCHED_LOAD_SCALE / (avg_period + 1);
```

后来 PELT 4.3 重构代码的时候, 删除了 avg_period, 转而使用 LOAD_AVG_MAX 替代. 参见 [9d89c257dfb9 sched/fair: Rewrite runnable load and utilization average tracking](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9d89c257dfb9).

```cpp
# https://elixir.bootlin.com/linux/v4.1/source/include/sched/sched.h#L12632
load_avg = load_sum / LOAD_AVG_MAX;

runnable_load_avg = runnable_load_sum /  LOAD_AVG_MAX;

util_avg = util_sum * SCHED_LOAD_SCALE / LOAD_AVG_MAX;
```


使用 LOAD_AVG_MAX 相当于假定最后窗口的 period_contrib 时间已经耗尽, 并且被认为是空闲的. 从而导致 CPU util 的负载永远漏掉了一个窗口的值, 对于一个负载非常重的 util, 它的值本应该保持在 1023 左右, 但是却一直维持在 [1002..1024] 的区间内. 因此在 PELT 4.13, 考虑使用 LOAD_AVG_MAX 直接作为进程的最大负载值是不合理的. 因此补丁 [625ed2bf049d sched/cfs: Make util/load_avg more stable](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=625ed2bf049d5a352c1bcca962d6e133454eaaff) 中考虑了一种最简易的计算方法.

```cpp
LOAD_AVG_MAX * y + 1024(us) = LOAD_AVG_MAX
max_value = LOAD_AVG_MAX * y + sa->period_contrib
```

那么进程如果打创建开始就一直投入运行, 那么能达到的负载最大值为:

```cpp
LOAD_AVG_MAX - 1024 + sa->period_contrib = LOAD_AVG_MAX - (1024 - sa->period_contrib)
```

语义上可以理解为, 最后一个的窗口只运行了 `sa->period_contrib`, 这个窗口不需要衰减.


# 5 PELT 4.17@2018 sched/fair: add util_est on top of PELT
-------

| 时间  | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:------:|:---:|
| 2020/2/21 | [sched/fair: add util_est on top of PELT](https://lore.kernel.org/patchwork/patch/889504) | 重构组调度的 PELT 跟踪, 在每次更新平均负载的时候, 更新整个 CFS_RQ 的平均负载| V10 ☑ 5.7 | [PatchWork](https://lore.kernel.org/patchwork/cover/1198654), [lkml](https://lkml.org/lkml/2020/2/21/1386) |


由于 `PELT` 衰减的特质, 它并不十分适合终端等场景. 对于 `big.LITTLE` 架构, 一个重量级的任务可能在长时间的睡眠唤醒之后, 进程的利用率 `util` 几乎衰减到微乎其微. 那么它很有可能被唤醒到 `LITTLE` 核上, 对性能造成影响. 其次任务的 `util` 每个周期(1024us) 更新一次, 因此对于一个一直运行的任务, 他的 util 是一个不断变化的值, 那么一个正在运行的任务其瞬时的 `PELT` 利用率直接参与调度器的决策不太合理.

因此内核需要一个平滑的估计值, 一个更稳定的利用率估计能更好的标记 CFS 和 RQ 的负载.


```cpp
d519329f72a6 sched/fair: Update util_est only on util_avg updates
a07630b8b2c1 sched/cpufreq/schedutil: Use util_est for OPP selection
f9be3e5961c5 sched/fair: Use util_est in LB and WU paths
7f65ea42eb00 sched/fair: Add util_est on top of PELT
```

该补丁合入之后, 在 `sched_avg` 结构中新增了一个 `util_est`, 用来标记估计的利用率信息.

```cpp
# https://elixir.bootlin.com/linux/v5.7/source/include/linux/sched.h#L278
# https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7f65ea42eb00bc902f1c37a71e984e4f4064cfa9
+/**
+ * struct util_est - Estimation utilization of FAIR tasks
+ * @enqueued: instantaneous estimated utilization of a task/cpu
+ * @ewma:     the Exponential Weighted Moving Average (EWMA)
+ *            utilization of a task
+ *
+ * Support data structure to track an Exponential Weighted Moving Average
+ * (EWMA) of a FAIR task's utilization. New samples are added to the moving
+ * average each time a task completes an activation. Sample's weight is chosen
+ * so that the EWMA will be relatively insensitive to transient changes to the
+ * task's workload.
+ *
+ * The enqueued attribute has a slightly different meaning for tasks and cpus:
+ * - task:   the task's util_avg at last task dequeue time
+ * - cfs_rq: the sum of util_est.enqueued for each RUNNABLE task on that CPU
+ * Thus, the util_est.enqueued of a task represents the contribution on the
+ * estimated utilization of the CPU where that task is currently enqueued.
+ *
+ * Only for tasks we track a moving average of the past instantaneous
+ * estimated utilization. This allows to absorb sporadic drops in utilization
+ * of an otherwise almost periodic task.
+ */
+struct util_est {
+   unsigned int            enqueued;
+   unsigned int            ewma;
+#define UTIL_EST_WEIGHT_SHIFT      2
+};
+
 /*
  * The load_avg/util_avg accumulates an infinite geometric series
  * (see __update_load_avg() in kernel/sched/fair.c).
@@ -335,6 +363,7 @@ struct sched_avg {
    unsigned long           load_avg;
    unsigned long           runnable_load_avg;
    unsigned long           util_avg;
+   struct util_est         util_est;
 };
```

# 6 PELT 5.1@2019 update scale invariance of PELT
-------

| 时间  | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:------:|:---:|
| 2019/01/16 | [sched/fair: update scale invariance of PELT](https://lore.kernel.org/patchwork/cover/1034952) |  | [v3](https://lore.kernel.org/patchwork/patch/784059)<br>[v9](https://lore.kernel.org/patchwork/cover/1034952) |

# 6 PELT 5.7@2020 remove runnable_load_avg and improve group_classify
-------

| 时间  | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:------:|:---:|
| 2020/2/21 | [remove runnable_load_avg and improve group_classify](https://lore.kernel.org/patchwork/cover/1198654) | 重构组调度的 PELT 跟踪, 在每次更新平均负载的时候, 更新整个 CFS_RQ 的平均负载| V10 ☑ 5.7 | [PatchWork](https://lore.kernel.org/patchwork/cover/1198654), [lkml](https://lkml.org/lkml/2020/2/21/1386) |


```cpp
070f5e860ee2 sched/fair: Take into account runnable_avg to classify group
9f68395333ad sched/pelt: Add a new runnable average signal
0dacee1bfa70 sched/pelt: Remove unused runnable load average
6499b1b2dd1b sched/numa: Replace runnable_load_avg by load_avg
6d4d22468dae sched/fair: Reorder enqueue/dequeue_task_fair path
```

# 7 参考资料
-------

[task 的 load_avg_contrib 的更新参考](https://www.codenong.com/cs106477101)

