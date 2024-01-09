# 非持久化存储的文件系统-Proc节点

## 1.Proc节点的创建与注册

在 proc 文件系统投入使用之前，必须向其中添加数据项。内核提供了几个辅助例程来添加文件、创建目录，等等，使得内核的其余部分能够尽可能容易地完成相关的任务。下文将讨论这些例程。

> 虽然很容易创建新的 proc 数据项，但事实上，用代码来创建新的数据项并不是常例。尽管如此，在进行测试时，这些接口很有用处。借助这些简单、轻量级的接口，我们就可以用很小的代价在内核与用户空间之间打开一条通信渠道用于测试。

### 1.1 数据项的创建

新数据项分两个步骤添加到 proc 文件系统。

首先，创建 proc_dir_entry 的一个新实例，填充描述该数据项的所有需要的信息。然后，将该实例注册到 proc 的数据结构，使得外部能看到该数据项。

因为这两个步骤从来都不独立执行，所以内核提供辅助函数合并了这两个操作，使得可以快捷地创建新的 proc 数据项。

**最常使用的函数是 create_proc_entry ，需要3个参数：**

```C++
extern struct proc_dir_entry *create_proc_entry(const char *name, mode_t mode,struct proc_dir_entry *parent);
```

- name 指定了文件名。
- mode 按传统的UNIX方案（用户/组/其他）指定了访问权限。
- parent 是一个指针，指向该文件父目录的 proc_dir_entry 实例。

**切记：该函数只填充了 proc_dir_entry 结构的一些必要的成员。因此必须对产生的结构作一些手工校正。**

### 1.2 示例代码

下列示例代码可以说明这一点，代码产生的数据项是 proc/net/hyperCard ，提供了一个很棒的网卡信息：

```C++
struct proc_dir_entry *entry = NULL;

entry = create_proc_entry("hyperCard", S_IFREG|S_IRUGO|S_IWUSR,&proc_net);

if (!entry) {
    printk(KERN_ERR "unable to create /proc/net/hyperCard\n");
    return -EIO;
} else {
    entry->read_proc = hypercard_proc_read;
    entry->write_proc = hypercard_proc_write;
}
```

### 1.3 数据项的注册

在创建了数据项之后，使用 fs/proc/generic.c 中的 proc_register 将其注册到 proc 文件系统。该任务划分为3个步骤：

1. 生成一个唯一的 proc 内部编号，向数据项赋予身份。 get_inode_number 返回一个未使用的编号，用于为动态生成的数据项。
2. 必须适当地设置 proc_dir_entry 实例的 next 和 parent 成员，将新数据项集成到 proc 文件系统的层次结构中。
3. 如果此前 proc_dir_entry 的成员 proc_iops 或 proc_fops 为 NULL 指针，那么需要根据文件类型，适当地设置指向 file_operations 和 inode_operations 结构实例的指针。否则，使用原值即可。

**那么对 proc 文件使用什么样的 file_operations 和 inode_operations？** 

相应的指针设置如下：

对普通文件，内核使用 proc_file_operations 和 proc_file_inode_operations 来定义文件和inode操作方法：

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=ZGRjNjdkNTkyYjNmNDYyZWEwNTE4Zjc3ZGQ0MDgwOGJfUUlidFRJeHlUTWE2SnBIdnprR0xvbEFxb0JaVGlEWWNfVG9rZW46RUN6QWJNYXhpb240dFh4dlRZV2N5Z0ZHbkljXzE3MDQ4MTUwOTM6MTcwNDgxODY5M19WNA)

### 1.4 数据项创建的辅助函数

除了 create_proc_entry ，内核还提供了两个辅助函数创建新的 proc 数据项。这3个简短的函数实际上都是 create_proc_entry 的包装器例程，后两个定义如下：

```C++
<proc_fs.h>
static inline struct proc_dir_entry *create_proc_read_entry(const char *name,mode_t mode, struct proc_dir_entry *base,read_proc_t *read_proc, void * data) { ... }
static inline struct proc_dir_entry *create_proc_info_entry(const char *name,mode_t mode, struct proc_dir_entry *base, get_info_t *get_info) { ... }
```

create_proc_read_entry 和 create_proc_info_entry 用于创建一个新的可读取的数据项。

