# 1.技术背景

处理了解Feature本身的实现逻辑，关于其如何起作用、如何注册、如何生效也需要探究一下。

> 参考文档：https://blog.csdn.net/feelabclihu/article/details/113409593

## 1.1 GKI背景

Google在android11-5.4分支上开始要求所有下游厂商使用Generic Kernel Image（GKI），需要将SoC和device相关的代码从核心内核剥离到可加载模块中（下文称之为GKI改造），从而解决内核碎片化问题。GKI为内核模块提供了稳定的内核模块接口（KMI），模块和内核可以独立更新。本文主要介绍了在GKI改造过程中需遵循的原则、遇到的问题和解决方法。

## 1.2 本方案技术逻辑

考虑到SoC和OEM厂商可能要对原生内核做一些客制化修改和优化，Google提供了一套vendor hook机制，下游厂商在需要修改内核源码的地方添加hook，并向Google申请，将patch upstream到AOSP。

## 1.3 vendor hook实现

### 1.3.1 创建头文件xxx.h

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

### 1.3.2 引用头文件xxx.h

在drivers/android/vendor_hooks.c文件中包含hook头文件xxx.h，export hook变量__tracepoint_android_vh_xxx给模块使用。

**注：这里其实是将函数export，使得内核模块可以调用到这个函数**

> 代码引用：https://github.com/MiCode/Xiaomi_Kernel_OpenSource/blob/bsp-vermeer-t-oss/drivers/android/vendor_hooks.c

```C++
#define CREATE_TRACE_POINTS
#include <trace/hooks/vendor_hooks.h>
#include <linux/tracepoint.h>
......
EXPORT_TRACEPOINT_SYMBOL_GPL(android_vh_binder_set_priority);
EXPORT_TRACEPOINT_SYMBOL_GPL(android_vh_binder_restore_priority);
```

### 1.3.3 实现register代码

在内核模块中增加register代码将回调函数绑定到hook变量__tracepoint_android_vh_xxx

**注：这里则是在我们核心代码中进行注册的代码，用于将此核心函数注册到该方法内**

> 代码引用：

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=MDRlZjVhMTE5YzkyYThjYTk3ZDNjZmQ5ZjBmYWM2NmJfbFl6dDZBdE04ZDYzS1ZVTGQzdFlTVVVLd0VSOUVlQWVfVG9rZW46S1lMVmJzMnFDb0xndVl4Mjl0ZmNIYkJWbjFnXzE3MDU2Nzc3NTU6MTcwNTY4MTM1NV9WNA)

### 1.3.4 调用hook接口

在内核xxx.c文件中包含hook头文件xxx.h，调用hook接口trace_android_vh_xxx(即hook变量__tracepoint_android_vh_xxx绑定的callback函数)

**注：这里其实是binder.c中调用trace_android_vh_binder_set_priority**

> 代码引用：https://github.com/MiCode/Xiaomi_Kernel_OpenSource/blob/bsp-vermeer-t-oss/drivers/android/binder.c

```C++
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

## 1.4 偶现问题与原因

**问题现象（xiaomi测试时未发现）**：测试偶现dump “BUG: scheduling while atomic:”

**原因分析：**

vendor hook变量有两种，都是基于tracepoints的：

正常的：使用DECLARE_HOOK宏创建tracepoint函数trace_<name>，要求name在trace中是独一无二的，callback函数的调用是在关抢占的场景中使用的

受限制的：受限制的hook在scheduler hook类的场景中使用，绑定的callback函数可以在cpu offline或非原子上下文中调用（调用前没有关抢占），受限制的vendor hook不能被解绑定，所以绑定的模块不能卸载，只允许有一个绑定（任何其他绑定将会返回-EBUSY错误）。

**解决方法：**根据使用场景选择适合的vendor hook变量，在可能会调度的场景需要使用受限制的vendor hook

# 2.其他参考文章

linux系统下的各种hook方式：https://dandelioncloud.cn/article/details/1567859018796593153