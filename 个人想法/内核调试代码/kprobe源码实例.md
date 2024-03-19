# Linux内核调试技术——kprobe源码实例

https://blog.csdn.net/liyongming1982/article/details/16362811

# kprobe_example

1. ## 背景

1. ## 代码详情

```C
// SPDX-License-Identifier: GPL-2.0-only
/*
 * Here's a sample kernel module showing the use of kprobes to dump a
 * stack trace and selected registers when kernel_clone() is called.
 *
 * For more information on theory of operation of kprobes, see
 * Documentation/trace/kprobes.rst
 *
 * You will see the trace data in /var/log/messages and on the console
 * whenever kernel_clone() is invoked to create a new process.
 */

#define pr_fmt(fmt) "%s: " fmt, __func__

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>

static char symbol[KSYM_NAME_LEN] = "kernel_clone";
module_param_string(symbol, symbol, KSYM_NAME_LEN, 0644);

/* For each probe you need to allocate a kprobe structure */
static struct kprobe kp = {
        .symbol_name        = symbol,
};

/* kprobe pre_handler: called just before the probed instruction is executed */
static int __kprobes handler_pre(struct kprobe *p, struct pt_regs *regs)
{
#ifdef CONFIG_X86
        pr_info("<%s> p->addr = 0x%p, ip = %lx, flags = 0x%lx\n",
                p->symbol_name, p->addr, regs->ip, regs->flags);
#endif
    ......
#ifdef CONFIG_ARM64
        pr_info("<%s> p->addr = 0x%p, pc = 0x%lx, pstate = 0x%lx\n",
                p->symbol_name, p->addr, (long)regs->pc, (long)regs->pstate);
#endif
#ifdef CONFIG_ARM
        pr_info("<%s> p->addr = 0x%p, pc = 0x%lx, cpsr = 0x%lx\n",
                p->symbol_name, p->addr, (long)regs->ARM_pc, (long)regs->ARM_cpsr);
#endif

        /* A dump_stack() here will give a stack backtrace */
        return 0;
}

/* kprobe post_handler: called after the probed instruction is executed */
static void __kprobes handler_post(struct kprobe *p, struct pt_regs *regs,
                                unsigned long flags)
{
#ifdef CONFIG_X86
        pr_info("<%s> p->addr = 0x%p, flags = 0x%lx\n",
                p->symbol_name, p->addr, regs->flags);
#endif
    ......
#ifdef CONFIG_ARM64
        pr_info("<%s> p->addr = 0x%p, pstate = 0x%lx\n",
                p->symbol_name, p->addr, (long)regs->pstate);
#endif
#ifdef CONFIG_ARM
        pr_info("<%s> p->addr = 0x%p, cpsr = 0x%lx\n",
                p->symbol_name, p->addr, (long)regs->ARM_cpsr);
#endif
}

static int __init kprobe_init(void)
{
        int ret;
        kp.pre_handler = handler_pre;
        kp.post_handler = handler_post;

        ret = register_kprobe(&kp);
        if (ret < 0) {
                pr_err("register_kprobe failed, returned %d\n", ret);
                return ret;
        }
        pr_info("Planted kprobe at %p\n", kp.addr);
        return 0;
}

static void __exit kprobe_exit(void)
{
        unregister_kprobe(&kp);
        pr_info("kprobe at %p unregistered\n", kp.addr);
}

module_init(kprobe_init)
module_exit(kprobe_exit)
MODULE_LICENSE("GPL");
```

1. ## 代码解析

1. ### 引入相关库函数与初始化部分数据结构

```C++
#define pr_fmt(fmt) "%s: " fmt, __func__

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>

static char symbol[KSYM_NAME_LEN] = "kernel_clone";
module_param_string(symbol, symbol, KSYM_NAME_LEN, 0644);
```

1. ### 内核函数的初始化与结束

```C++
static int __init kprobe_init(void)
{
        int ret;
        kp.pre_handler = handler_pre;
        kp.post_handler = handler_post;

        ret = register_kprobe(&kp);
        if (ret < 0) {
                pr_err("register_kprobe failed, returned %d\n", ret);
                return ret;
        }
        pr_info("Planted kprobe at %p\n", kp.addr);
        return 0;
}

static void __exit kprobe_exit(void)
{
        unregister_kprobe(&kp);
        pr_info("kprobe at %p unregistered\n", kp.addr);
}

module_init(kprobe_init)
module_exit(kprobe_exit)
MODULE_LICENSE("GPL");
```