因为该任务可以用两种不同的方式完成（10.1.2节讨论过这个问题），必须有两个对应的例程。

尽管create_proc_info_entry 需要一个类型为 get_info_t 的函数指针作为参数，但 create_proc_read_entry 不仅需要一个类型为 read_proc_t 的函数指针参数，还需要一个数据指针参数，以便将同一读取例程用于不同的 proc 数据项，只是根据 data 参数来进行区分。

### 1.5 对于创建节点的补充

> 参考文档：https://www.cnblogs.com/liulaolaiu/p/11744588.html

在实际工作中我们可能也会遇到使用``proc_create``方法用于创建一个新的节点。

其实create_proc_entry比proc_create多了一个赋值默认文件操作的动作，而proc_create是创建时要提供自己的文件操作函数的。

**对于create_proc_entry函数：**

```C++

```

## 2.Proc的读取和写入

10.1.5节提到，内核使用保存在 proc_file_operations 中的操作来读写常规 proc 数据项的内容。该结构中的函数指针，所指向的目标函数如下：

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=ZGExYWQ5NzJkZGY3ZTU5ZWU2ODI1NGE5NzUwNTFkOGFfV3ZVZ2cxUmk1ZlVMcTJWd3pXSzlnc0k2NXFmbDB2N0xfVG9rZW46UFNkeWIwc2t3bzdSTmd4WFdUd2NZeE9ybjVjXzE3MDQ4MTUwOTM6MTcwNDgxODY5M19WNA)

本节将讨论通过 proc_file_read 和 proc_file_write 实现的读写操作。

### 2.1 proc_file_read 的实现

1. 从 proc 文件读取数据的操作分为3个步骤：

1. 分配一个内核内存页面，产生的数据将填充到页面中；
2. 调用一个特定于文件的函数，向内核内存页面填充数据；
3. 数据从内核空间复制到用户空间。

1. 显然，第2个步骤是最重要的，因为必须为此特意准备好子系统的数据和内核中的数据结构。其他两个步骤都是简单的例行任务。

10.1.2节提到，内核在 proc_dir_entry 结构中提供了两个函数指针get_info 和 read_proc 。这两个函数用于读取数据，而内核必须选择一个匹配的来使用。

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=YmVkMmQ0ODRiZTU2MjY3Njc0ZjUwMWQ2YzVjYThiMGVfM2EwMTF1YTBEU29rWXI4R2xEMnlweFB2TmplcHBNRTRfVG9rZW46SFB5UGJsbDBjb0h3dWZ4aXF2S2NjUDVDbkJnXzE3MDQ4MTUwOTM6MTcwNDgxODY5M19WNA)

page 是一个指针，指向第一步中分配的用于保存数据的内存页面。由于10.1.5节已经给出了 read_proc 的一个示例实现，我就不在这里重复了。

### 2.2 proc_file_write 的实现

向 proc 文件写入数据也很简单，至少从该文件系统来看是这样。 proc_file_write 的代码非常紧凑，因而在下面完全复制过来。

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=YjNjNWE1Mzc1OWQ4NmVhYmVlNWNjMDA2NWEwM2U4MGNfTnVkZ3RWWENvdGFlOXJaOURpV0FTSW91NlRmZGdoVVNfVG9rZW46TmNDcWJ1S041b2V4NzV4c3BxZ2NjZDR4bkhoXzE3MDQ4MTUwOTM6MTcwNDgxODY5M19WNA)

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=MjE0ODg0ZTQ5NTY0NzI1NDA3YWU0MzM2MDBkYmM3Y2ZfR0VTNlVkWkJKZXc4SEwySkIyRWVNNU5HRE1BUWFDaWtfVG9rZW46Qkd6MmJFQTIzb2JTWW14TnV5RGNDeks5bjVmXzE3MDQ4MTUwOTM6MTcwNDgxODY5M19WNA)

PDE 函数用于从VFS inode使用container_of机制获得所需的 proc_dir_entry 实例，它是非常简单的。它不过是执行 PROC_I(inode)->pde 而已。在10.1.2节讨论过， PROC_I 用于找到与inode关联的proc_inode 实例（就 proc inode而论，其inode数据总是紧接着VFS inode之前）。

