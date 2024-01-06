# 1. TaskTurbo简介

# 2. Task Turbo接口

在task turbo中，一共暴露出了三个与task turbo相关的接口，feats，turbo_pid与unset_turbo_pid，分别。

## 2.1 代码位置与导出接口

代码位置：

```C++
drivers\misc\mediatek\task_turbo\task_turbo.c
```

导出接口：/sys/module/task_turbo/parameters

```C++
/sys/module/task_turbo/parameters # ls -l
-rw-rw-r-- 1 system system 4096 2021-01-02 09:39 feats
-rw-r--r-- 1 root   root   4096 2021-01-02 09:39 turbo_pid
-rw-r--r-- 1 root   root   4096 2021-01-02 08:35 unset_turbo_pid
```

## 2.2 接口说明

### 2.2.1 feats

feats的主要作用还是用于判断需要设置的功能，从代码的定义上来看主要包含latency和launch，但是就MTK同事所言，目前只实现了launch的功能，feats 对应操作函数集为 task_turbo_feats_param_ops。

```C++
// drivers\misc\mediatek\task_turbo\task_turbo.c
static struct kernel_param_ops task_turbo_feats_param_ops = {
        .set = set_task_turbo_feats,
        .get = param_get_uint,
};

param_check_uint(feats, &task_turbo_feats);
module_param_cb(feats, &task_turbo_feats_param_ops, &task_turbo_feats, 0644);
MODULE_PARM_DESC(feats, "enable task turbo features if needed");

// 根据bit位决定是哪个功能
static uint32_t latency_turbo = SUB_FEAT_LOCK | SUB_FEAT_BINDER | SUB_FEAT_SCHED;
static uint32_t launch_turbo =  SUB_FEAT_LOCK | SUB_FEAT_BINDER | SUB_FEAT_SCHED | SUB_FEAT_FLAVOR_BIGCORE;

//turbo_common.h
enum {
    SUB_FEAT_LOCK        = 1U << 0,
    SUB_FEAT_BINDER        = 1U << 1,
    SUB_FEAT_SCHED        = 1U << 2,
    SUB_FEAT_FLAVOR_BIGCORE = 1U << 3, //这个是更偏向于大核？
};

//判断使能的是哪个feature的
inline bool latency_turbo_enable(void)
{
    return task_turbo_feats == latency_turbo;
}
inline bool launch_turbo_enable(void)
{
    return task_turbo_feats == launch_turbo;
}
```

文件取值是下面字段的组成的位掩码，每个bit是一个子feature，由于子feature之间存在依赖并不是所有掩码都能设置成功，实测只能设置7和15，区别如下，只有使能了SUB_FEAT_FLAVOR_BIGCORE 才会设置 p->cpu_prefer 。

若往里面设置0，会清空数组 turbo_pid[8] 里面所有的pid，并且unset数组里面所有pid对应的任务。

### 2.2.2 turbo_pid

turbo_pid 对应操作函数集为 turbo_pid_param_ops。

```C++
// drivers\misc\mediatek\task_turbo\task_turbo.c

static struct kernel_param_ops turbo_pid_param_ops = {
        .set = set_turbo_task_param,
        .get = param_get_int,
};

param_check_uint(turbo_pid, &turbo_pid_param);
module_param_cb(turbo_pid, &turbo_pid_param_ops, &turbo_pid_param, 0644);
MODULE_PARM_DESC(turbo_pid, "set turbo task by pid");
```

调用逻辑：

set_turbo_task_param->add_turbo_list_by_pid->set_turbo_task->set_scheduler_tuning->set_user_nice

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=ZWU4MjQ5NzY1YzcwODdlOWQyOWIxMWNiZjJkZWMzMTJfdlVJZW1BNnZnSTd2NEc5NEhvSktEUXc5bG1WNjlUSEJfVG9rZW46WXA0dGJRemZsbzFvRkF4TDMxa2NZSWVZbkZnXzE3MDQ1MzEzMzQ6MTcwNDUzNDkzNF9WNA)

存储写入的pid，使用一个一维8个元素的数组来存储，最大应该支持同时boost 8个任务。

先存到数组turbo_pid[8]中，保证不重复，若数组存满了，就直接返回false退出设置。然后将任务的p->turbo=1，p->cpu_prefer=1(perfer big=1,mid=2)。

设置完后立即调用了一次 set_user_nice 触发设置生效。

### 2.2.3 unset_turbo_pid 

unset_turbo_pid 对应操作函数集为 unset_turbo_pid_param_ops。

```C++
static struct kernel_param_ops unset_turbo_pid_param_ops = {
        .set = unset_turbo_task_param,
        .get = param_get_int,
};

param_check_uint(unset_turbo_pid, &unset_turbo_pid_param);
module_param_cb(unset_turbo_pid, &unset_turbo_pid_param_ops,
                &unset_turbo_pid_param, 0644);
MODULE_PARM_DESC(unset_turbo_pid, "unset turbo task by pid");
```

清除对某个pid对应任务的设置，先从 turbo_pid[8] 数组中清除掉这个pid，然后对这个任务执行复位操作：p->turbo=0，p->cpu_prefer=0。

设置完后立即调用了一次 set_user_nice 触发设置生效。

## 2.3 补充

只支持对cfs任务的设置生效，应该在配置关键函数中判断非 fair_policy(task->policy) 的话就直接退出了。设置前需要先将 feats 文件设置好，因为在pid的设置路径中判断了若 feats 没有设置就直接退出了。从 is_turbo_task() 判断中可以看出，不仅支持直接设置还支持继承设置，p->inherit_types 不为0就表示是继承的，也会被 boost.

# 3. 实现逻辑

## 3.1 设置大核亲核性

### 3.1.1 set_scheduler_tuning与unset_scheduler_tuning

set_scheduler_tuning是真正落实了对指定进程设置优先级与亲核性的方法。

