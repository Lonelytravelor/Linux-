# Linux内核调试技术——kretprobe使用与实现详解

> 参考文档：https://blog.csdn.net/luckyapple1028/article/details/54782659

前两篇博文介绍了kprobes探测技术中kprobe和jprobe的使用与实现。本文介绍kprobes中的最后一种探测技术kretprobe，它同样基于kprobe实现，可用于探测函数的返回值以及计算函数执行的耗时。本文首先通过一个简单的示例程序介绍kretprobe的使用方式，然后通过源码分析它是如何实现的。

内核源码：Linux-4.1.x

# 1.kretprobe使用示例

使用kretprobe探测函数的返回值同jprobe一样需要编写内核模块，当然内核也提供了一个简单的示例程序kretprobe_example.c（位于sample/kprobes目录），该程序的实现更为通用，用户可以在使用时直接通过模块参数指定需要探测的函数，它在探测函数返回值的同时还统计了被探测函数的执行用时。在分析kretprobe_example.c之前先熟悉一下kretprobe的结构体定义和API接口。

## 1.1 kretprobe结构体与API介绍

### 1.1.1 kretprobe结构体

```C
/*
Function-return probe -
Note:
User needs to provide a handler function, and initialize maxactive.
maxactive - The maximum number of instances of the probed function that
can be active concurrently.
nmissed - tracks the number of times the probed function's return was
ignored, due to maxactive being too low.
 *
 */
struct kretprobe {
        struct kprobe kp;
        kretprobe_handler_t handler;
        kretprobe_handler_t entry_handler;
        int maxactive;
        int nmissed;
        size_t data_size;
        struct hlist_head free_instances;
        raw_spinlock_t lock;
};
```

struct kretprobe结构体用于定义一个kretprobe:

- 由于它的实现基于**kprobe**，结构体中自然也少不了kprobe字段；
- **handler**和**entry_handler**分别表示两个回调函数，用户自行定义:
  - entry_handler会在被探测函数执行之前被调用
  - handler在被探测函数返回后被调用（一般在这个函数中打印被探测函数的返回值）；
- **maxactive**表示同时支持并行探测的上限:
  - 因为kretprobe会跟踪一个函数从开始到结束，因此对于一些调用比较频繁的被探测函数，在探测的时间段内重入的概率比较高;
  - 这个maxactive字段值表示在重入情况发生时，支持同时检测的进程数（执行流数）的上限，
  - 若并行触发的数量超过了这个上限，则kretprobe不会进行跟踪探测，仅仅增加nmissed字段的值以作提示；
- **data_size**字段表示kretprobe私有数据的大小，在注册kretprobe时会根据该大小预留空间；
- **free_instances**表示空闲的kretprobe运行实例链表，它链接了本kretprobe的空闲实例struct kretprobe_instance结构体表示。

### 1.1.2 kretprobe的运行实例结构体

前文说过被探测函数在跟踪期间可能存在并发执行的现象，因此kretprobe**使用kretprobe_instance来跟踪一个执行流**，支持的上限为maxactive。

```C++
struct kretprobe_instance {
        struct hlist_node hlist;
        struct kretprobe *rp;
        kprobe_opcode_t *ret_addr;
        struct task_struct *task;
        char data[0];
};
```

这个结构体表示kretprobe的运行实例，在没有触发探测时，所有的kretprobe_instance实例都保存在free_instances表中，每当有执行流触发一次kretprobe探测，都会从该表中取出一个空闲的kretprobe_instance实例用来跟踪:

