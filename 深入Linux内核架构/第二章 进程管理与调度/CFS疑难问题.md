# CFS疑难问题

# CFS如何保证在一个调度延迟周期中保证所有进程都被调度到?

**回答:在每个周期调度器的每个滴答周期中,检查当前进程的执行时间是否超过预期时间;**

> 注:这个检查是否超过预期是在每次周期调度器中去计算和验证的,而非在调度周期开始的时候进行计算的;

1. ## 背景

在CFS调度器中,CFS会保证每一个调度周期中每一个进程都会被调度;

1. ## 概念

完全公平调度类使用了一种动态时间片的算法，给每个进程分配CPU占用时间。

**调度延迟指的是任何一个可运行进程两次运行之间的时间间隔。**

比如调度延迟是20毫秒，那 么每个进程可以执行10毫秒；如果是4个进程，可以执行5毫秒。调度延迟称为 sysctl_sched_latency，记录在/proc/sys/kernel/sched_latency_ns中，以纳秒为单位。

如果进程很多，那么可能每个进程每次运行的时间都很短，这浪费了大量的时间进行 调度。所以引入了调度最小粒度的概念。除非进程进行了阻塞任务或者主动让出CPU， 否则进程至少执行调度最小粒度的时间。调度最小粒度称为sysctl_sche_min_granularity， 记录在/proc/sys/sched_min_granulariry_ns中，以纳秒为单位。

[ sched_nr_latency=\frac{sysctl_sched_latency}{sysctl_sched_min_granularity} ]

这个比值是一个调度延迟内允许的最大运行数目;

1. ## 数据结构

```C
/*
 CPU绑定任务的目标抢占延迟
 
 NOTE: 这个抢占延迟值与传统的“时间片长度”概念并不相同。
 在传统的时间片轮转调度中，时间片是固定长度的，而在 CFS（Completely Fair Scheduler）中，
 时间片长度是可变的，并且没有持久的概念。

 (default: 6ms * (1 + ilog(ncpus)), units: nanoseconds)
*/
unsigned int sysctl_sched_latency           = 6000000ULL;  // 6ms
static unsigned int normalized_sysctl_sched_latency = 6000000ULL;

/*
 * CPU绑定任务的最小抢占粒度
 *
 * (default: 0.75 msec * (1 + ilog(ncpus)), units: nanoseconds)
 */
unsigned int sysctl_sched_min_granularity           = 750000ULL;  // 0.75ms
static unsigned int normalized_sysctl_sched_min_granularity = 750000ULL;
```

1. ## 检查过程

1. ### 实际方式总述

其实,保证保证一个调度周期汇总的所有进程都被执行是非常简单的一件事情,当我们换一个角度去看这个问题的时候:

**保证一个调度周期汇总的所有进程都被执行 == 保证就绪队列中的每个进程都不运行超过预期的时间片**

> 当然,还有一些细节,例如在本轮的就绪队列是不可以发生改变的,换句话说,例如新被创建的进程的,这需要特殊的处理,后面我们补充了这种情况的处理方式;

```C
周期调度器检查是否需要抢占 entity_tick
-> 计算真实的运行时间和预期的运行时间 check_preempt_tick
  -> 计算预期时间   sched_slice
    -> 计算当前延迟调度周期时间     __sched_period
    -> 计算当前理想的真实运行时间     __calc_delta
```

1. ### 延迟调度的计算

- 如果可运行进程个数小于 sched_nr_latency，调度周期等于调度延迟。
- 如果可运行进程超过了sched_nr_latency， 系统就不去理会调度延迟，转而保证调度最小粒度，这种情况下，调度周期等于最小粒度乘可运行进程个数。这在kernel/sched/fair.c中计算调度周期的函数可以看出来。

```C
/*
 * The idea is to set a period in which each task runs once.
 *
 * When there are too many tasks (sched_nr_latency) we have to stretch
 * this period because otherwise the slices get too small.
 *
 * p = (nr <= nl) ? l : l*nr/nl
 */
static u64 __sched_period(unsigned long nr_running)
{
    if (unlikely(nr_running > sched_nr_latency))
        return nr_running * sysctl_sched_min_granularity;
    else
        return sysctl_sched_latency;
}
```

1. ### 计算进程在一个调度周期内的真实运行时间

在kernel/sched/fair.c中定义了完全公平调度的相关函数。其中sched_slice负责计 算一个进程在本轮调度周期应分得的真实运行时间。