```C++
// 设置了turbo_task的亲核性
static inline void set_scheduler_tuning(struct task_struct *task){
        int cur_nice = task_nice(task);

        if (!fair_policy(task->policy))
                return;
        if (!sub_feat_enable(SUB_FEAT_SCHED))
                return;
        if (sub_feat_enable(SUB_FEAT_FLAVOR_BIGCORE))
                sched_set_cpuprefer(task->pid, SCHED_PREFER_BCPU);

        /* trigger renice for turbo task */
        set_user_nice(task, 0xbeef);

        trace_sched_turbo_nice_set(task, NICE_TO_PRIO(cur_nice), task->prio);
}

// 取消了turbo_task的亲核性
static inline void unset_scheduler_tuning(struct task_struct *task){
        int cur_prio = task->prio;

        if (!fair_policy(task->policy))
                return;

        sched_set_cpuprefer(task->pid, SCHED_PREFER_NONE);
        set_user_nice(task, 0xbeee);

        trace_sched_turbo_nice_set(task, cur_prio, task->prio);
}
```

### 3.1.2 sched_set_cpuprefer：设置亲核性

```C++
int sched_set_cpuprefer(pid_t pid, unsigned int prefer_type)
{
        struct task_struct *p;
        unsigned long flags;
        int retval = 0;

        if (!valid_cpu_prefer(prefer_type) || pid < 0)
                return -EINVAL;

        rcu_read_lock();
        retval = -ESRCH;
        p = find_task_by_vpid(pid);
        if (p != NULL) {
                raw_spin_lock_irqsave(&p->pi_lock, flags);
                p->cpu_prefer = prefer_type;
                raw_spin_unlock_irqrestore(&p->pi_lock, flags);
                trace_sched_set_cpuprefer(p);
                retval = 0;
        }
        rcu_read_unlock();

        return retval;
}
```

### 3.1.3 set_user_nice：设置优先级

主要是通过 ``nice = rlimit_to_nice(task_rlimit(p, RLIMIT_NICE));``来获取具体的nice值并在下文中进行设置；

**`task_rlimit(p, RLIMIT_NICE)`****作用为：返回进程p的RLIMIT_NICE资源限制值，然后****`rlimit_to_nice`****将该值转换为对应的nice值。最终，变量nice的值就是进程p的nice值。**

- `task_rlimit`是用于用于获取或设置进程的资源限制，包含了很多资源项，其中RLIMIT_NICE代表用于获取其进程的nice值限制。

> 其中，RLIMIT_NICE ：*#define RLIMIT_NICE    13     /\* max nice prio allowed to raise to  0-39 for nice level 19 .. -20 \*/*

- `rlimit_to_nice`是用于将资源限制值转换为对应的nice值。

```C++
// F:\Xiaomi_Kernel_OpenSource-selene-r-oss\kernel\sched\core.c
void set_user_nice(struct task_struct *p, long nice)
{
        bool queued, running;
        int old_prio, delta;
        struct rq_flags rf;
        struct rq *rq;

#ifdef CONFIG_MTK_TASK_TURBO
        if ((nice < MIN_NICE || nice > MAX_NICE) && !task_turbo_nice(nice))
                return;
#else
        if (task_nice(p) == nice || nice < MIN_NICE || nice > MAX_NICE)
                return;
#endif
        /*
         * We have to be careful, if called from sys_setpriority(),
         * the task might be in the middle of scheduling on another CPU.
         */
        rq = task_rq_lock(p, &rf);
        update_rq_clock(rq);

#ifdef CONFIG_MTK_TASK_TURBO
        /* for general use, backup it */
        if (!task_turbo_nice(nice))
                p->nice_backup = nice;

        if (is_turbo_task(p)) {
                nice = rlimit_to_nice(task_rlimit(p, RLIMIT_NICE));
                if (unlikely(nice > MAX_NICE)) {
                        printk_deferred("[name:task-turbo&]pid=%d RLIMIT_NICE=%ld is not set\n",p->pid, nice);
                        nice = p->nice_backup;
                }
        }
        else
                nice = p->nice_backup;

        trace_sched_set_user_nice(p, nice, is_turbo_task(p));
#endif

        /*
         * The RT priorities are set via sched_setscheduler(), but we still
         * allow the 'normal' nice value to be set - but as expected
         * it wont have any effect on scheduling until the task is
         * SCHED_DEADLINE, SCHED_FIFO or SCHED_RR:
         */
        if (task_has_dl_policy(p) || task_has_rt_policy(p)) {
                p->static_prio = NICE_TO_PRIO(nice);
                goto out_unlock;
        }
        queued = task_on_rq_queued(p);
        running = task_current(rq, p);
        if (queued)
                dequeue_task(rq, p, DEQUEUE_SAVE | DEQUEUE_NOCLOCK);
        if (running)
                put_prev_task(rq, p);

        p->static_prio = NICE_TO_PRIO(nice);
        set_load_weight(p);
        old_prio = p->prio;
        p->prio = effective_prio(p);
        delta = p->prio - old_prio;

        if (queued) {
                enqueue_task(rq, p, ENQUEUE_RESTORE | ENQUEUE_NOCLOCK);
                /*
                 * If the task increased its priority or is running and
                 * lowered its priority, then reschedule its CPU:
                 */
                if (delta < 0 || (delta > 0 && task_running(rq, p)))
                        resched_curr(rq);
        }
        if (running)
                set_curr_task(rq, p);
out_unlock:
        task_rq_unlock(rq, p, &rf);
}
EXPORT_SYMBOL(set_user_nice);
```

## 3.2 设置亲核性的时机(*)

- 修改task_turbo接口：feats、turbo_pid、unset_turbo_pid
- 修改cgroup接口：tasks、cgroup.procs

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=MGM1NGExNTljMmU5NmM1MzBkMjQyMWVlY2I2NzQ4YjVfM2xEVHpjeFBoTXZNVWxsSzQzS2JrUm45blpIRlUzbGFfVG9rZW46VE5iMWI2MlVUb2g5RnF4eVg5MmNnRTJubkFoXzE3MDQ1MzEzMzQ6MTcwNDUzNDkzNF9WNA)

## 3.3 设置task_turbo接口（set_turbo_task）

这里主要说的是通过注册文件系统的节点的读写方法，注册了set_task_turbo_feats（feats）、set_turbo_task_param（turbo_pid）、unset_turbo_task_param（unset_turbo_pid），来实现对目标进程的亲核性设置。

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjNmYWVkZTNiZjdiYmIyMzZhYjRmNWZhYTA5ZjI0MzdfMXhSYlVma3VCZVRSUHRXOEo3eWdpUTFsd0QyMFVKeVRfVG9rZW46U3FlRmI4REpXbzUxWHF4d0hRMWNUVW1IbmplXzE3MDQ1MzEzMzQ6MTcwNDUzNDkzNF9WNA)