- **kretprobe_instance**结构提中的rp指针指向所属的kretprobe；
- **ret_addr**用于保存原始被探测函数的返回地址（后文会看到被探测函数返回地址会被暂时替换）；
- **task**用于绑定其跟踪的进程；
- 最后**data**保存用户使用的kretprobe私有数据，它会在整个kretprobe探测运行期间在entry_handler和[handler](https://so.csdn.net/so/search?q=handler&spm=1001.2101.3001.7020)回调函数之间进行传递（一般用于实现统计被探测函数的执行耗时）。

**注意：在声明.handler等函数时，其函数原型为****``(struct kretprobe_instance \*ri, struct pt_regs \*regs)``****，其中****`kretprobe_instance`****即为运行实例，可以看当前这里存储的数据进行数据的传递。**

## 1.2 示例kretprobe_example分析与演示

内核提供的kretprobe_example.c示例程序默认探测do_fork函数的执行耗时和返回值，支持通过模块参数指定被探测函数，用户若需要探测其他函数，只需要在加载内核模块时传入自己需要探测的函数名即可，无需修改模块代码。

```C++
static char func_name[NAME_MAX] = "do_fork";
module_param_string(func, func_name, NAME_MAX, S_IRUGO);
MODULE_PARM_DESC(func, "Function to kretprobe; this module will report the"
                        " function's execution time");
```

### 1.2.1 内核模块实例

下面详细分析：

```C++
/* per-instance private data */
struct my_data {
    ktime_t entry_stamp;
};
 
static struct kretprobe my_kretprobe = {
    .handler                = ret_handler,
    .entry_handler            = entry_handler,
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
        printk(KERN_INFO "register_kretprobe failed, returned %d\n",ret);
        return -1;
    }
    printk(KERN_INFO "Planted return probe at %s: %p\n",my_kretprobe.kp.symbol_name, my_kretprobe.kp.addr);
    return 0;
}
 
static void __exit kretprobe_exit(void)
{
    unregister_kretprobe(&my_kretprobe);
    printk(KERN_INFO "kretprobe at %p unregistered\n",my_kretprobe.kp.addr);

    /* nmissed > 0 suggests that maxactive was set too low. */
    printk(KERN_INFO "Missed probing %d instances of %s\n",my_kretprobe.nmissed, my_kretprobe.kp.symbol_name);
}
```

**代码详解:**

1. 程序定义了一个结构体struct my_data，其中唯一的参数entry_stamp用于计算函数执行的时间;
2. 程序定义了一个struct kretprobe实例，注意其中的私有数据长度为my_data的长度，最大支持的并行探测数为20（即若在某一时刻，do_fork函数同时调用的执行流数量超过20那将不会再进行探测跟踪）。
3. 最后在模块的init和exit函数中仅仅调用register_kretprobe和unregister_kretprobe函数对my_kretprobe进行注册和注销，在kretprobe注册完成后就默认启动探测了。

### 1.2.2 entry_handler与ret_handler

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
 
/*
Return-probe handler: Log the return value and duration. Duration may turn
out to be zero consistently, depending upon the granularity of time
accounting on the platform.
 */
static int ret_handler(struct kretprobe_instance *ri, struct pt_regs *regs)
{
        int retval = regs_return_value(regs);
        struct my_data *data = (struct my_data *)ri->data;
        s64 delta;
        ktime_t now;
 
        now = ktime_get();
        delta = ktime_to_ns(ktime_sub(now, data->entry_stamp));
        printk(KERN_INFO "%s returned %d and took %lld ns to execute\n",
                        func_name, retval, (long long)delta);
        return 0;
}
```

**函数entry_handler在do_fork函数被调用时触发调用**，注意第一个入参不是struct kretprobe结构，而是代表一个探测实例的struct kretprobe_instance结构，它从kretprobe的free_instances链表中分配，在跟踪完本次触发流程后回收。entry_handler函数利用了kretprobe_instance中私有数据，保存do_fork函数执行的开始时间。

**函数ret_handler在do_fork函数执行完成返回后被调用**，它根据当前的时间减去kretprobe_instance中私有数据保存的起始时间，即可计算出do_fork函数执行的耗时。

同时它调用regs_return_value函数获取do_fork函数的返回值，该函数是架构相关的：

```C++
static inline long regs_return_value(struct pt_regs *regs)
{
        return regs->ARM_r0;
}
```

例如arm环境是通过r0寄存器传递返回值的，因此该函数的实现仅仅是返回r0寄存器的值（需要注意的是regs指针传递的是do_fork函数返回时所保存的寄存器信息，这一点后面会分析）。ret_handler函数最后打印出do_fork函数的返回值和执行耗时（单位ns）。

### 1.2.3 实际效果

下面在x86_64环境下演示该示例程序的实际效果（环境配置请参考《[Linux内核调试技术——kprobe使用与实现](http://blog.csdn.net/luckyapple1028/article/details/52972315)）：

```C
<6>[ 1217.859349] _do_fork returned 1838 and took 518081 ns to execute
<6>[ 1217.863880] _do_fork returned 1839 and took 223701 ns to execute
<6>[ 1217.865731] _do_fork returned 1840 and took 221746 ns to execute
<6>[ 1220.077508] _do_fork returned 1841 and took 433573 ns to execute
<6>[ 1220.081512] _do_fork returned 1842 and took 362684 ns to execute
<6>[ 1220.083767] _do_fork returned 1843 and took 284184 ns to execute
<6>[ 1220.995537] _do_fork returned 1844 and took 503414 ns to execute
<6>[ 1221.000363] _do_fork returned 1845 and took 427427 ns to execute
```

加载kretprobe_example.ko，不指定探测函数，默认探测do_fork函数，内核输出以上messages。可见do_fork函数的返回值（新创建进程的pid）是依次递增的，同时函数执行用时也呈现的非常直观。

因此，使用kretprobe可以简单的获取一个函数在执行时的返回值，在内核调试时非常有用。探测其他函数方法类似，不再赘述。

# 2.kretprobe实现分析

kretprobe的实现基于kprobe，因此这里将在前一篇博文《Linux内核调试技术——kprobe使用与实现》的基础之上分析它的实现，主要包括kretprobe注册流程和触发探测流程，涉及kprobe的部分不再详细描。

## 2.1 kretprobe实现原理

同jprobe类似，kretprobe也是一种特殊形式的kprobe，它有自己私有的pre_handler，并不支持用户定义pre_handler和post_handler等回调函数。其中它的pre_handler回调函数会为kretprobe探测函数执行的返回值做准备工作，其中最主要的就是替换掉正常流程的返回地址，让被探测函数在执行之后能够跳转到kretprobe所精心设计的函数中去，它会获取函数返回值，然后调用kretprobe->handler回调函数（被探测函数的返回地址此刻得到输出），最后恢复正常执行流程。

## 2.2 注册一个kretprobe

### 2.2.1 注册kretprobe流程

kretprobe探测模块调用register_kretprobe函数向内核注册一个kretprobe实例，代码路径为kernel/kprobes.c，其主要流程如下图：

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=YmIwNWY0NDEwZjVmYTA5OTYwNDBlMzdlMDJlOTM1NGNfWkhUaXNYcUhtV2REa2ROeUFIVXI4MDFSVmxFZEVVMUZfVG9rZW46VUppeGJlVnRpb3dIaHJ4VWl0WmNEcXJKbnFoXzE3MTA3NDU1MzA6MTcxMDc0OTEzMF9WNA)

### 2.2.2 函数的主要代码一

```C
int register_kretprobe(struct kretprobe *rp)
{
    int ret = 0;
    struct kretprobe_instance *inst;
    int i;
    void *addr;

    if (kretprobe_blacklist_size) {
            addr = kprobe_addr(&rp->kp);
        if (IS_ERR(addr))
            return PTR_ERR(addr);

        for (i = 0; kretprobe_blacklist[i].name != NULL; i++) {
            if (kretprobe_blacklist[i].addr == addr)
                return -EINVAL;
        }
    }
    ......
}
```

函数的开头首先处理kretprobe所特有的blacklist，如果指定的被探测函数在这个blacklist中就直接返回EINVAL，表示不支持探测。其中kretprobe_blacklist_size表示队列的长度，kretprobe_blacklist是一个全局结构体数组，每一项都是一个struct kretprobe_blackpoint结构体：

```C
struct kretprobe_blackpoint {
        const char *name;
        void *addr;
};
```

其中name字段表示函数名，addr表示函数的运行地址。该kretprobe_blacklist是架构相关的，用于申明该架构哪些函数是不支持使用kretprobe探测的，其中arm架构并没有被定义，而x86_64架构的定义如下：

```C++
struct kretprobe_blackpoint kretprobe_blacklist[] = {
        {"__switch_to", }, /* This function switches only current task, but
                              doesn't switch kernel stack.*/
        {NULL, NULL}        /* Terminator */
};
```

这表明在x86_64架构下的__switch_to函数不可以被kretprobe所探测（这一点在内核的kprobes.txt中已经有说明）。回到register_kretprobe函数中，在blacklist检测时比较的是函数运行地址addr字段，该字段在kprobes子系统初始化函数init_kprobes中初始化：

```C++
        if (kretprobe_blacklist_size) {
                /* lookup the function address from its name */
                for (i = 0; kretprobe_blacklist[i].name != NULL; i++) {
                        kprobe_lookup_name(kretprobe_blacklist[i].name,
                                           kretprobe_blacklist[i].addr);
                        if (!kretprobe_blacklist[i].addr)
                                printk("kretprobe: lookup failed: %s\n",
                                       kretprobe_blacklist[i].name);
                }
        }
```

### 2.2.3 函数的主要代码二

继续往下分析register_kretprobe注册函数：

```C
    rp->kp.pre_handler = pre_handler_kretprobe;
    rp->kp.post_handler = NULL;
    rp->kp.fault_handler = NULL;
    rp->kp.break_handler = NULL;

    /* Pre-allocate memory for max kretprobe instances */
    if (rp->maxactive <= 0) {
#ifdef CONFIG_PREEMPT
        rp->maxactive = max_t(unsigned int, 10, 2*num_possible_cpus());
#else
        rp->maxactive = num_possible_cpus();
#endif
    }
```

此处只指定了kprobe的pre_handler回调函数为pre_handler_kretprobe；然后若用户没有指定最大并行探测数maxactive，这里会计算并设置一个默认的值。

```C++
    raw_spin_lock_init(&rp->lock);
    INIT_HLIST_HEAD(&rp->free_instances);
    for (i = 0; i < rp->maxactive; i++) {
        inst = kmalloc(sizeof(struct kretprobe_instance) +rp->data_size, GFP_KERNEL);
        if (inst == NULL) {
            free_rp_inst(rp);
            return -ENOMEM;
        }
        INIT_HLIST_NODE(&inst->hlist);
        hlist_add_head(&inst->hlist, &rp->free_instances);
    }

    rp->nmissed = 0;
    /* Establish function entry probe point */
    ret = register_kprobe(&rp->kp);
    if (ret != 0)
        free_rp_inst(rp);
    return ret;
}
```

接下来根据maxactive的值，为各个kretprobe_instance实例分配内存并将它们链接到kretprobe的free_instances链表中，最后调用register_kprobe函数注册内嵌的kprobe。

以上就是kretprobe的注册流程，可见它同jprobe一样也非常的简单，最终依赖的依然是kprobe机制。

## 2.3 触发kretprobe探测

### 2.3.1 触发kretprobe流程

基于kprobe机制，在执行到指定的被探测函数后，会触发CPU异常，进入kprobe探测流程。

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=MWMxYmJiNjU3Yzk1MmJhZDFkMDU3ZmVhYmVlZDRjYWJfYlVXOWc0UkY1TGRJOW91cjZ4dDdjVkx4dk1MQm9wQnZfVG9rZW46RDk4OWJ5SW15b25GUFF4eHpSMWN3VkNVbjNDXzE3MTA3NDU1MzA6MTcxMDc0OTEzMF9WNA)

首先由kprobe_handler函数调用pre_handler回调函数，此处为pre_handler_kretprobe函数。

该函数首先找到一个空闲的kretprobe_instance探测实例并将它和当前进程绑定，然后调用entry_handler回调函数，接着保存并替换被探测函数的返回地址，最后kprobe探测流程结束并回到正常的执行流程执行被探测函数。

在函数返回后将跳转到被替换的kretprobe_trampoline，该函数会获取被探测函数的寄存器信息并调用用户定义的回调函数输出其中的返回值，最后函数返回正常的执行流程。

### 2.3.2 pre_handler_kretprobe详解

```C
/*
 * This kprobe pre_handler is registered with every kretprobe. When probe
 * hits it will set up the return probe.
 */