1. ### 实现kretprobe结构体

```C++
static struct kprobe kp = {
        .symbol_name        = symbol,
};
```

1. ### 实现处理函数

```C++
/* kprobe pre_handler: 在被探测指令执行之前调用 */
static int __kprobes handler_pre(struct kprobe *p, struct pt_regs *regs)
{
#ifdef CONFIG_X86
        pr_info("<%s> p->addr = 0x%p, ip = %lx, flags = 0x%lx\n",
                p->symbol_name, p->addr, regs->ip, regs->flags);
#endif
    ......
#ifdef CONFIG_ARM64
        pr_info("<%s> p->addr = 0x%p, pc = 0x%lx, pstate = 0x%lx\n",
                p->symbol_name, p->addr, (long)regs->pc, (long)regs->pstate);
#endif
#ifdef CONFIG_ARM
        pr_info("<%s> p->addr = 0x%p, pc = 0x%lx, cpsr = 0x%lx\n",
                p->symbol_name, p->addr, (long)regs->ARM_pc, (long)regs->ARM_cpsr);
#endif

        /* A dump_stack() here will give a stack backtrace */
        return 0;
}

/* kprobe post_handler: 在被探测指令执行后调用 */
static void __kprobes handler_post(struct kprobe *p, struct pt_regs *regs,
                                unsigned long flags)
{
#ifdef CONFIG_X86
        pr_info("<%s> p->addr = 0x%p, flags = 0x%lx\n",
                p->symbol_name, p->addr, regs->flags);
#endif
    ......
#ifdef CONFIG_ARM64
        pr_info("<%s> p->addr = 0x%p, pstate = 0x%lx\n",
                p->symbol_name, p->addr, (long)regs->pstate);
#endif
#ifdef CONFIG_ARM
        pr_info("<%s> p->addr = 0x%p, cpsr = 0x%lx\n",
                p->symbol_name, p->addr, (long)regs->ARM_cpsr);
#endif
}
```

# kretprobe_example

> 代码来源：kernel_platform/msm-kernel/samples/kprobes/kretprobe_example.c 

1. ## 背景

下面是一个示例内核模块，展示了如何使用返回探测来报告返回值和被探测函数运行所需的总时间。

1. ## 用法

用法：insmod kretprobe_example.ko func=<func_name>

如果未指定func_name，则检测kernel_clone

1. ## 说明

有关kretprobes操作理论的更多信息，请参阅文档/跟踪/kprobes.rst

按照kprobe示例中的操作构建并插入内核模块。

您将在/var/log/messages和控制台上看到跟踪数据

每当探测的函数返回时。（ 如果syslogd配置为消除重复消 某些消息可能会被抑制。）

1. ## 代码详情

```C
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/ktime.h>
#include <linux/sched.h>

static char func_name[KSYM_NAME_LEN] = "kernel_clone";
module_param_string(func, func_name, KSYM_NAME_LEN, 0644);
MODULE_PARM_DESC(func, "Function to kretprobe; this module will report the"
                        " function's execution time");

/* per-instance private data */
struct my_data {
        ktime_t entry_stamp;
};

/* Here we use the entry_hanlder to timestamp function entry */
static int entry_handler(struct kretprobe_instance *ri, struct pt_regs *regs)
{
        struct my_data *data;

        if (!current->mm)
                return 1;        /* Skip kernel threads */

        data = (struct my_data *)ri->data;
        data->entry_stamp = ktime_get();
        return 0;
}
NOKPROBE_SYMBOL(entry_handler);

/*
返回探测处理程序：记录返回值和持续时间。
持续时间可能始终为零，这取决于平台上时间核算的粒度。
 */
static int ret_handler(struct kretprobe_instance *ri, struct pt_regs *regs)
{
        unsigned long retval = regs_return_value(regs);
        struct my_data *data = (struct my_data *)ri->data;
        s64 delta;
        ktime_t now;

        now = ktime_get();
        delta = ktime_to_ns(ktime_sub(now, data->entry_stamp));
        pr_info("%s returned %lu and took %lld ns to execute\n",
                        func_name, retval, (long long)delta);
        return 0;
}
NOKPROBE_SYMBOL(ret_handler);

static struct kretprobe my_kretprobe = {
        .handler                = ret_handler,
        .entry_handler                = entry_handler,
        .data_size                = sizeof(struct my_data),
        /* Probe up to 20 instances concurrently. */
        .maxactive                = 20,
};

static int __init kretprobe_init(void)
{
        int ret;

        my_kretprobe.kp.symbol_name = func_name;
        ret = register_kretprobe(&my_kretprobe);
        if (ret < 0) {
                pr_err("register_kretprobe failed, returned %d\n", ret);
                return ret;
        }
        pr_info("Planted return probe at %s: %p\n",
                        my_kretprobe.kp.symbol_name, my_kretprobe.kp.addr);
        return 0;
}

static void __exit kretprobe_exit(void)
{
        unregister_kretprobe(&my_kretprobe);
        pr_info("kretprobe at %p unregistered\n", my_kretprobe.kp.addr);

        /* nmissed > 0 suggests that maxactive was set too low. */
        pr_info("Missed probing %d instances of %s\n",
                my_kretprobe.nmissed, my_kretprobe.kp.symbol_name);
}

module_init(kretprobe_init)
module_exit(kretprobe_exit)
MODULE_LICENSE("GPL");
```