### 3.3.1 set_turbo_task

**代码作用：**

该代码的主要作用是设置指定进程的 Turbo 任务。通过传入的进程ID (`pid`) 和一个标志位 (`val`)，可以启用或禁用该进程的 Turbo 功能。Turbo 功能可能与调度器（scheduler）的调优有关，可以根据需要进行相应的设置。

**说明：**

相当于task_turbo暴露的三个接口使用的调用set_scheduler_tuning的接口，通过传入val来决定后面的行为，相当于一个函数可以传递开启和关闭两种状态的方法。

```C++
static int set_turbo_task(int pid, int val)
{
        struct task_struct *p;
        int retval = 0;

        if (pid < 0 || pid > PID_MAX_DEFAULT)
                return -EINVAL;

        if (val < 0 || val > 1)
                return -EINVAL;

        rcu_read_lock();
        p = find_task_by_vpid(pid);

        if (p != NULL) {
                get_task_struct(p);
                p->turbo = val;
                if (p->turbo == TURBO_ENABLE)
                        set_scheduler_tuning(p);
                else
                        unset_scheduler_tuning(p);
                trace_turbo_set(p);
                put_task_struct(p);
        } else
                retval = -ESRCH;
        rcu_read_unlock();

        return retval;
}
```

### 3.3.2 set_task_turbo_feats（feats）

**代码作用：**

该代码的主要目的是设置任务的 Turbo 特性，其中 Turbo 特性可能包括 latency_turbo 和 launch_turbo。同时，该代码还处理了禁用 Turbo 的情况，即当传入的值为 0 时，会移除所有 Turbo 任务。

**说明：**

在这个函数中，**不做对进程设置亲核性的操作，而是只设置了一下feat的值**，也可以说是只通过feat设置了当前的功能，而在关闭turbo的时候（val == 0），移除当前所有的turbo task。

```C++
static int set_task_turbo_feats(const char *buf, const struct kernel_param *kp)
{
        int ret = 0, i;
        unsigned int val;
        
        // 使用 kstrtouint 函数将输入缓冲区中的值转换为无符号整数，并将结果存储在 val 中。
        ret = kstrtouint(buf, 0, &val);


        mutex_lock(&TURBO_MUTEX_LOCK);
        if (val == latency_turbo || val == launch_turbo  || val == 0)
                ret = param_set_uint(buf, kp);
        else
                ret = -EINVAL;

        /* if disable turbo, remove all turbo tasks */
        /* mutex_lock(&TURBO_MUTEX_LOCK); */
        if (val == 0) {
                for (i = 0; i < TURBO_PID_COUNT; i++) {
                        if (turbo_pid[i]) {
                                unset_turbo_task(turbo_pid[i]);
                                turbo_pid[i] = 0;
                        }
                }
        }
        mutex_unlock(&TURBO_MUTEX_LOCK);

        if (!ret)
                pr_info("task_turbo_feats is change to %d successfully",
                                task_turbo_feats);
        return ret;
}
```

### 3.3.3 set_turbo_task_param（turbo_pid）

**代码作用：**

该代码的主要作用是设置 Turbo 任务的参数，通过传入的进程ID将其添加到 Turbo 列表中，并更新全局变量 `turbo_pid_param` 为传入的进程ID。

**说明：**

在这个函数中，在外部会设置pid的值到turbo_pid中，并触发set_turbo_task_param，在这个函数中，会使用add_turbo_list_by_pid方法将指定进程加入turbo_list中，并最终触发set_scheduler_tuning设置亲核性。

```C++
static int set_turbo_task_param(const char *buf, const struct kernel_param *kp)
{
        int retval = 0;
        pid_t pid;

        retval = kstrtouint(buf, 0, &pid);

        if (!retval)
                retval = add_turbo_list_by_pid(pid);

        if (!retval)
                turbo_pid_param = pid;

        return retval;
}


/*
 * use pid set turbo task
 */
static int add_turbo_list_by_pid(pid_t pid)
{
        int retval = -EINVAL;

        if (!task_turbo_feats)
                return retval;

        if (pid < 0 || pid > PID_MAX_DEFAULT)
                return retval;

        mutex_lock(&TURBO_MUTEX_LOCK);
        if (!add_turbo_list_locked(pid))
                goto unlock;

        retval = set_turbo_task(pid, TURBO_ENABLE);
unlock:
        mutex_unlock(&TURBO_MUTEX_LOCK);
        return retval;
}

static int set_turbo_task(int pid, int val)
{
       // 参考第一小节
}
```

### 3.3.4 unset_turbo_task_param（unset_turbo_pid）

**代码作用：**

该代码的主要作用是取消 Turbo 任务的参数，通过传入的进程ID将其添加到 Turbo 列表中，并更新全局变量 `turbo_pid_param` 为传入的进程ID。

**说明：**

相当于set_turbo_task_param的反函数，用于取消置顶pid的进程的turbo_task状态，其实本质上unset_turbo_task只是不带参数的set_turbo_task的包装。

```C++
static int unset_turbo_task_param(const char *buf,const struct kernel_param *kp){
        int retval = 0;
        pid_t pid;

        retval = kstrtouint(buf, 0, &pid);

        if (!retval)
                retval = unset_turbo_list_by_pid(pid);

        if (!retval)
                unset_turbo_pid_param = pid;

        return retval;
}

static int unset_turbo_list_by_pid(pid_t pid){
        int retval = -EINVAL;

        if (pid < 0 || pid > PID_MAX_DEFAULT)
                return retval;

        mutex_lock(&TURBO_MUTEX_LOCK);
        remove_turbo_list_locked(pid);
        retval = unset_turbo_task(pid);
        mutex_unlock(&TURBO_MUTEX_LOCK);
        return retval;
}

static inline int unset_turbo_task(int pid){
        return set_turbo_task(pid, TURBO_DISABLE);
}
```

## 3.4 设置cgroup接口

在启动时，需要将对应进程设置为turbo task，便于后面对turbo task的处理，具体的设置时机为：**attach 到 top-app 分组时，判断是否为render线程或进程的主线程，如果是就将其设置为 turbo task.**