static int pre_handler_kretprobe(struct kprobe *p, struct pt_regs *regs)
{
        struct kretprobe *rp = container_of(p, struct kretprobe, kp);
        unsigned long hash, flags = 0;
        struct kretprobe_instance *ri;
 
        /*
         * To avoid deadlocks, prohibit return probing in NMI contexts,
         * just skip the probe and increase the (inexact) 'nmissed'
         * statistical counter, so that the user is informed that
         * something happened:
         */
        if (unlikely(in_nmi())) {
                rp->nmissed++;
                return 0;
        }
 
        /* TODO: consider to only swap the RA after the last pre_handler fired */
        hash = hash_ptr(current, KPROBE_HASH_BITS);
        raw_spin_lock_irqsave(&rp->lock, flags);
        if (!hlist_empty(&rp->free_instances)) {
                ri = hlist_entry(rp->free_instances.first,
                                struct kretprobe_instance, hlist);
                hlist_del(&ri->hlist);
                raw_spin_unlock_irqrestore(&rp->lock, flags);
 
                ri->rp = rp;
                ri->task = current;
 
                if (rp->entry_handler && rp->entry_handler(ri, regs)) {
                        raw_spin_lock_irqsave(&rp->lock, flags);
                        hlist_add_head(&ri->hlist, &rp->free_instances);
                        raw_spin_unlock_irqrestore(&rp->lock, flags);
                        return 0;
                }
 
                arch_prepare_kretprobe(ri, regs);
 
                /* XXX(hch): why is there no hlist_move_head? */
                INIT_HLIST_NODE(&ri->hlist);
                kretprobe_table_lock(hash, &flags);
                hlist_add_head(&ri->hlist, &kretprobe_inst_table[hash]);
                kretprobe_table_unlock(hash, &flags);
        } else {
                rp->nmissed++;
                raw_spin_unlock_irqrestore(&rp->lock, flags);
        }
        return 0;
}
```

pre_handler触发流程：

1. 首先根据当前的进程描述符地址以及KPROBE_HASH_BITS值计算出hash索引值，如果kretprobe的free_instances链表不为空，则从中找到一个空闲的kretprobe_instance实例；
2. 然后对其中的rp和task字段赋值，表示将该探测实例和当前进程绑定；
3. 然后调用entry_handler回调函数（前文kretprobe_example示例程序中的entry_handler函数在此被调用）；
4. 接下来调用**arch_prepare_kretprobe**函数，该函数架构相关，用于保存并替换regs中的返回地址；
5. 接下来将本次使用的**kretprobe_instance**链接到全局kretprobe_inst_table哈希表中.
   1. > 该哈希表在init_kprobes中初始化。最后如果kretprobe的free_instances链表为空，则说明被探测函数的并行触发流程超过了指定的maxactive上限，仅增加nmissed值不进行探测跟踪。

### 2.3.3 arch_prepare_kretprobe在不同架构下的实现

arch_prepare_kretprobe**在arm架构的实现如下：**

```C
void __kprobes arch_prepare_kretprobe(struct kretprobe_instance *ri,
                                      struct pt_regs *regs)
{
        ri->ret_addr = (kprobe_opcode_t *)regs->ARM_lr;
 
