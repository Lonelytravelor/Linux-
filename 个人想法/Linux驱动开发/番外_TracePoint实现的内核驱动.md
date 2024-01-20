> 注:本文使用的笔记与代码均来源于网上与开源代码库

# 1.TracePoint背景

> A tracepoint placed in code provides a hook to call a function (probe) that you can provide at runtime. A tracepoint can be “on” (a probe is connected to it) or “off” (no probe is attached). When a tracepoint is “off” it has no effect, except for adding a tiny time penalty (checking a condition for a branch) and space penalty (adding a few bytes for the function call at the end of the instrumented function and adds a data structure in a separate section). When a tracepoint is “on”, the function you provide is called each time the tracepoint is executed, in the execution context of the caller. When the function provided ends its execution, it returns to the caller (continuing from the tracepoint site).

一个 tracepoint 可以放置在 code 中，提供 hook (钩子) 的功能，该 hook 功能可以 call 一个 function (probe，探针)，这个 function 可以在 runtime 时提供。tracepoint 可以是 “on”（连接了一个 probe探针），或者是 “off”（没有探针连接）。当一个 tracepoint 是 “off” 时，除了会有一点点时间惩罚（检查分支条件）和空间惩罚（在插入指令的函数的末尾为函数调用添加几个字节，并在单独的 section 添加数据结构）外，就没有其它影响了。当 tracepoint 为 “on” 时，每当执行到跟踪点时你提供的 probe function 都会在调用方（caller）的上下文中被调用。所以当 probe function 执行完时，都会回到 caller 继续执行。

> You can put tracepoints at important locations in the code. They are lightweight hooks that can pass an arbitrary number of parameters, which prototypes are described in a tracepoint declaration placed in a header file.

您可以在代码中的重要位置放置 tracepoints，它们是轻量级的挂钩（lightweight hooks），可以传递任意数量的参数，原型被描述在头文件中的 tracepoint 声明中。

> They can be used for tracing and performance accounting.

它们可以用于跟踪和统计。

# 2.TracePoint举例

## 2.1 用法综述

tracepoint 的使用需要下面三个条件：

- \#include <linux/tracepoint.h>
- 定义 tracepoint，应该定义在**头文件**中。
- 在 C 的代码中放置 tracepoint。

## 2.2 代码示例

首先在 include/trace/events/subsys.h 中：

```C++
// 使用 DECLARE_TRACE() 定义了一个 tracepoint

#undef TRACE_SYSTEM
#define TRACE_SYSTEM subsys

#if !defined(_TRACE_SUBSYS_H) || defined(TRACE_HEADER_MULTI_READ)
#define _TRACE_SUBSYS_H

#include <linux/tracepoint.h>

DECLARE_TRACE(subsys_eventname,
        TP_PROTO(int firstarg, struct task_struct *p),
        TP_ARGS(firstarg, p));

#endif /* _TRACE_SUBSYS_H */

/* This part must be outside protection */
#include <trace/define_trace.h>
```

其次在内核模块中进行注册:

```C++
register_trace_subsys_eventname(xxx,NULL)
unregister_trace_subsys_eventname(xxx,NULL)
```

最后在 subsys/file.c 中：

```C++
// 在 somefct() 函数中添加了先前定义的 tracepoint

#include <trace/events/subsys.h>

#define CREATE_TRACE_POINTS
DEFINE_TRACE(subsys_eventname);

void somefct(void)
{
        ...
        trace_subsys_eventname(arg, task);
        ...
}
```

其中关键点：

- subsys_eventname 是你给这个事件（tracepoint）取的标识符名称
- `TP_PROTO(int firstarg, struct task_struct *p)` 是 tracepoint 的 callback 的函数原型
- `TP_ARGS(firstarg, p)` 是参数的名字，可以发现和函数原型中的一致
- 如果头文件被多个源文件使用，#define CREATE_TRACE_POINTS 只应该在一个源文件中出现

## 2.3 其他注意点

跟踪点机制支持插入同一跟踪点的多个实例，但必须在所有内核上对给定的跟踪点名称进行单个定义，以确保不会发生类型冲突。跟踪点的名称管理是使用原型来完成的，以确保键入的内容是正确的。编译器在注册站点验证探针类型的正确性。跟踪点可以放在内联函数、内联静态函数、展开循环以及常规函数中。