函数的调用逻辑为：

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=YzlmMWU5YWMwYmQ3NWRkNjA5NjUzOGYxNWMzMTkxNjdfVXVNRU5oSFpSanlNejhYdllBU3FiVE9nYUtIR1BkY0RfVG9rZW46SEZzRGJKNUVQb2NTblR4NHZVOWNFUnZ0bmJlXzE3MDQ1MzEzMzQ6MTcwNDUzNDkzNF9WNA)

### 3.4.1 add_turbo_list

用于把指定进程加入turbo_list，同时使用set_scheduler_tuning设置其亲核性。

```C++
static void add_turbo_list(struct task_struct *p){
        mutex_lock(&TURBO_MUTEX_LOCK);
        if (add_turbo_list_locked(p->pid)) {
                p->turbo = TURBO_ENABLE;
                set_scheduler_tuning(p);
                trace_turbo_set(p);
        }
        mutex_unlock(&TURBO_MUTEX_LOCK);
}


// 补充：remove app list
static void remove_turbo_list_locked(pid_t pid){
        int i;
        for (i = 0; i < TURBO_PID_COUNT; i++) {
                if (turbo_pid[i] == pid) {
                        turbo_pid[i] = 0;
                        break;
                }
        }
}
```

### 3.4.2 cgroup1_procs_write&&cgroup1_tasks_write

在attach 到 top-app 分组时，会触发到对应的回调函数：cgroup1_procs_write和cgroup1_tasks_write。

这两个函数被注册在cgroup_base_files[]中，下一小节我们会讲cgroup_base_files的注册流程。

```C++
// F:\Xiaomi_Kernel_OpenSource-selene-r-oss\kernel\cgroup\cgroup-v1.c
cgroup1_base_files.write //回调，对应文件"cgroup.procs"
    cgroup1_procs_write
        __cgroup1_procs_write(of, buf, nbytes, off, true); //cgroup_v1.c
            cgroup_set_turbo_task

cgroup1_base_files.write //回调，对应文件"tasks"
    cgroup1_tasks_write
        __cgroup1_procs_write(of, buf, nbytes, off, false); //cgroup_v1.c
            cgroup_set_turbo_task
```

cgroup1_procs_write&&cgroup1_tasks_write实际上都是调用了__cgroup1_procs_write进而调用了cgroup_set_turbo_task函数：

```C++
static ssize_t cgroup1_procs_write(struct kernfs_open_file *of,char *buf, size_t nbytes, loff_t off){
    return __cgroup1_procs_write(of, buf, nbytes, off, true);
}

static ssize_t cgroup1_tasks_write(struct kernfs_open_file *of, char *buf, size_t nbytes, loff_t off){
    return __cgroup1_procs_write(of, buf, nbytes, off, false);
}
```

### 3.4.3 __cgroup1_procs_write

这个函数不属于task_turbo.c的内容，严格上来说是定义在kernel中的内核函数，只是并非Kernel原生函数，是MTK客制化的文件。（这里存疑）

```C++
// F:\Xiaomi_Kernel_OpenSource-selene-r-oss\kernel\cgroup\cgroup-v1.c
static ssize_t __cgroup1_procs_write(struct kernfs_open_file *of, char *buf, size_t nbytes, loff_t off,bool threadgroup){
        struct cgroup *cgrp;
        struct task_struct *task;
        const struct cred *cred, *tcred;
        ssize_t ret;

        cgrp = cgroup_kn_lock_live(of->kn, false);
        if (!cgrp)
                return -ENODEV;

        task = cgroup_procs_write_start(buf, threadgroup);
        ret = PTR_ERR_OR_ZERO(task);
        if (ret)
                goto out_unlock;

        /*
         * Even if we're attaching all tasks in the thread group, we only
         * need to check permissions on one of them.
         */
        cred = current_cred();
        tcred = get_task_cred(task);
        if (!uid_eq(cred->euid, GLOBAL_ROOT_UID) &&
            !uid_eq(cred->euid, tcred->uid) &&
            !uid_eq(cred->euid, tcred->suid) &&
            !ns_capable(tcred->user_ns, CAP_SYS_NICE))
                ret = -EACCES;
        put_cred(tcred);
        if (ret)
                goto out_finish;

        ret = cgroup_attach_task(cgrp, task, threadgroup);
#ifdef CONFIG_MTK_TASK_TURBO
        if (!ret)
                cgroup_set_turbo_task(task);
#endif

out_finish:
        cgroup_procs_write_finish(task);
out_unlock:
        cgroup_kn_unlock(of->kn);

        return ret ?: nbytes;
}
```

### 3.4.4 cgroup_set_turbo_task

此函数则是task_turbo.c的内容，主要逻辑就是根据进程P来判断是否加入turbo_list中。

```C++
// F:\Xiaomi_Kernel_OpenSource-selene-r-oss\drivers\misc\mediatek\task_turbo\task_turbo.c
void cgroup_set_turbo_task(struct task_struct *p)
{
    /* if group stune of top-app */
    if (get_st_group_id(p) == TOP_APP_GROUP_ID) { //对应/dev/stune/top-app
        if (!cgroup_check_set_turbo(p)) //p不是turbo线程并且p是render线程并且p是主线程才执行，，主线程不一定是ui线程啊！##############
            return;
        add_turbo_list(p);
    } else { /* other group */
        if (p->turbo)
            remove_turbo_list(p);
    }
}

// F:\Xiaomi_Kernel_OpenSource-selene-r-oss\drivers\misc\mediatek\task_turbo\task_turbo.c
static inline bool cgroup_check_set_turbo(struct task_struct *p)
{
    if (p->turbo)
        return false;
    /* set critical tasks for UI or UX to turbo */
    return (p->render || (p == p->group_leader && p->real_parent->pid != 1));
}
```