        /* Replace the return addr with trampoline addr. */
        regs->ARM_lr = (unsigned long)&kretprobe_trampoline;
}
```

这里将regs->ARM_lr保存到了ri->ret_addr中，然后原始值被替换成了kretprobe_trampoline函数的地址。

> 注意regs->ARM_lr值的含义是原始代码流程调用被探测函数后的下一条指令的地址（由于regs中指向的是执行被探测函数入口指令时所保存的寄存器值，因此lr寄存器中的内容为执行被探测函数的返回地址），经过这一替换，原始执行流程在执行完整个被探测函数后将跳转到kretprobe_trampoline函数执行，整个函数稍后分析。

在来看**x86_64架构的函数实现：**

```C
void arch_prepare_kretprobe(struct kretprobe_instance *ri, struct pt_regs *regs)
{
        unsigned long *sara = stack_addr(regs);
 
        ri->ret_addr = (kprobe_opcode_t *) *sara;
 
        /* Replace the return addr with trampoline addr */
        *sara = (unsigned long) &kretprobe_trampoline;
}
```

整体大同小异，x86_64架构的函数调用栈同arm的不同，它将返回地址保存在栈顶空间中（即sp指向的位置），因此保存和替换的方式同arm架构略有不同。

### 2.3.4 kretprobe_trampoline在不同架构下的实现

pre_handler_kretprobe函数返回后，kprobe流程接着执行singlestep流程并返回到正常的执行流程，被探测函数（do_fork）继续执行，直到它执行完毕并返回。

由于返回地址被替换为kretprobe_trampoline，所以跳转到kretprobe_trampoline执行，该函数架构相关且有嵌入汇编实现，具体分析一下。

#### arm架构实现

```C++
/*
When a retprobed function returns, trampoline_handler() is called,
calling the kretprobe's handler. We construct a struct pt_regs to
give a view of registers r0-r11 to the user return-handler.  This is
not a complete pt_regs structure, but that should be plenty sufficient
for kretprobe handlers which should normally be interested in r0 only
anyway.
 */