```C
/*
 * delta_exec * weight / lw.weight
 *   OR
 * (delta_exec * (weight * lw->inv_weight)) >> WMULT_SHIFT
 *
 * Either weight := NICE_0_LOAD and lw \e sched_prio_to_wmult[], in which case
 * we're guaranteed shift stays positive because inv_weight is guaranteed to
 * fit 32 bits, and NICE_0_LOAD gives another 10 bits; therefore shift >= 22.
 *
 * Or, weight =< lw.weight (because lw.weight is the runqueue weight), thus
 * weight/lw.weight <= 1, and therefore our shift will also be positive.
 */
static u64 __calc_delta(u64 delta_exec, unsigned long weight, struct load_weight *lw)
{
        u64 fact = scale_load_down(weight);
        int shift = WMULT_SHIFT;
 
        __update_inv_weight(lw);
 
        if (unlikely(fact >> 32)) {
                while (fact >> 32) {
                        fact >>= 1;
                        shift--;
                }
        }
 
        /* hint to use a 32x32->64 mul */
        fact = (u64)(u32)fact * lw->inv_weight;
 
        while (fact >> 32) {
                fact >>= 1;
                shift--;
        }
 
        return mul_u64_u32_shr(delta_exec, fact, shift);
}
 
/*
 * The idea is to set a period in which each task runs once.
 * 其想法是设置一个周期，每个任务在该周期内运行一次。
 
 * When there are too many tasks (sched_nr_latency) we have to stretch
 * this period because otherwise the slices get too small.
 *
 * p = (nr <= nl) ? l : l*nr/nl
 */
static u64 __sched_period(unsigned long nr_running)
{
        if (unlikely(nr_running > sched_nr_latency))
                return nr_running * sysctl_sched_min_granularity;
        else
                return sysctl_sched_latency;
}
 
/*
 * We calculate the wall-time slice from the period by taking a part
 * proportional to the weight.
 *
 * s = p*P[w/rw]
 */
static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
        u64 slice = __sched_period(cfs_rq->nr_running + !se->on_rq);
 
        for_each_sched_entity(se) {
                struct load_weight *load;
                struct load_weight lw;
 
                cfs_rq = cfs_rq_of(se);
                load = &cfs_rq->load;
 
                if (unlikely(!se->on_rq)) {
                        lw = cfs_rq->load;
 
                        update_load_add(&lw, se->load.weight);
                        load = &lw;
                }
                slice = __calc_delta(slice, se->load.weight, load);
        }
        return slice;
}

#define for_each_sched_entity(se) \
        for (; se; se = NULL)
```

1. ### 处理周期性调度器

在周期性调度器中检查当前进程是否超过预期的时间,这里调用了check_preempt_tick方法;

可以看到,其实周期性调度器做的任务是非常简单的,只有更新当前的se节点,更新rq组的load等等;

```C
static void entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
{
    /*
     * Update run-time statistics of the 'current'.
     */
    update_curr(cfs_rq);

    /*
     * Ensure that runnable average is periodically updated.
     */
    update_load_avg(cfs_rq, curr, UPDATE_TG);
    update_cfs_group(curr);

    if (cfs_rq->nr_running > 1)
        check_preempt_tick(cfs_rq, curr);
}
```

1. ### 检查是否需要抢占

check_preempt_tick作为具体的检查是否超过预期的时间;

```C
/*
 * Preempt the current task with a newly woken task if needed:
 */
static void
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
    unsigned long ideal_runtime, delta_exec;
    struct sched_entity *se;
    s64 delta;
    
    // 理想的运行时间
    ideal_runtime = sched_slice(cfs_rq, curr);
    // 真实的运行时间
    delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
    if (delta_exec > ideal_runtime) {
        resched_curr(rq_of(cfs_rq));
        /*
         * The current task ran long enough, ensure it doesn't get
         * re-elected due to buddy favours.
         */
        clear_buddies(cfs_rq, curr);
        return;
    }

    /*
     * Ensure that a task that missed wakeup preemption by a
     * narrow margin doesn't have to wait for a full slice.
     * This also mitigates buddy induced latencies under load.
     */
    if (delta_exec < sysctl_sched_min_granularity)
        return;

    se = __pick_first_entity(cfs_rq);
    delta = curr->vruntime - se->vruntime;

    if (delta < 0)
        return;

    if (delta > ideal_runtime)
        resched_curr(rq_of(cfs_rq));
}
```