这里的：p->render || (p == p->group_leader && p->real_parent->pid != 1

- p->render ：判断是不是渲染线程，sys_set_turbo_task函数中赋值
- p == p->group_leader：判断是不是主线程
- p->real_parent->pid != 1：判断父进程不是 `init` 进程，这里可能是用于判断父进程是否退出。

> 例如在进程退出后，如果其父进程已经退出，那么 `init` 进程将成为其实际父进程。

### 3.4.5 sys_set_turbo_task

设置系统调用，将指定进程设置为turbo task。

```C++
// F:\Xiaomi_Kernel_OpenSource-selene-r-oss\drivers\misc\mediatek\task_turbo\task_turbo.c
// 名字是"RenderThread"，并且launch_turbo_enable是使能的
// 并且是top-app分组中的任务才设置p->render，设置后立即触发调度
extern void sys_set_turbo_task(struct task_struct *p){
        if (strcmp(p->comm, RENDER_THREAD_NAME))
                return;
        if (!launch_turbo_enable())
                return;
        if (get_st_group_id(p) != TOP_APP_GROUP_ID)
                return;
        p->render = 1;
        add_turbo_list(p);
}
```

该函数定义在SYSCALL_DEFINE5中：

```C++
SYSCALL_DEFINE5(prctl, int, option, unsigned long, arg2, unsigned long, arg3,
                unsigned long, arg4, unsigned long, arg5)
{
    ......
    switch()
        ......
        case PR_SET_NAME:
            comm[sizeof(me->comm) - 1] = 0;
            if (strncpy_from_user(comm, (char __user *)arg2,
                                  sizeof(me->comm) - 1) < 0)
                    return -EFAULT;
            set_task_comm(me, comm);
            proc_comm_connector(me);
#ifdef CONFIG_MTK_TASK_TURBO
            sys_set_turbo_task(me);
#endif
            break;
    ......
}
```

## 3.5 补充：task_turbo和cgroup接口的注册

### 3.5.1 cgroup接口的注册  

在内核的启动时，会调用start_kernel方法，在该方法中会初始化cgroup，在cgroup_init中注册了对各个节点的读写方法。

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=NDA4MThlYmVkM2ZiODcxMDY4Mzc2YjczY2RmNzA3M2RfemNtd3hFV2dMUWJTZXRESDVnTkdVdHpTWjM5WkM0c2JfVG9rZW46TXhQWmJlWnk3b1V1Q0x4RFZSbmNpRFM5bldmXzE3MDQ1MzEzMzQ6MTcwNDUzNDkzNF9WNA)

具体的注册流程：

```C++
// F:\Xiaomi_Kernel_OpenSource-selene-r-oss\init\main.c
asmlinkage __visible void __init start_kernel(void){
    ...
    cgroup_init();
    ...
}

// F:\Xiaomi_Kernel_OpenSource-selene-r-oss\kernel\cgroup\cgroup.c
int __init cgroup_init(void) {
    ...
    BUG_ON(cgroup_init_cftypes(NULL, cgroup_base_files));
    BUG_ON(cgroup_init_cftypes(NULL, cgroup1_base_files));
    ...
}

// F:\Xiaomi_Kernel_OpenSource-selene-r-oss\kernel\cgroup\cgroup-v1.c
/* cgroup core interface files for the legacy hierarchies */
struct cftype cgroup1_base_files[] = {
    {
        .name = "cgroup.procs",
        .seq_start = cgroup_pidlist_start,
        .seq_next = cgroup_pidlist_next,
        .seq_stop = cgroup_pidlist_stop,
        .seq_show = cgroup_pidlist_show,
        .private = CGROUP_FILE_PROCS,
        .write = cgroup1_procs_write,
    },
    {
        .name = "tasks",
        .seq_start = cgroup_pidlist_start,
        .seq_next = cgroup_pidlist_next,
        .seq_stop = cgroup_pidlist_stop,
        .seq_show = cgroup_pidlist_show,
        .private = CGROUP_FILE_TASKS,
        .write = cgroup1_tasks_write,
    }
}
```

## 3.6 处理继承关系

看来目前只支持 rwsem 和 binder 两种继承。 rwsem继承和binder继承的也不会写入到 turbo_pid[8] 数组中的。

### 3.6.1 binder继承

binder中的继承和取消继承：

```C++
static bool binder_proc_transaction(struct binder_transaction *t, struct binder_proc *proc, struct binder_thread *thread)
{
    if (thread) {
        if (binder_start_turbo_inherit(t->from ? t->from->task : NULL, thread->task))
            t->inherit_task = thread->task; //binder_transaction中还建了一个结构
    }
}

static void binder_transaction(struct binder_proc *proc, struct binder_thread *thread,
    struct binder_transaction_data *tr, int reply, binder_size_t extra_buffers_size)
{
    ...
    t->inherit_task = NULL;
    ...
    if (reply) {
        if (thread->task && in_reply_to->inherit_task == thread->task) {
            binder_stop_turbo_inherit(thread->task); //停止继承
            in_reply_to->inherit_task = NULL;
        }
    }
    ...
    if (in_reply_to) {
        if (thread->task && in_reply_to->inherit_task == thread->task) {
            binder_stop_turbo_inherit(thread->task);
            in_reply_to->inherit_task = NULL;
        }
    }
    ...

}

static int binder_thread_read(struct binder_proc *proc, struct binder_thread *thread,
    binder_uintptr_t binder_buffer, size_t size, binder_size_t *consumed, int non_block)
{
    ...
    if (wait_for_proc_work) {
        binder_stop_turbo_inherit(current);
    }
    ...
    if (t_from) {
        if (binder_start_turbo_inherit(t_from->task, thread->task))
            t->inherit_task = thread->task;
    }
    ...
}
```

### 3.6.2  rwsem 继承

```C++
cgroup1_base_files.write //回调，对应文件"cgroup.procs"
    cgroup1_procs_write
        __cgroup1_procs_write(of, buf, nbytes, off, true); //cgroup_v1.c

cgroup1_base_files.write //回调，对应文件"tasks"
    cgroup1_tasks_write
        __cgroup1_procs_write(of, buf, nbytes, off, false); //cgroup_v1.c


static ssize_t __cgroup1_procs_write(struct kernfs_open_file *of, char *buf, size_t nbytes, loff_t off, bool threadgroup)
{
    ret = cgroup_attach_task(cgrp, task, threadgroup); //成功返回0
    if (!ret)
        cgroup_set_turbo_task(task);
}

void cgroup_set_turbo_task(struct task_struct *p)
{
    /* if group stune of top-app */
    if (get_st_group_id(p) == TOP_APP_GROUP_ID) { //对应/dev/stune/top-app
        if (!cgroup_check_set_turbo(p)) //p不是turbo线程并且p是render线程并且p是主线程才执行，，主线程不一定是ui线程啊！##############
            return;
        add_turbo_list(p);
    } else { /* other group */
        if (p->turbo)
            remove_turbo_list(p);
    }
}

static inline bool cgroup_check_set_turbo(struct task_struct *p)
{
    if (p->turbo)
        return false;
    /* set critical tasks for UI or UX to turbo */
    return (p->render || (p == p->group_leader && p->real_parent->pid != 1));
}
```