void __naked __kprobes kretprobe_trampoline(void)
{
        asm volatile (
                "stmdb        sp!, {r0 - r11}                \n\t"
                "mov        r0, sp                        \n\t"
                "bl        trampoline_handler        \n\t"
                "mov        lr, r0                        \n\t"
                "ldmia        sp!, {r0 - r11}                \n\t"
ifdef CONFIG_THUMB2_KERNEL
                "bx        lr                        \n\t"
else
                "mov        pc, lr                        \n\t"
endif
                : : : "memory");
}
```

该函数在栈空间构造出一个不完整的pt_regs结构体变量，仅仅填充了r0~r11寄存器（由于kretprobe所关注的仅是函数返回值r0，这已经足够了），然后跳转到trampoline_handler函数执行：

```C++
/* Called from kretprobe_trampoline */
static __used __kprobes void *trampoline_handler(struct pt_regs *regs)
{
        struct kretprobe_instance *ri = NULL;
        struct hlist_head *head, empty_rp;
        struct hlist_node *tmp;
        unsigned long flags, orig_ret_address = 0;
        unsigned long trampoline_address = (unsigned long)&kretprobe_trampoline;
 
        INIT_HLIST_HEAD(&empty_rp);
        kretprobe_hash_lock(current, &head, &flags);
 
        /*
         * It is possible to have multiple instances associated with a given
         * task either because multiple functions in the call path have
         * a return probe installed on them, and/or more than one return
         * probe was registered for a target function.
         *
         * We can handle this because:
         *     - instances are always inserted at the head of the list
         *     - when multiple return probes are registered for the same
         *       function, the first instance's ret_addr will point to the
         *       real return address, and all the rest will point to
         *       kretprobe_trampoline
         */
        hlist_for_each_entry_safe(ri, tmp, head, hlist) {
                if (ri->task != current)
                        /* another task is sharing our hash bucket */
                        continue;
 
                if (ri->rp && ri->rp->handler) {
                        __this_cpu_write(current_kprobe, &ri->rp->kp);
                        get_kprobe_ctlblk()->kprobe_status = KPROBE_HIT_ACTIVE;
                        ri->rp->handler(ri, regs);
                        __this_cpu_write(current_kprobe, NULL);
                }
 
                orig_ret_address = (unsigned long)ri->ret_addr;
                recycle_rp_inst(ri, &empty_rp);
 
                if (orig_ret_address != trampoline_address)
                        /*
                         * This is the real return address. Any other
                         * instances associated with this task are for
                         * other calls deeper on the call stack
                         */
                        break;
        }
 
        kretprobe_assert(ri, orig_ret_address, trampoline_address);
        kretprobe_hash_unlock(current, &flags);
 
        hlist_for_each_entry_safe(ri, tmp, &empty_rp, hlist) {
                hlist_del(&ri->hlist);
                kfree(ri);
        }
 
        return (void *)orig_ret_address;
}
```

由于前面的kprobe执行流程已经完全退出了，因此这里无法通过传参的手段来获取所触发的到底是哪一个kretprobe_instance，所以只能通过前面的全局kretprobe_inst_table哈希表和current进程描述符指针来确定kretprobe_instance实例：

1. 所以函数首先遍历kretprobe_inst_table哈希表，找到和当前进程绑定的kretprobe_instance。找到了以后会临时修改current_kprobe的值和kprobe的状态值，表明又进入了kprobe的处理流程，防止冲突。
2. 接着调用handler回调函数，传入的第二个入参就是前面kretprobe_trampoline函数构造出来的pt_regs，注意其中的r0寄存器保存的是函数的返回值。
3. handler回调函数执行完毕以后，调用recycle_rp_inst函数将当前的kretprobe_instance实例从kretprobe_inst_table哈希表释放，重新链入free_instances中，以备后面kretprobe触发时使用，另外如果kretprobe已经被注销则将它添加到销毁表中待销毁：

```C++
/*
 * When a retprobed function returns, trampoline_handler() is called,
 * calling the kretprobe's handler. We construct a struct pt_regs to
 * give a view of registers r0-r11 to the user return-handler.  This is
 * not a complete pt_regs structure, but that should be plenty sufficient
 * for kretprobe handlers which should normally be interested in r0 only
 * anyway.
 */