这里建议使用命名方案“subsys_event”作为限制冲突的约定。跟踪点名称是内核的全局名称：无论是在内核映像中还是在模块中，它们都被认为是相同的。

如果 tracepoint 不得不在内核模块中使用，可以使用 EXPORT_TRACEPOINT_SYMBOL_GPL() 或 EXPORT_TRACEPOINT_SYMBOL() 来导出定义的 tracepoint。

**注：**方便的宏 TRACE_EVENT() 提供了另一种方法来定义 tracepoint，访问 [Using the TRACE_EVENT() macro (Part 1)](https://lwn.net/Articles/379903/) ，[(Part 2)](https://lwn.net/Articles/381064) 和 [(Part 3)](http://lwn.net/Articles/383362) 来了解更多细节。

注2:上述过程中没看懂也没有关系,实现的比较粗略,可以参考后面的详细实现过程.

## 2.4 实际使用案例

**实际上,就是将一个方法xxx()使用DECLARE_TRACE将其包装成trace_xxx(),并且可以使用register_trace_tp_test将某个函数注册至该方法中.**

### 2.4.1 实现方法的注册与调用

**注:这里要注意 callback 的第一个参数一定是 (void\*) 类型！！不得不说，万恶之源宏定义。**

```C++
// module_entry.c
#include <linux/init.h>
#include <linux/module.h>
#include "module_tracepoint.h"

#define CREATE_TRACE_POINTS
DEFINE_TRACE(tp_test);

static void tp_callback_1(void * data, int num)
{
    printk("%s: n = %d\n", __func__, num);
}
static void tp_callback_2(void * data, int num)
{
    printk("%s: n = %d\n", __func__, num * 2);
}

static __init int __module_init(void)
{
    int i = 0;
    printk("Hello, %s.\n", __func__);

    trace_tp_test(++i);
    register_trace_tp_test(tp_callback_1, NULL);
    trace_tp_test(++i);
    printk("event: register tp_callback_2\n");
    register_trace_tp_test(tp_callback_2, NULL);
    trace_tp_test(++i);
    
    return 0;
}
static __exit void __module_exit(void)
{
    unregister_trace_tp_test(tp_callback_1, NULL);
    unregister_trace_tp_test(tp_callback_2, NULL);
    printk("Hello, %s.\n", __func__);
    return;
}

module_init(__module_init);
module_exit(__module_exit);
MODULE_LICENSE("Dual BSD/GPL");
```

**这个方法中主要实现了两件事情:**

- 一个是将tp_callback_1和tp_callback_2注册到trace_tp_test函数当中,这样每次执行的时候都会默认执行被注册的函数
- 二是正好在内核模块中执行该函数.

并且需要注意的是,对同一个方法可以注册多个函数,这样在调用时会依次调用注册的函数.

### 2.4.2 实现tp_test方法的声明

在对应头文件中声明该函数,相当于将此函数通过DECLARE_TRACE的方式进行声明.

```C++
// module_tracepoint.h
#ifndef _MODULE_TRACEPOINT_
#define _MODULE_TRACEPOINT_

#if !defined(_TRACE_SUBSYS_H) || defined(TRACE_HEADER_MULTI_READ)
#define _TRACE_SUBSYS_H

#include <linux/tracepoint.h>

DECLARE_TRACE(tp_test,
    TP_PROTO(int num),
    TP_ARGS(num));

#endif /* _TRACE_SUBSYS_H */
    
/* This part must be outside protection */
#include <trace/define_trace.h>
#endif // _MODULE_TRACEPOINT_
```

这里其实可以理解成把tp_test包装成了trace_tp_test函数

### 2.4.3 Makefile

```C++
CONFIG_MODULE_SIG=n
KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)
MODULE := xhr

obj-m := $(MODULE).o

all:
        $(MAKE) -C $(KDIR) M=$(PWD) modules

.PHONY:clean
clean:
        rm -f *.o *.ko *.symvers *.order *.mod.c *.mod .*.cmd

$(MODULE)-objs += module_entry.o
```

### 2.4.4 试验结果

```C++
root@ubuntu:/home/xhr/study/code/module& insmod xhr.ko
[ 6543.758265] Hello, __module_init.
[ 6543.758631] tp_callback_1: n = 2
[ 6543.758633] event: register tp_callback_2
[ 6543.758634] tp_callback_1: n = 3
[ 6543.758635] tp_callback_2: n = 6
root@ubuntu:/home/xhr/study/code/module& rmmod xhr
[ 6557.501027] Hello, __module_exit.
```

**实际上,就是将一个方法xxx()使用DECLARE_TRACE将其包装成trace_xxx(),并且可以使用register_trace_tp_test将某个函数注册至该方法中.**

# 3.技术方案实现

处理了解Feature本身的实现逻辑，关于其如何起作用、如何注册、如何生效也需要探究一下。

> 参考文档：https://blog.csdn.net/feelabclihu/article/details/113409593

## 3.1 GKI背景

Google在android11-5.4分支上开始要求所有下游厂商使用Generic Kernel Image（GKI），需要将SoC和device相关的代码从核心内核剥离到可加载模块中（下文称之为GKI改造），从而解决内核碎片化问题。GKI为内核模块提供了稳定的内核模块接口（KMI），模块和内核可以独立更新。本文主要介绍了在GKI改造过程中需遵循的原则、遇到的问题和解决方法。

## 3.2 本方案技术逻辑

考虑到SoC和OEM厂商可能要对原生内核做一些客制化修改和优化，Google提供了一套vendor hook机制，下游厂商在需要修改内核源码的地方添加hook，并向Google申请，将patch upstream到AOSP。

## 3.3 vendor hook实现

![image-20240120231009824](/Users/wang/Downloads/image-20240120231009824.png)

### 3.3.1 创建头文件xxx.h

在include/trace/hooks/目录下创建一个新的头文件xxx.h，定义一组hook接口register_trace_android_vh_xxx、trace_android_vh_xxx和全局变量__tracepoint_android_vh_xxx

**注：这里其实就是相当于声明一下具体的函数头**

> 代码引用：https://github.com/MiCode/Xiaomi_Kernel_OpenSource/blob/bsp-vermeer-t-oss/include/trace/hooks/binder.h

```C++
#include <trace/hooks/vendor_hooks.h>
......
DECLARE_HOOK(android_vh_binder_set_priority,
        TP_PROTO(struct binder_transaction *t, struct task_struct *task),
        TP_ARGS(t, task));
```

### 3.3.2 export头文件xxx.h

在drivers/android/vendor_hooks.c文件中包含hook头文件xxx.h，export hook变量__tracepoint_android_vh_xxx给模块使用。

**注：这里其实是将函数export，使得内核模块可以调用到这个函数**

> 代码引用：https://github.com/MiCode/Xiaomi_Kernel_OpenSource/blob/bsp-vermeer-t-oss/drivers/android/vendor_hooks.c

```C++
#define CREATE_TRACE_POINTS
#include <trace/hooks/vendor_hooks.h>
#include <trace/hooks/binder.h>
......
EXPORT_TRACEPOINT_SYMBOL_GPL(android_vh_binder_set_priority);
EXPORT_TRACEPOINT_SYMBOL_GPL(android_vh_binder_restore_priority);
```

### 3.3.3 实现register代码

在内核模块中增加register代码将回调函数绑定到hook变量__tracepoint_android_vh_xxx

**注：这里则是在我们核心代码中进行注册的代码，用于将此核心函数注册到该方法内**

> 代码引用：NULL

```C++
#include<trace/hocks/binder.h>

...
static void xxx(){
    print("hello hock!");
}

rc = register_trace_android_vh_binder_set_priority(xxx,NULL);
```

### 3.3.4 调用hook接口

在内核xxx.c文件中包含hook头文件xxx.h，调用hook接口trace_android_vh_xxx(即hook变量__tracepoint_android_vh_xxx绑定的callback函数)

**注：这里其实是binder.c中调用trace_android_vh_binder_set_priority**

> 代码引用：https://github.com/MiCode/Xiaomi_Kernel_OpenSource/blob/bsp-vermeer-t-oss/drivers/android/binder.c

```C++
#include <linux/android_vendor.h>
#include <trace/hooks/binder.h>
......
static void binder_transaction_priority(struct binder_thread *thread,
                                        struct binder_transaction *t,
                                        struct binder_node *node)
{
        struct task_struct *task = thread->task;
        struct binder_priority desired = t->priority;
        ......
        binder_set_priority(thread, &desired);
        trace_android_vh_binder_set_priority(t, task);
}
```

## 3.4 一些技术细节

### 3.4.1 DECLARE_TRACE与DEFINE_TRACE[ChatGPT]

在Linux内核中，`DEFINE_TRACE`和`DECLARE_TRACE`是用于定义和声明事件跟踪点的宏。

1. **DECLARE_TRACE:**
   1. `DECLARE_TRACE`用于在头文件中声明一个跟踪点，但不分配其具体的实现。这通常在头文件中用于声明一个跟踪点，以便在多个文件中引用相同的跟踪点。
   2. 示例：
   3. ```C
      DECLARE_TRACE(my_event_name);
      ```
2. **DEFINE_TRACE:**
   1. `DEFINE_TRACE`用于在源文件中为跟踪点定义实现。它负责为事件跟踪点生成实际的代码，包括事件注册、注销和调用相关的处理函数。
   2. 示例：
   3. ```C
      DEFINE_TRACE(my_event_name);
      ```

通常，这两者一起使用。首先，在头文件中使用`DECLARE_TRACE`声明跟踪点，然后在一个或多个源文件中使用`DEFINE_TRACE`定义实际的跟踪点实现。

例如，如果你在一个头文件中声明了一个跟踪点：

```C
// my_trace.h
DECLARE_TRACE(my_event_name);
```

然后在源文件中定义该跟踪点：

```C
// my_trace.c   
#include "my_trace.h"
DEFINE_TRACE(my_event_name);
```

这样，你就可以在整个代码库中使用`my_event_name`这个事件跟踪点，而实现部分则在`my_trace.c`中定义。

### 3.4.2 为什么在vendor hook实现没有用到DEFINE_TRACE[ChatGPT]

在你提供的代码中，通过 `DECLARE_TRACE(tp_test, TP_PROTO(int num), TP_ARGS(num));` 语句已经声明了 `tp_test` 这个跟踪点，而在代码中并没有直接使用 `DEFINE_TRACE(tp_test);`。在这个特定的情境下，由于 `DEFINE_TRACE` 的定义通常隐含在 `DECLARE_TRACE` 的内部，你的代码是没有问题的。

在内核中，`DECLARE_TRACE` 通常会包含足够的信息以便生成跟踪点的实现。这可能包括注册、注销和触发函数的相关信息。实际上，你的代码中的 `DEFINE_TRACE(tp_test);` 部分可能已经由内核的其他部分自动生成，而不需要显式写出来。

总的来说，你的代码是合法的，因为 `DECLARE_TRACE` 在这里提供了足够的信息，以便内核在需要时自动生成相应的 `DEFINE_TRACE` 部分。

### 3.4.3 DECLARE_HOOK和DECLARE_TRACE的关系

因为在include/trace/hooks/vendor_hooks.h中将DECLARE_HOOK宏定义为DECLARE_TRACE.

> 代码链接:https://github.com/MiCode/Xiaomi_Kernel_OpenSource/blob/bsp-vermeer-t-oss/include/trace/hooks/vendor_hooks.h

```C++
// include/trace/hooks/vendor_hooks.h
...
#define DECLARE_HOOK DECLARE_TRACE
...
```

## 3.5 偶现问题与原因

**问题现象（xiaomi测试时未发现）**：测试偶现dump “BUG: scheduling while atomic:”

**原因分析：**

vendor hook变量有两种，都是基于tracepoints的：

正常的：使用DECLARE_HOOK宏创建tracepoint函数trace_<name>，要求name在trace中是独一无二的，callback函数的调用是在关抢占的场景中使用的

受限制的：受限制的hook在scheduler hook类的场景中使用，绑定的callback函数可以在cpu offline或非原子上下文中调用（调用前没有关抢占），受限制的vendor hook不能被解绑定，所以绑定的模块不能卸载，只允许有一个绑定（任何其他绑定将会返回-EBUSY错误）。

**解决方法：**根据使用场景选择适合的vendor hook变量，在可能会调度的场景需要使用受限制的vendor hook

> 参考文档:https://blog.csdn.net/u012849539/article/details/106751596

# 4.其他参考文章

linux系统下的各种hook方式：https://dandelioncloud.cn/article/details/1567859018796593153