## 3.7 futex：对等待队列操作的优化

用户空间锁虽然没有继承，但是应该是加了优先唤醒优化。

```C++
// F:\Xiaomi_Kernel_OpenSource-selene-r-oss\kernel\futex.c
static inline void __queue_me(struct futex_q *q, struct futex_hash_bucket *hb)
{
        int prio;
        /*
        用于注册此元素的优先级是实时线程（即优先级低于MAX_RT_PRIO的线程）的实际线程优先级，
        或者是非RT线程的MAX_RT_ PRIO。因此，所有RT线程按优先级顺序首先被唤醒，
        其他线程按FIFO顺序最后被唤醒。
         */
        prio = min(current->normal_prio, MAX_RT_PRIO);

        plist_node_init(&q->list, prio);
#ifdef CONFIG_MTK_TASK_TURBO
        futex_plist_add(q, hb);
#else
        plist_add(&q->list, &hb->chain);
#endif
        q->task = current;
}
```

futex_plist_add方法

用于将一个 `struct futex_q` 结构的节点添加到 `struct futex_hash_bucket` 结构的等待链表中。

```C++
inline void futex_plist_add(struct futex_q *q, struct futex_hash_bucket *hb)
{
    struct futex_q *this, *next;
    struct plist_node *current_node = &q->list;
    struct plist_node *this_node;

    // 如果不启用特殊功能 SUB_FEAT_LOCK 并且当前不是 turbo 任务，直接添加到等待链表
    if (!sub_feat_enable(SUB_FEAT_LOCK) &&
        !is_turbo_task(current)) {
        plist_add(&q->list, &hb->chain);
        return;
    }

    // 遍历等待链表
    plist_for_each_entry_safe(this, next, &hb->chain, list) {
        // 如果当前节点没有 pi_state 或者 rt_waiter，并且不是 turbo 任务，执行以下操作
        if ((!this->pi_state || !this->rt_waiter) &&
            !is_turbo_task(this->task)) {
            this_node = &this->list;
            // 将当前节点插入到当前节点的前面
            list_add(&current_node->node_list, this_node->node_list.prev);
            return;
        }
    }

    // 如果以上条件都不满足，则直接添加到等待链表
    plist_add(&q->list, &hb->chain);
}
```

# 4. 起作用的位置

core.c 中检索 turbo 得到的，没啥作用，就是设置优先级时触发重新调度。还是要看下面成员的使用：

- p->turbo //主要用来标记的
- p->inherit_types //和 binder/sem 中的继承有关
- p->cpu_prefer //在最终的选核结果上控制从哪个cluster再选一次
- p->render //设置任务comm路径中经过判断设置下来的，见上面

整个 task_turbo feature 只是对上大核比较激进，并且有binder、resem继承，cgroup top-app分组名为 RenderThread 的线程进行设置。

> 只是迁核，没有涉及对task的util的更新，这块可以做一下，尤其是针对 RenderThread 的，方便做一些。

## 4.1 p->turbo 和 p->inherit_types

主要是上面看到的地方使用了，一些binder继承，rwsem继承，futex持锁优先唤醒等。主要是在 is_turbo_task() 中一起判断的，这个函数除了继承使用外，还有：

### 4.1.1 cache

不限制turbo_task对L1和L2 cache的使用，默认是限制bg任务对cache的使用的。

```C++
static inline bool is_important(struct task_struct *task)
{
    int grp_id = get_stune_id(task);

#ifdef CONFIG_MTK_TASK_TURBO
    if (ctl_turbo_group && is_turbo_task(task))
        return true;
#endif
    if (ctl_suppress_group & (1 << grp_id))
        return false;
    return true;
}
调用路径：
__schedule //core.c
    context_switch
        prepare_task_switch
            hook_ca_context_switch //cache_ctrl.c
                restrict_next_task
                    audit_next_task //判断下面函数返回true直接返回false
                        is_important
```

### 4.1.2 创建子进程

创建子进程，若是 turbo_task 子进程不继承 cpu_prefer 属性。

```C++
int sched_fork(unsigned long clone_flags, struct task_struct *p) //core.c
{
    ...
    p->prio = current->normal_prio;
#ifdef CONFIG_MTK_TASK_TURBO
    if (unlikely(is_turbo_task(current)))
        set_user_nice(p, current->nice_backup); //不污染子进程的prio
#endif
    ...
#ifdef CONFIG_MTK_SCHED_BOOST
    p->cpu_prefer = current->cpu_prefer;
#ifdef CONFIG_MTK_TASK_TURBO
    if (unlikely(is_turbo_task(current)))
        p->cpu_prefer = 0; // SCHED_PREFER_NONE，RenderThread若是创建子线程了，子线程不继承
#endif
#endif
    ...
}
```

## 4.2 p->cpu_prefer

### 4.2.1 sched_fork

由 sched_fork() 可知，fork新进程时 p->cpu_prefer 会被继承，但是 turbo_task 的不会继承。

### 4.2.2 fbt_set_task_policy