void __naked __kprobes kretprobe_trampoline(void)
{
        asm volatile (
                "stmdb        sp!, {r0 - r11}                \n\t"
                "mov        r0, sp                        \n\t"
                "bl        trampoline_handler        \n\t"
                "mov        lr, r0                        \n\t"
                "ldmia        sp!, {r0 - r11}                \n\t"
#ifdef CONFIG_THUMB2_KERNEL
                "bx        lr                        \n\t"
#else
                "mov        pc, lr                        \n\t"
#endif
                : : : "memory");
}
```

回到trampoline_handler函数中，接下来有一种情况需要注意，由于此处在查找kretprobe_instance时采用的时遍历全局哈希表的方法，同时可能会存在多个kretprobe实例同当前进程绑定的情况，因为在一个被探测函数的调用流程中是可能会调用到其他的被探测函数的，例如下面这种情况：

```C++
int b(void)
{
        int ret;
        
        ...
        
        return ret;
}
 
int a(void)
{
        int ret;
        
        ret = b();
        ...
        
        return ret;
}
```

如果对a函数和b函数同时注册了kretprobe，就会出现多kretprobe_instance绑定同一进程的情况。对于这种多绑定的情况，在处理b函数返回值时可能会错误的找到绑定到a函数的kretprobe_instance实例，导致探测出现错误。那如何避免这种错误？其实在注释中已经给出说明。这里采用了一种非常巧妙的方法，首先每次插入kretprobe_inst_table表时都是从头插入的，在取出的时候也是从头获取，类似一个堆栈，其次在循环的最后给出了一个break条件，那就是如果函数的原始返回地址不等于kretprobe_trampoline函数的地址，那就break，不再循环查找下一个kretprobe_instance实例。我们知道在一般的情况下这break条件必然满足，所以这里找到的必然是流程上最后一次触发kretprobe探测的实例。

回到trampoline_handler函数最后遍历empty_rp销毁需要释放的kretprobe_instance实例。最后返回被探测函数的原始返回地址，执行流程再次回到kretprobe_trampoline函数中：

```C
                "mov        lr, r0                        \n\t"
                "ldmia        sp!, {r0 - r11}                \n\t"
