# 基于QEMU与BusyBox的Linux内核编译与运行

> 参考文献：
>
> https://ty-chen.github.io/linux-kernel-qemu/
>
> https://www.xjx100.cn/news/16164.html?action=onClick

# 1.源码部分

## 1.1下载Kernel源代码

首先，我们需要下载Linux的代码：https://www.kernel.org/

以Kernel5.15为例，然后，把它解压到一个地方，比如在指定位置新建一个叫做Linux的文件夹，然后把代码解压。

这个时候，我们的目录结构应该是这样子：

```C++
| - Linux
| -  -- linux-5.15.140/
| -  -- linux-5.15.140.tar.xz
```

## 1.2 安装所需的软件包

在控制台输入以下命令，安装所需的软件包。

```C++
sudo apt-get install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison qemu qemu-system qemu-kvm
```

如果你的电脑运行的不是Ubuntu/Debian系列的Linux发行版，请使用对应的包管理器安装以上软件。

## 1.3 配置Linux的编译选项

Linux有很多的编译选项，我们选择默认的即可。

首先，我们在linux源代码的文件夹内，右键打开终端。请确保终端显示的“当前工作目录”为``Linux/linux-5.15.140``。

如果你想对linux内核进行裁剪或者交叉编译，请使用”make menuconfig”选项，可以自定义你的编译配置。当然，对于新手来说，默认配置就可以了。

```C++
make menuconfig

Kernel hacking  --->
Compile-time checks and compiler options  --->
[*] Compile the kernel with debug info 
[*] Compile the kernel with frame pointers
```

## 1.4 开始编译Linux

终于，我们可以开始编译Linux内核了，我们只需要在控制台输入以下命令即可。

```Plaintext
make -j $(nproc) bzImage
// example: make -j 8
```

根据电脑的性能不同，编译时间从1分钟到二十分钟甚至更久。

当显示“Kernel: arch/x86/boot/bzImage is ready“的时候，意味着编译完成了。

> 以下是可能发生的错误：
>
> 编译卡住-没有规则可制作的目标：https://blog.csdn.net/qq_36393978/article/details/118157426 编译内核报错BTF: .tmp_vmlinux.btf: pahole (pahole) is not available：https://blog.csdn.net/qq_36393978/article/details/124274364 zstd: not found：https://blog.csdn.net/qq_43509129/article/details/126065591

# 2.环境部分

## 2.1 配置BusyBox

按照百科的定义：BusyBox 是一个集成了三百多个最常用Linux命令和工具的软件。BusyBox 包含了一些简单的工具，例如ls、cat和echo等等，还包含了一些更大、更复杂的工具，例grep、find、mount以及telnet等。

我们可以这样，在Linux文件夹下，输入以下命令，即可下载busybox的代码。

```Plaintext
wget https://busybox.net/downloads/busybox-1.35.0.tar.bz2
```

也可以在该网站中下载：https://busybox.net/

然后解压的指定文件夹，这里推荐安装到上文提到的Linux目录下。

解压之后，目录结构如图所示：

```C++
| - Linux
| -  -- busybox-1.36.1/
| -  -- busybox-1.36.1.tar.bz2
| -  -- linux-5.15.140/
| -  -- linux-5.15.140.tar.xz
```

然后我们在busybox文件夹下打开控制台，输入以下命令，配置它的编译选项：

```Plaintext
make menuconfig
```

然后就会弹出这样一个界面：

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=YWZlOTlhMTRlNzUwYWQ5MmExZDI3MjZlYTc3OGFjNDdfTjI4SUpjckNvdElMWVpjSHZocW54WmlhMWJ2Y1FlWVZfVG9rZW46UmJwM2JKTm1tb0pvRjF4TExPQ2NDa2x5bnJnXzE3MDU0MTM3MzU6MTcwNTQxNzMzNV9WNA)

我们可以通过上下左右键来操作它。选中Settings，然后回车进入。

一直按方向键，往下找到Build Options

然后选中“Build static binary“，按下键盘的y键，即可选中它。选中之后前面的方框的*号会亮起。如图所示：

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=OWY2MDU1NTdiM2FmNjJlNjIzOGU2NGU4NGZiYjk5MGNfdzZWRXpjek10WFR5NFZwTVdDOGVvb05oTjBmYTBiN1RfVG9rZW46VlBxMWJLTTFWb2pDRXh4MEJscWNoUWR2blFrXzE3MDU0MTM3MzU6MTcwNTQxNzMzNV9WNA)

接着，我们按键盘的右键，将底部光标移动到Exit处，按回车，回到主界面。

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=ZWM4OWZmOTcwYWI4OWVmYTg4NDc0MzljYjc1Y2ZkN2JfR2V1Q3Z0clJVOWhxdlQ0cHdJMkNvY0F0TnEzY0RPcGRfVG9rZW46QnJPbGIwcXJEb3hINUZ4Sm9HRmNTc2N3bmRoXzE3MDU0MTM3MzU6MTcwNTQxNzMzNV9WNA)

接着在主界面同样也是exit.

就会弹出界面询问我们是否保存，我们选择Yes即可。

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=Y2NkNWQ3OGYzNjRlZDhjY2E0ODI3YjgwMDM4MmI5MTFfT3RWS09PT1pLRGRPbGxKeXJKVzNSTHM1RHVzMUFzZUVfVG9rZW46UzVHTmJVU3Ztb0ZvS094cXAyeGN4M0h5bldoXzE3MDU0MTM3MzU6MTcwNTQxNzMzNV9WNA)

## 2.2 创建磁盘镜像

接着，我们回到“Linux”文件夹，在控制台输入以下命令，创建磁盘镜像：

```C++
dd if=/dev/zero of=rootfs.img bs=1M count=20
mkfs.ext4 rootfs.img
```

然后，我们接着运行以下命令，编译BusyBox它装进磁盘镜像。

```C++
mkdir fs
sudo mount -t ext4 -o loop rootfs.img ./fs
cd busybox-1.36.1/
sudo make install -j $(nproc) CONFIG_PREFIX=../fs
```

为文件系统创建一些必须的文件夹

```Shell
cd ../fs
sudo mkdir proc dev etc home mnt
sudo cp -r ../busybox-1.36.1/examples/bootfloppy/etc/* etc/
cd ..
```

更改权限，以免无法运行

```C++
sudo chmod -R 777 fs/ 
```

卸载磁盘镜像

```C++
sudo umount fs
```