fbt_cpu.c 中 fbt_set_task_policy(0 也会设置 p->cpu_prefer）

没看懂这里有什么用？fbt_set_task_policy在那里被调用？这些参数的含义是什么？

```C++
fbt_set_task_policy(fl, llf_task_policy, FPSGO_PREFER_LITTLE, 0)
```

### 4.2.3 select_task_prefer_cpu 

在 select_task_prefer_cpu 中会影响从哪个cluster开始选核，是在之前选核的基础上选核的。

这里是在select_task_rq_fair函数中（我实际从select_task_prefer_cpu看到了修改但是不确定在select_task_rq_fair函数中是否有客制化）进行了客制化的逻辑，使用#ifdef CONFIG_MTK_TASK_TURBO的方式进行加入业务逻辑。

```C++
 fair_sched_class.select_task_rq
        select_task_rq_fair //fair.c
            select_task_prefer_cpu_fair //fair.c 待加上对中核的支持
            select_task_rq_rt //rt.c MTK CONFIG_MTK_SCHED_BOOST 加的，直接在结果上更改，是否合理还有待考证
                select_task_prefer_cpu(struct task_struct *p, int new_cpu) //ched_ctl.c
```

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=MWQ4MDJiZDQ4ZWIzZWI3NDI0OTdlYmMxZjM4ZTllODlfcmh3c2tUcTlGRWZWblNnejJWdWxGeEtjNjBFc1ZlYTFfVG9rZW46RDlZbGJLRGpnb1dHTFl4V25LVGN1MVZ3bk9iXzE3MDQ1MzEzMzQ6MTcwNDUzNDkzNF9WNA)

### 4.2.4 store_sched_boost

没搞清楚这里为什么重点写了一下store_sched_boost函数。

注：sched boost为所有任务大核优先，4.19上新增的功能，5.10后被移除，并不是所属于task_turbo的feature中。

```C++
// sched_ctl.h
// 取值范围：
enum {
    SCHED_NO_BOOST = 0,
    SCHED_ALL_BOOST,
    SCHED_FG_BOOST,
    SCHED_UNKNOWN_BOOST
};
//设置路径：
/sys/devices/system/cpu/sched/sched_boost 对应的设置文件
    store_sched_boost //sched_ctl.c
        set_sched_boost
            sched_boost_type = val;
```

一是perfer小核的关键任务取消perfer小核的特性。

二是在通过 sched_boost 文件进行boost的情况下，前台和top-app分组运行3ms就改为从中核开始选核，其它分组直接从小核开始选核。

### 4.2.5 设置perfer总结

**/sys/module/task_turbo/parameters/turbo_pid**

 //设置前要先设置好同目录下的feats文件，CONFIG_MTK_TASK_TURBO的接口 **/sys/devices/system/cpu/sched/cpu_prefer**

 //CONFIG_MTK_SCHED_BOOST的接口，设置不检查feats文件，直接设置，只是单纯的设置p->cpu_prefer，不涉及task_turbo的其它feature

注：sched_ctl.c导出,echo <pid> <n> > cpu_prefer,n取值none=0,big=1,lit=2,med=3，直接设置到p->perfer_cpu,cfs和rt选核路径中在选核结果上select_task_prefer_cpu()中使用perfer_cpu属性重新选核.

### 4.2.6 清除perfer总结

/sys/module/task_turbo/parameters/unset_turbo_pid //只复位一个任务的设置，CONFIG_MTK_TASK_TURBO的接口

/sys/module/task_turbo/parameters/feats //写为0复位turbo_pid[8]中所有的任务，CONFIG_MTK_TASK_TURBO的接口

### 4.2.7 task_turbo 相关trace

```C++
sys_set_turbo_task //设置p->render
    set_turbo_task
    add_turbo_list
        trace_turbo_set

sched_set_cpuprefer
    trace_sched_set_cpuprefer(p); //trace选核倾向的cluster

set_user_nice
    trace_sched_set_user_nice

set_scheduler_tuning
    trace_sched_turbo_nice_set

rwsem_start_turbo_inherit
binder_start_turbo_inherit
    trace_turbo_inherit_start(from, to);

rwsem_stop_turbo_inherit
    trace_turbo_inherit_end(current);
```

# 5. 调用逻辑

## 5.1 内核驱动的注册

总体而言，这段代码的作用是注册一个名为 `feats` 的模块参数，用于控制任务加速特性。该参数的值将被存储在 `task_turbo_feats` 变量中，可以通过内核模块的参数传递或者在加载模块时手动设置。此外，通过添加描述信息，提高了代码的可读性和用户友好性。

```C++
param_check_uint(feats, &task_turbo_feats);
module_param_cb(feats, &task_turbo_feats_param_ops, &task_turbo_feats, 0644);
MODULE_PARM_DESC(feats, "enable task turbo features if needed");
```

1. `param_check_uint(feats, &task_turbo_feats);`
   1. 这个宏或函数的作用是检查模块参数 `feats` 的值是否是一个无符号整数（uint），并将其值存储在 `task_turbo_feats` 中。可能是一个自定义的宏或函数，用于处理参数的验证和存储。
2. `module_param_cb(feats, &task_turbo_feats_param_ops, &task_turbo_feats, 0644);`
   1. 这个宏用于将一个模块参数 `feats` 注册到内核，并指定一个回调函数集合 `task_turbo_feats_param_ops` 来处理参数的设置和获取。参数的值将被存储在 `task_turbo_feats` 中。第四个参数 `0644` 是权限位，表示该参数的读写权限。
3. `MODULE_PARM_DESC(feats, "enable task turbo features if needed");`
   1. 这个宏用于为模块参数 `feats` 添加一个描述信息，描述了该参数的作用。在加载模块时，这个描述信息可能会被显示出来，以帮助用户理解该参数的用途。

## 5.2 Feats的读写的定义

```C++
static struct kernel_param_ops task_turbo_feats_param_ops = {
        .set = set_task_turbo_feats,
        .get = param_get_uint,
};
```

## 5.3 Feats的写 -- set_task_turbo_feats

作者的话：

```C++
static int set_task_turbo_feats(const char *buf, const struct kernel_param *kp)
{
        int ret = 0, i;
        unsigned int val;

        ret = kstrtouint(buf, 0, &val);


        mutex_lock(&TURBO_MUTEX_LOCK);
        if (val == latency_turbo || val == launch_turbo  || val == 0)
                ret = param_set_uint(buf, kp);
        else
                ret = -EINVAL;

        /* if disable turbo, remove all turbo tasks */
        /* mutex_lock(&TURBO_MUTEX_LOCK); */
        if (val == 0) {
                for (i = 0; i < TURBO_PID_COUNT; i++) {
                        if (turbo_pid[i]) {
                                unset_turbo_task(turbo_pid[i]);
                                turbo_pid[i] = 0;
                        }
                }
        }
        mutex_unlock(&TURBO_MUTEX_LOCK);

        if (!ret)
                pr_info("task_turbo_feats is change to %d successfully",
                                task_turbo_feats);
        return ret;
}
```