ifdef CONFIG_THUMB2_KERNEL
                "bx        lr                        \n\t"
else
                "mov        pc, lr                
```

接下来从r0寄存器中取出原始的返回地址，然后恢复原始函数调用栈空间，最后跳转到原始返回地址执行，至此函数调用的流程就回归正常流程了，整个kretprobe探测结束。

#### x86_64架构实现

```C
/*
 * When a retprobed function returns, this code saves registers and
 * calls trampoline_handler() runs, which calls the kretprobe's handler.
 */
static void __used kretprobe_trampoline_holder(void)
{
        asm volatile (
                        ".global kretprobe_trampoline\n"
                        "kretprobe_trampoline: \n"
#ifdef CONFIG_X86_64
                        /* We don't bother saving the ss register */
                        "        pushq %rsp\n"
                        "        pushfq\n"
                        SAVE_REGS_STRING
                        "        movq %rsp, %rdi\n"
                        "        call trampoline_handler\n"
                        /* Replace saved sp with true return address. */
                        "        movq %rax, 152(%rsp)\n"
                        RESTORE_REGS_STRING
                        "        popfq\n"
#else
                        "        pushf\n"
                        SAVE_REGS_STRING
                        "        movl %esp, %eax\n"
                        "        call trampoline_handler\n"
                        /* Move flags to cs */
                        "        movl 56(%esp), %edx\n"
                        "        movl %edx, 52(%esp)\n"
                        /* Replace saved flags with true return address. */
                        "        movl %eax, 56(%esp)\n"
                        RESTORE_REGS_STRING
                        "        popf\n"
#endif
                        "        ret\n");
}
```

实现的原理同arm是一致的，这里会调用SAVE_REGS_STRING把寄存器压栈，构造出pt_regs变量，然后调用trampoline_handler函数，这个函数基本同arm的一模一样，就不贴了，最后kretprobe_trampoline_holder恢复栈空间和原始返回地址，跳转到正常的执行流程中继续执行。

# 3.总结

kretprobe探测技术基于kprobe实现，是kprobes探测技术中的最后一种，内核开发人员可以通过它来动态的探测函数执行的返回值，并且也可以做一些定制话的动作，例如检测函数的执行时间等，使用非常方便。本文介绍了kretprobe探测工具的使用方式及其原理，并通过源码分析了arm架构和x86_64架构下它的实现方式。