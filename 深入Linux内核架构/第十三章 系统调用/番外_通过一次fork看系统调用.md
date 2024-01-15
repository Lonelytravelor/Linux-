# 番外_通过 fork 看一次系统调用

> 文章参考：https://mp.weixin.qq.com/s/rYBSH_AZDwgc8knSKDSSxA
>
> 注：本文主要参考与Linux0.11，后续在实现和调用上均有修改，可以参考其设计思想，但不推荐作为流程去学习。

# 1.fork 函数

首先看下在Linux0.11中，初始化后进入的main函数中调用了fork，实现了操作系统中首次创建新进程：

```C++
void main(void) {
    ...    
    move_to_user_mode();
    if (!fork()) {
        init();
    }
    for(;;) pause();
}
```

fork 函数的简介：

```C++
static _inline _syscall0(int,fork)

#define _syscall0(type,name) \
type name(void) \
{ \
long __res; \
__asm__ volatile ("int $0x80" \
    : "=a" (__res) \
    : "0" (__NR_##name)); \
if (__res >= 0) \
    return (type) __res; \
errno = -__res; \
return -1; \
}
```

具象化一点：

```C++
#define _syscall0(type,name) \
type name(void) \
{ \
    volatile long __res; \
    _asm { \
        _asm mov eax,__NR_##name \
        _asm int 80h \
        _asm mov __res,eax \
    } \
    if (__res >= 0) \
        return (type) __res; \
    errno = -__res; \
    return -1; \
}
```

具体看一下 fork 函数里面的代码，又是讨厌的内联汇编，不过上面我已经变成好看一点的样子了，而且不用你看懂，听我说就行。 

- 关键指令就是一个 0x80 号软中断的触发，**int 80h**。
- 其中还有一个 eax 寄存器里的参数是 **__NR_fork**，这也是个宏定义，值是 **2**。

# 2.syscall

## 2.1 了解系统调用

上面说的0x80 号中断其实是用于处理系统调用的中断，使用``set_system_gate(0x80, &system_call);``进行注册。

在看一下 system_call 的汇编代码：

```C++
_system_call:
    ...
    call [_sys_call_table + eax*4]
    ...
```

## 2.2 系统调用表

刚刚那个值就用上了，eax 寄存器里的值是 2，所以这个就是在这个 **sys_call_table** 表里找下标 2 位置处的函数，然后跳转过去。

那我们接着看 sys_call_table 是个啥。

```C++
fn_ptr sys_call_table[] = { sys_setup, sys_exit, sys_fork, sys_read,
  sys_write, sys_open, sys_close, sys_waitpid, sys_creat, sys_link,
  sys_unlink, sys_execve, sys_chdir, sys_time, sys_mknod, sys_chmod,
  sys_chown, sys_break, sys_stat, sys_lseek, sys_getpid, sys_mount,
  sys_umount, sys_setuid, sys_getuid, sys_stime, sys_ptrace, sys_alarm,
  sys_fstat, sys_pause, sys_utime, sys_stty, sys_gtty, sys_access,
  sys_nice, sys_ftime, sys_sync, sys_kill, sys_rename, sys_mkdir,
  sys_rmdir, sys_dup, sys_pipe, sys_times, sys_prof, sys_brk, sys_setgid,
  sys_getgid, sys_signal, sys_geteuid, sys_getegid, sys_acct, sys_phys,
  sys_lock, sys_ioctl, sys_fcntl, sys_mpx, sys_setpgid, sys_ulimit,
  sys_uname, sys_umask, sys_chroot, sys_ustat, sys_dup2, sys_getppid,
  sys_getpgrp, sys_setsid, sys_sigaction, sys_sgetmask, sys_ssetmask,
  sys_setreuid, sys_setregid
};
```

各种函数指针组成的一个数组，是一个系统调用函数表。通过下标2找到的第二项就是 **sys_fork** 函数！

至此，我们终于找到了 fork 函数，通过系统调用这个中断，最终走到内核层面的函数是什么，就是 sys_fork。

实际的执行者sys_fork

然后我们可以看下，这里请求到了内核方法中的_sys_fork，用于处理真正的fork函数；

```C++
_sys_fork:
    call _find_empty_process
    testl %eax,%eax
    js 1f
    push %gs
    pushl %esi
    pushl %edi
    pushl %ebp
    pushl %eax
    call _copy_process
    addl $20,%esp
1:  ret
```

# 3.补充

定义 fork 的系统调用模板函数时，用的是 **syscall0**，其实这个表示参数个数为 0，也就是 sys_fork 函数并不需要任何参数。

在 unistd.h 头文件里，还定义了 syscall0 ~ syscall3 一共四个宏。

```C++
#define _syscall0(type,name)
#define _syscall1(type,name,atype,a)
#define _syscall2(type,name,atype,a,btype,b)
#define _syscall3(type,name,atype,a,btype,b,ctype,c)
```

**syscall1** 就表示有**一个参数**，**syscall2** 就表示有**两个参数，以此类推。**

> 后面《深入Linux内核设计》中会详细讲述一下各个函数的含义以及对于syscall0等宏的使用。
>
> 上面有提到所有的处理程序最多接受五个参数，并且参数与寄存器之间精确对应；

# 4.总结

操作系统通过**系统调用**，提供给用户态可用的功能，都暴露在 **sys_call_table** 里了。

系统调用统一通过 **int 0x80** 中断来进入，具体调用这个表里的哪个功能函数，就由 **eax** 寄存器传过来，这里的值是个数组索引的下标，通过这个下标就可以找到在 sys_call_table 这个数组里的具体函数。

同时也可以看出，用户进程调用内核的功能，可以直接通过写一句 int 0x80 汇编指令，并且给 eax 赋值，当然这样就比较麻烦。

所以也可以直接调用 fork 这样的包装好的方法，而这个方法里本质也是 int 0x80 以及 eax 赋值而已。

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDk5YzFkYmFhMzkxYTljOWQxNDQ1MTFhMjg2MDMwYTVfU0V5aXhvOUhqWkthUkFZWXk5aDdtdTRzcW1kRmJPanlfVG9rZW46Um93TWJtdDE1b3cyOGZ4UDVEYmNEdjFDbmZlXzE3MDUzMzI0OTY6MTcwNTMzNjA5Nl9WNA)