### 5.3.1 补充：kernel_param 

`kernel_param` 是 Linux 内核中用于处理模块参数的数据结构。这个结构体用于描述模块参数的属性，包括参数的名称、类型以及操作函数等。

在模块中定义参数时，通常会使用 `module_param` 或类似的宏，这些宏会生成一个 `kernel_param` 结构体，并将其添加到一个全局的模块参数数组中。这个数组在模块加载时会被内核用于解析命令行传递的参数。

```C++
// F:\Xiaomi_Kernel_OpenSource-selene-r-oss\include\linux\moduleparam.h
struct kernel_param {
        const char *name;
        struct module *mod;
        const struct kernel_param_ops *ops;
        const u16 perm;
        s8 level;
        u8 flags;
        union {
                void *arg;
                const struct kparam_string *str;
                const struct kparam_array *arr;
        };
};
```

### 5.3.2 补充：kstrtouint函数

`kstrtouint` 是 Linux 内核中用于将字符串转换为无符号整数（unsigned int）的函数。这个函数的作用是将一个表示整数的字符串转换为相应的无符号整数值，并将结果存储在传入的变量中。

```C++
int kstrtouint(const char *s, unsigned int base, unsigned int *res);
```

- `s`：要转换的字符串。
- `base`：进制，可以是 2、8、10 或 16。如果是 0，则根据字符串的前缀来确定进制（例如，0x 表示十六进制，0 表示八进制，没有前缀表示十进制）。
- `res`：用于存储结果的无符号整数指针。

函数返回值是一个整数，表示转换的结果。如果转换成功，返回 0；如果发生错误，返回相应的错误代码。

## 5.4 Feats的读--param_get_uint

在 Linux 内核中，`param_get_uint` 函数通常用于获取指定模块参数的无符号整数值。这个函数主要用于处理模块参数，这些参数在模块加载时可以通过命令行传递，或者在内核源代码中进行硬编码。

```C++
extern int param_get_uint(char *buffer, const struct kernel_param *kp);

// 补充
extern int param_set_uint(const char *val, const struct kernel_param *kp);
```

# 6. 实测

## 6.1 测试 CONFIG_MTK_TASK_TURBO

```C++
/sys/module/task_turbo/parameters # ps -AT | grep system_server
system        1565  1565   795 23058820 491980 SyS_epoll_wait     0 S system_server

/sys/module/task_turbo/parameters # echo 15 > feats //必须先执行
/sys/module/task_turbo/parameters # echo 1565 > turbo_pid
/sys/module/task_turbo/parameters # echo 1565 > unset_turbo_pid //抓trace后清理设置，若将feats文件设置为0清理全部
```

task_turbo 选到 cpu7 上，需满足 cpu7 是online的、非isolated状态的、是在任务的cpus_allow里面的、此时idle不idle都行。一旦cpu7没选上，task_turbo 的 cpu_prefer 特性就不参与选核了。

结果：1565 虽然跑的少，每次运行时间短，但是运行在大核上。

结论：跑在大核上，符合预期。

## 6.2 测试 CONFIG_MTK_SCHED_BOOST

```C++
/sys/devices/system/cpu/sched # ps -AT | grep system_server
system        1540  1540   809 23600516 456832 SyS_epoll_wait     0 S system_server

/sys/devices/system/cpu/sched # echo 1540 1 > cpu_prefer //cpu prefer大核

// 清理设置：
/sys/devices/system/cpu/sched # echo 1540 0 > cpu_prefer
```

结果：

a. system_server 虽然跑的很少，每次跑的也很短，但是每次都能跑在大核上，符合预期。

b. 写3从中核开始选的一致性要差一些，但是绝大多数情况下也是在中核上。

结论：从哪个cluster开始选核，就大概率会选择上哪个cluster的CPU。

## 6.3 sched_boost 情况下验证自己加的过滤策略

```C++
/sys/devices/system/cpu/sched # echo 1 > sched_boost //使能自己对前后台区分的过滤
/sys/devices/system/cpu/sched # let i=0; while true; do if [ i -lt 100 ]; then let i=i+1; else let i=0; sleep 0.01; fi; done &
[1] 16018
# echo 16018 > /dev/stune/top-app/tasks
# echo 15 > /sys/module/task_turbo/parameters/feats       //必须先执行
# echo 16018 > /sys/module/task_turbo/parameters/turbo_pid
/dev/stune/top-app # let i=0; while true; do if [ i -lt 100 ]; then let i=i+1; else let i=0; sleep 0.01; fi; done &
[2] 31467
/dev/stune/top-app # echo 31467 > /sys/module/task_turbo/parameters/turbo_pid
/dev/stune/top-app # echo 31467 > /dev/stune/background/tasks
```

结果：16018 跑大核CPU7，31467 跑小核，符合预期。只有使能了 sched_boost，自己加的这个过滤才生效，到时候功耗差的话，可以全局打开。

## 6.4 补充实验

既然是死循环计数，就可以用来验证算力，可以将大核和小核都定到最大频点对比一下

16018: 1.7ms，频点2G

31467: 4ms，频点3G

结论：同频点下算力差：4/1.7/(3/2) = 1.57 倍，也即是同频点下大核比小核快1.57倍。看 cpu_capacity 文件的值再这算也行。

## 6.5 补充实验：验证cgroup-attach TOP APP

注：在5.x版本上此逻辑被移除，故此在4.19的设备上进行实验，目的为验证在前台应用发生变化时，将前台进程写入cgroup.procs以及tasks节点。

**实验：**在前台启动应用安兔兔（或者其他应用），验证``/dev/stune/top-app/cgroup.procs``是否发生变化。

**验证：**

```C++
/dev> adb shell cat /dev/stune/top-app/cgroup.procs  // 启动应用前
538
542
760
842
1257
/dev> adb shell cat /dev/stune/top-app/cgroup.procs  // 启动应用后
538
542
760
842
1257
3855
```

**补充：**

```C++
/dev> adb shell ps | grep antutu
u0_a115       3855   538 50775628 314164 do_epoll_wait      0 S com.antutu.ABenchMark
```

结论：

在应用启动时，当前台变为