1. ## 代码解析

1. ### 引入相关库函数与初始化部分数据结构

```C
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/ktime.h>
#include <linux/sched.h>

static char func_name[KSYM_NAME_LEN] = "kernel_clone";
module_param_string(func, func_name, KSYM_NAME_LEN, 0644);
MODULE_PARM_DESC(func, "Function to kretprobe; this module will report the"
                        " function's execution time");
```

1. ### 内核函数的初始化与结束

```C
static int __init kretprobe_init(void)
{
        int ret;

        my_kretprobe.kp.symbol_name = func_name;
        ret = register_kretprobe(&my_kretprobe);
        if (ret < 0) {
                pr_err("register_kretprobe failed, returned %d\n", ret);
                return ret;
        }
        pr_info("Planted return probe at %s: %p\n",
                        my_kretprobe.kp.symbol_name, my_kretprobe.kp.addr);
        return 0;
}

static void __exit kretprobe_exit(void)
{
        unregister_kretprobe(&my_kretprobe);
        pr_info("kretprobe at %p unregistered\n", my_kretprobe.kp.addr);

        /* nmissed > 0 suggests that maxactive was set too low. */
        pr_info("Missed probing %d instances of %s\n",
                my_kretprobe.nmissed, my_kretprobe.kp.symbol_name);
}

module_init(kretprobe_init)
module_exit(kretprobe_exit)
MODULE_LICENSE("GPL");
```

1. ### 实现kretprobe结构体

```C
static struct kretprobe my_kretprobe = {
        .handler                = ret_handler,
        .entry_handler                = entry_handler,
        .data_size                = sizeof(struct my_data),
        /* Probe up to 20 instances concurrently. */
        .maxactive                = 20,
};

struct my_data {
        ktime_t entry_stamp;
};
```

1. ### 实现处理函数

```C
/* Here we use the entry_hanlder to timestamp function entry */
static int entry_handler(struct kretprobe_instance *ri, struct pt_regs *regs)
{
        struct my_data *data;

        if (!current->mm)
                return 1;        /* Skip kernel threads */

        data = (struct my_data *)ri->data;
        data->entry_stamp = ktime_get();
        return 0;
}
NOKPROBE_SYMBOL(entry_handler);

/*
返回探测处理程序：记录返回值和持续时间。
持续时间可能始终为零，这取决于平台上时间核算的粒度。
 */
static int ret_handler(struct kretprobe_instance *ri, struct pt_regs *regs)
{
        // 不理解:关于regs参数需要再确认下
        unsigned long retval = regs_return_value(regs);
        // 一个指向my_data的指针,可以通过instance结构体进行获得,实现操作数据结构的能力
        struct my_data *data = (struct my_data *)ri->data;
        s64 delta;
        ktime_t now;
        
        // 内核函数,获取时间
        now = ktime_get();
        // 计算当前时间和之前进入函数的时间差值,并转化为ns值
        delta = ktime_to_ns(ktime_sub(now, data->entry_stamp));
        pr_info("%s returned %lu and took %lld ns to execute\n",
                        func_name, retval, (long long)delta);
        return 0;
}
NOKPROBE_SYMBOL(ret_handler);
```