### 2.3 客制化proc_file_write

找到了 proc_dir_entry 实例后，必须用适当的参数来调用注册的写例程，当然，这里假定该例程是存在的，不是 NULL 指针。

**内核如何为 proc 数据项实现写例程？**

答案是使用 proc_write_foobar ，该函数是一个示例，内核源代码用其来示范如何编写例程，来处理对 proc 数据项的写入。

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=ZWFjMTc3MTg2ZWY0ZDEzYTAwNTA1MjhjNmU3YWE2ODhfWWhUcVAxdXRaVHJFdXVFV0JCNXI1aDFrN29NSmI2Y1lfVG9rZW46TW9nZmJ1ZHVub0pISEh4bEttVGMweVlIbjNnXzE3MDQ4MTUwOTM6MTcwNDgxODY5M19WNA)

通常， proc_write 的实现会执行下列操作：

- 首先，必须检查用户输入的长度（使用 count 参数确定），确保不超出所分配区域的长度。
- 数据从用户空间复制到分配的内核空间区域。
- 从字符串中提取出信息。该操作称之为解析（parsing），这是从编译器设计借用的术语。在上述例子中，该任务委托给 cpufreq_parse_policy 函数。
- 接下来，根据收到的用户信息，对该（子）系统进行操作。

## 3.Proc节点的管理

尽管我们对其实现不感兴趣，我在下面仍然列出了一组其他的辅助函数，用于管理 proc 数据项。

- proc_mkdir 创建一个新目录。
- proc_mkdir_mode 创建一个新目录，目录的访问权限可以显式指定。
- proc_symlink 生成一个符号链接。
- remove_proc_entry 从 proc 目录中删除一个动态生成的数据项。

内核源代码包含了一个示例文件，在 Documentation/DocBook/procfs_example.c 中。该文件演示了这里描述的选项，可以用作编写 proc 例程的模板。10.1.6节包含了一些示例内核源代码例程，负责 proc 文件系统的读/写例程和内核子系统之间的交互。

## 4.Proc节点的查找

用户空间应用程序访问 proc 文件时，就像是访问常规文件系统中的普通文件一样。

换句话说，搜索 proc 数据项时所经由的代码路径，与第8章中讲述的VFS例程是相同的。按第8章的讨论，查找过程（例如，在open系统调用中的查找)将在一定的时间到达real_lookup，该函数将调用。

inode_operations 的 lookup 函数指针，根据文件名的各个路径分量，来确定文件名所对应的inode。

本节中，我们讨论一下内核在 proc 文件系统中查找文件时，需要哪些步骤。

对 proc 数据项的搜索从 proc 文件系统的装载点开始，通常是 /proc 。在10.1.2节中，读者已经看到，在 proc 文件系统根目录的 file_operations 实例中，其 lookup 指针指向了 proc_root_lookup 函数。图中给出了相关的代码流程图。

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=NWE0YTk0ZGY2Yzk1ZDBmOWRlNDhhMDZjY2ViMDZjMzBfdEN2VkQzRkdZS0ZxRnZwY09VV2RQdnhxT054QXd6YzRfVG9rZW46VGtUR2JaalNZb3JEWjl4cWlJd2NYb242blFkXzE3MDQ4MTUwOTM6MTcwNDgxODY5M19WNA)

在将实际工作委托给具体的例程之前，内核使用该例程来区分两种不同类型的 proc 数据项。

数据项可能是某个特定于进程的目录中的文件，例如 /proc/1/maps 。另外，数据项也可能是驱动程序或子系统动态注册的文件（例如， /proc/cpuinfo 或 /proc/net/dev ）。区分这两种文件是内核的责任。

- 内核首先调用 proc_lookup 查找常规的数据项。如果函数找到了所查找的文件（顺序扫描指定路径的各个分量），那么一切都好，查找操作就此结束。
- 如果 proc_lookup 没有找到数据项，内核将调用 proc_pid_lookup 查找特定于进程的数据项。

这里不讨论这些函数的细节了。我们只需要知道，函数需要返回一个适当的inode类型（10.1.7节将再次讨论 proc_pid_lookup ，该节的讨论将涉及特定于进程的inode的创建和结构）。