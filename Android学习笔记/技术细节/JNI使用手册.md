# 1.前言

> 文章参考:
>
> - 详解JNI(一):https://www.cnblogs.com/longfurcat/p/9830129.html
> - 一篇文章教你完全掌握jni技术:https://juejin.cn/post/7198434169982304316
> - Android JNI使用全面讲解:https://zhuanlan.zhihu.com/p/97691316
> - NDK 系列（5）：JNI 从入门到实践：https://zhuanlan.zhihu.com/p/547250316
> - NDK 系列（6）：说一下注册 JNI 函数的方式和时机：https://juejin.cn/post/7125021894562349092/

## 1.1 什么是JNI

jni全称java native interface，我把它分为三部分：

- java代表java语言；
- native代表当前程序运行的本地环境，一般指windows/linux，而这些操作系统都是通过C/C++实现的，所以native通常也指C/C++语言；
- interface代表java跟native两者之间的通信接口。

jni可以实现java和C/C++通信。它是java生态的特征，所以定义在jdk标准当中。

## 1.2 **使用场景和优势**

**跨平台调用**,java虽然跨平台，但仍然运行在具体平台(windows，linux)之上，对于需要操作硬件的功能，必须通过系统的C/C++方法对硬件进行直接操作，比如打开文件，java层必须调用系统的open方法（linux是open，windows是openFile）才能打开文件，这个时候就涉及到java代码如何调用C/C++代码的问题

**更高的执行效率**,在一些拥有复杂算法的场景（音视频编解码，图像绘制等），java的执行效率远低于C/C++的执行效率，使用jni技术，在java层调用C/C++代码，可以提高程序的执行效率，最大化利用机器的硬件资源。

**native层的代码往往更加安全**，反编译so文件比反编译jar文件要难得多，所以，我们往往把涉及到密码密钥相关的功能用C/C++实现，然后java层通过jni调用

**更丰富的调用库**,已经有大量的库已经被实现，编程者可直接使用，不用再自行编写。这里的库指的是用其他编程语言实现的程序库，例如IO流或者线程等底层与OS交互的操作都是由C/C++实现的。

## 1.3 JNI通信原理

java运行在jvm，jvm本身就是使用C/C++编写的，因此jni只需要在java代码、jvm、C/C++代码之间做切换即可.

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=NmFhOThlNGQ3ZTk5NGIxZjUxY2YyNmJiMGM5NjM0NWZfN0RCcEFmT01ZN01pTEc1VDZJT2liNmFVWHVWNEdIdENfVG9rZW46SUJ1d2JjMXlLb25XeHd4d3RhOGNzVGx6bmhlXzE3MDY1NDIzNzc6MTcwNjU0NTk3N19WNA)

## 1.4 JNI和NDK

**什么是JNI/NDK？二者的区别是什么？**

- JNI，全名 Java Native Interface，是Java本地接口，JNI是Java调用Native语言的一种特性，通过JNI可以使得Java与C/C++机型交互。简单点说就是JNI是Java中调用C/C++的统称。
- NDK 全名Native Develop Kit，官方说法：Android NDK 是一套允许您使用 C 和 C++ 等语言，以原生代码实现部分应用的工具集。在开发某些类型的应用时，这有助于您重复使用以这些语言编写的代码库。

JNI和NDK都是调用C/C++代码库。所以总体来说，除了应用场景不一样，其他没有太大区别。细微的区别就是：JNI可以在Java和Android中同时使用，NDK只能在Android里面使用。

**JNI/NDK用来做什么？**

一句话，快速调用C/C++的动态库。除了调用C/C++之外别无它用。

# 2.JNI用法说明

## 2.1 数据类型转换

开头提到，java和C/C++通信是通过jni来完成的，那么在jni方法中就涉及到对java变量的访问（变量类型包括基本数据类型和引用数据类型），对java方法的调用，java对象的创建等，而java语法跟jni语法不一定是一 一对应的，比如，java中叫`boolean`，jni中叫`jboolean`.

那怎么解决这个问题呢，jni给我们提供了若干个映射表，将java中的类型与jni中的类型进行了一 一映射，其中包括基本数据类型映射，引用数据类型映射，方法签名(包含参数和返回值)映射，以下是这三个映射表：

### 2.1.1 基本数据类型映射表

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=ODA5MWQ1YjFjNmRiZGY0ODllZGJhMmIxNGY1NWIyYWVfcFozZm1naERaQWVsOVJSUDJnNXd5Y0hWN2p6dG5zb1NfVG9rZW46V2RUWmJoNUdWb25MZUp4SFNXdGNHb2R1bkllXzE3MDY1NDIzNzc6MTcwNjU0NTk3N19WNA)

### 2.1.2 引用数据类型映射表

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=ODVhZDI3YTMxMmNiOWI1NThmNGJiODE0OTUyNmY5YjVfV0NTcngxSWNQTW5laTExZ1J5SE5CMUNmcngwYVJ3NU5fVG9rZW46S3pyd2IwNWlSb3J1UnN4emxTd2NlSlFkbnFiXzE3MDY1NDIzNzc6MTcwNjU0NTk3N19WNA)

### 2.1.3 方法签名

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=NDBkOTU2M2Y3ZGU2ODM0ZGE1MTcwZGYxN2NiMzQzMmZfczJzdE5ZbWJSbldUNlh5dFBLaVR6TXBhZEhQWXVEQlNfVG9rZW46R29JNGI4emxJb0RVbDR4b3l1NGN1WlFhblNlXzE3MDY1NDIzNzc6MTcwNjU0NTk3N19WNA)

### 2.1.4 示例demo

```Java
//Java方法
public static native String helloJni();
public static native float helloJni2(int age, boolean isChild);

//jni方法
extern "C"JNIEXPORT jstring JNICALL Java_com_jason_jni_JNIDemo_helloJni
    (JNIEnv *env, jclass clazz){    
    return env->NewStringUTF("I am from c++");
}
extern "C"JNIEXPORT jfloat JNICALL Java_com_jason_jni_JNIDemo_helloJni2
    (JNIEnv *env, jclass clazz, jint age, jboolean isChild){ 
}
```

java方法`helloJni()`的返回值为`String`，映射到jni方法中的返回值即为`jstring`，我们新增一个方法`helloJni2(int age, boolean isChild)`，增加了两个参数`int`和`boolean`，对应的映射为`jint`和`jboolean`，同时返回值`float`映射为`jfloat`。

## 2.2 一些访问java成员的API

### 2.2.1 jni访问调用对象

| 方法名         | 作用                                 |
| -------------- | ------------------------------------ |
| GetObjectClass | 获取调用对象的类，我们称其为target   |
| FindClass      | 根据类名获取某个类，我们称其为target |
| IsInstanceOf   | 判断一个类是否为某个类型             |
| IsSameObject   | 是否指向同一个对象                   |

### 2.2.2 jni访问java成员/静态成员变量的值

| 方法名            | 作用                                                         |
| ----------------- | ------------------------------------------------------------ |
| GetFieldId        | 根据变量名获取target中成员变量的ID                           |
| GetIntField       | 根据变量ID获取int变量的值，对应的还有byte，boolean，long等   |
| SetIntField       | 修改int变量的值，对应的还有byte，boolean，long等             |
| GetStaticFieldId  | 根据变量名获取target中静态变量的ID                           |
| GetStaticIntField | 根据变量ID获取int静态变量的值，对应的还有byte，boolean，long等 |
| SetStaticIntField | 修改int静态变量的值，对应的还有byte，boolean，long等         |

### 2.2.4 jni访问java成员/静态方法

| 方法名               | 作用                                                   |
| -------------------- | ------------------------------------------------------ |
| GetMethodID          | 根据方法名获取target中成员方法的ID                     |
| CallVoidMethod       | 执行无返回值成员方法                                   |
| CallIntMethod        | 执行int返回值成员方法，对应的还有byte，boolean，long等 |
| GetStaticMethodID    | 根据方法名获取target中静态方法的ID                     |
| CallStaticVoidMethod | 执行无返回值静态方法                                   |
| CallStaticIntMethod  | 执行int返回值静态方法，对应的还有byte，boolean，long等 |

### 2.2.6 jni访问java构造方法

| 方法名      | 作用                                                     |
| ----------- | -------------------------------------------------------- |
| GetMethodID | 根据方法名获取target中构造方法的ID，注意，方法名传<init> |
| NewObject   | 创建对象                                                 |

### 2.2.7 jni创建引用

| 方法名           | 作用                                       |
| ---------------- | ------------------------------------------ |
| NewGlobalRef     | 创建全局引用                               |
| NewWeakGlobalRef | 创建弱全局引用                             |
| NewLocalRef      | 创建局部引用                               |
| DeleteGlobalRef  | 释放全局对象，引用不主动释放会导致内存泄漏 |
| DeleteLocalRef   | 释放局部对象，引用不主动释放会导致内存泄漏 |

### 2.2.8 jni异常处理机制

jni提供了异常处理机制，处理方式跟java一样有两种，要么往上(java层)抛，要么自己捕获处理

| 方法名            | 作用                       |
| ----------------- | -------------------------- |
| ExceptionOccurred | 判断是否有异常发生         |
| ExceptionClear    | 清除异常                   |
| Throw             | 往上(java层)抛出异常       |
| ThrowNew          | 往上(java层)抛出自定义异常 |

## 2.3 静态注册和动态注册

Java 的 native 方法和 JNI 函数是一一对应的映射关系，建立这种映射关系的注册方式有 2 种：

- 方式 1 - 静态注册： 基于命名约定建立映射关系；
- 方式 2 - 动态注册： 通过 `JNINativeMethod` 结构体建立映射关系。

jni静态注册的方式，按照jni规范的命名规则进行查找，格式为`Java_类路径_方法名`，这种方式在应用层开发用的比较广泛，因为Android Studio默认就是用这种方式，而在framework当中几乎都是采用动态注册的方式来实现java和c/c++的通信。

# 3.实例演示

总的来说,使用JNI的主要方法是:

1. 编写Java文件,并使用native关键字声明方法；
2. 编译Java文件,获取对应的class文件与.h文件；
3. 编写C代码,引用对应的.h文件,并实现对应方法；
4. 使用gcc生成动态链接库
5. 运行Java程序

具体流程如下：

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=ZTdhMTgwNGJhMmY5NTQ5ZGVmNTIwNjYwNmQ5NDE0MmJfaHhKV1FIemxJSzVUczN3cVRQYWYybnhhNmpkTk9aVlhfVG9rZW46TVNLTWJZd0Jvb2tnUjJ4VklWQmNzM284bjVnXzE3MDY1NDIzNzc6MTcwNjU0NTk3N19WNA)

## 3.1 Hello World的实现

### 3.1.1 创建Java文件

在本地文件夹中创建一个Java文件,在其中实现如下代码:

```Java
public class HelloJNI{
    static{
        System.load("home/mi/JNI/hello.so");
    }
    
    private native void sayHello();
    
    public static void main(String[] args){
        new HelloJNI().sayHello();
    }
}
```

这里，我们定义了一个native方法，是个空方法体，我们在主函数内对其进行调用。

> 注一:hello.so是后面编译出来的动态链接库,目前还未生成该文件
>
> 注二：这里使用的是System.load从绝对路径引用动态链接库，当然也可以使用loadLibrary方法，其是从java.library.path对应路径下搜索对应名称的库文件并加载。

### 3.1.2 编译Java文件

编译HelloJNI.java文件，生成类文件,并使用JDK提供的JNI命令工具，javah生成 .h头文件。

```C
// 编译HelloJNI.java文件，生成类文件
javac HelloJNI.java
// 使用JDK提供的JNI命令工具，javah生成 .h头文件
javah helloJNI
```

> 注(使用ubuntu未发现该问题)：发现Linux环境下，javah居然不能从当前文件夹扫描到类文件，需要指定类路径 其中 -cp 就是-classpath

以下是利用javah生成的头文件：

```C
/* DO NOT EDIT THIS FILE - it is machine generated */
#include <jni.h>  // 这个头文件来自JDK 后面会提到
/* Header for class HelloJNI */

#ifndef _Included_HelloJNI
#define _Included_HelloJNI
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     HelloJNI
 * Method:    sayHello
 * Signature: ()V
 */
JNIEXPORT void JNICALL Java_HelloJNI_sayHello
  (JNIEnv *, jobject);

#ifdef __cplusplus
}
#endif
#endif
```

### 3.1.3 编写C实体方法

创建HelloJNI.c文件，编写实现体,引入刚刚生成的HelloJNI.h文件;

```C
#include<jni.h>
#include<stdio.h>
#include"HelloJNI.h"

JNIEXPORT void JNICALL Java_HelloJNI_sayhello(JNIEnv* env,jobject thisObject) {
    printf("Hello World!\n");
    return;
}
```

**注意：**这里定义的方法应该与javah生成的头文件中对应的方法名一致。

### 3.1.4 利用gcc生成动态链接库

利用gcc生成动态链接库，注意我们这里有引用到jni.h这个头文件，此文件由JDK提供，另外jni.h还引用了jni_md.h这个文件。必须引入这两个头文件，才能通过编译。

> jni.h和jni_md.h两个文件的所在地，本人JDK的安装路径在/usr/java下，每个人可能都不一样。

在gcc命令内通过指定（ -I 路径 ）引入库所在的目录，利用前面前面的头文件和源文件编译成动态链接库 hello.so

```Shell
gcc HelloJNI.c HelloJNI.h -I /usr/lib/jvm/java-8-openjdk-amd64/include/ -I /usr/lib/jvm/java-8-openjdk-amd64/include/linux -fPIC -shared -o hello.so
```

这时候可能会报错：fatal error: jni.h: 没有那个文件或目录，具体问题请参考后面小节。

### 3.1.5 运行Java程序

在Java文件中引用so动态库(步骤一中的``System.load``)，并使用java命令(或者IDE)运行java程序:

```C
// 使用java运行程序，这里实际上运行的是class文件
> java HelloJNI
Hello World!
```

或者将该xxxxx.so库添加到虚拟机运行环境,使用IDE进行运行:

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=MTMwZTE3ZWY1NzQ5ODQ5ZjJmZmQxMGMyZmEyODVkYmVfQ2tJc2RvOVRjY0p2NmRkSmJBZ0lPbDBheVFlNWhETVlfVG9rZW46UUNsbmJCOFdOb1Jvd2h4aUpaT2NrUE91bmliXzE3MDY1NDIzNzc6MTcwNjU0NTk3N19WNA)

```C
// 值设置为-Djava.library.path=/home/q/IdeaProjects/JNIDemo/libs，等号后面为libjnidemo.so文件所在的路径
// 或者使用IDE运行对应程序
```

## 3.2 传递参数并获取结果

### 3.2.1 创建Java文件

在本地文件夹中创建一个Java文件,在其中实现如下代码:

```Java
public class HelloJNI{
    static{
        System.load("home/mi/JNI/hello.so");
    }
    
    private native void average(int n1, int n2);
    
    public static void main(String[] args){
        System.out.println(new HelloJNI().average(1, 2));
    }
}
```

这里我们定义了一个native方法，此方法用于计算两数平均值。将两个Java int类型的值传递给C代码，使其计算并返回double值。然后输出到标准IO流

> 注：这里加载动态链接库的方式，改为了loadLibrary，只需提供库名即可，但是接下来在运行的时候，需要指定java.library.path，使其指向库所在的目录。

### 3.2.2 编译java代码

将java文件

```C
javac -h . helloJNI.java
```

javac 命令有 -h 选项，即编译并生成头文件，-h 对应的参数，是头文件生成的地址。这里"."表示，在当前目录下生成。

### 3.2.3 编写源文件

创建HelloJNI.c文件，编写实现体,引入刚刚生成的HelloJNI.h文件;

```C
#include<jni.h>
#include<stdio.h>
#include"HelloJNI.h"

// 命名规则:JAVA_ClassName_MethodName
JNIEXPORT jdouble JNICALL Java_HelloJNI_average(JNIEnv* env,jobject thisObject, jint n1, jint n2) {
    jdouble res = 0;
    printf("In c , the numbers average are %d and %d\n", n1, n2);
    res = (jdouble)(n1+n2) / 2.0;
    return res;
}
```

这里C获取到参数，并输出到标准IO流，然后将计算结果返回给Java。

### 3.2.4 利用gcc生成动态链接库

```Shell
gcc HelloJNI.c HelloJNI.h -I /usr/lib/jvm/java-8-openjdk-amd64/include/ -I /usr/lib/jvm/java-8-openjdk-amd64/include/linux  -fPIC -shared -o hello.so
```

由于Java环境变量已配置，可直接引用。生成的动态链接库名为demo.so

### 3.2.5 执行Java程序

在Java文件中引用so动态库(步骤一中的``System.load``)，并使用java命令(或者IDE)运行java程序:

```C
// 使用java运行程序
> java HelloJNI
// 以下是输出
In c , the numbers average are 1 and 2
1.5
```

## 3.3 C代码访问Java对象

### 3.3.1 创建Java程序

```Java
public class HelloJNI{
    static{
        System.load("home/mi/JNI/hello.so");
    }
    
    private int num = 123;
    private String str = "Hello Woeld!";
    
    private native void modify();
    
    public static void main(String[] args){
        HelloJNI hello = new HelloJNI();
        hello.modify();
    }
}
```

定义两个实例变量，一个为基本类型，另一个为对象类型。利用C代码对其进行更改，然后输出结果，校验其实例变量是否改变。

### 3.3.2 编译Java程序，并生成相关头文件

```Java
javac -h . helloJNI.java
```

### 3.3.3 编写源文件实现

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=NjI4ODVkOGQ2ZWM0ZDExZjI2NzAyMWMyMTQ0MDRhN2FfYXdmQTh2UXhUYmFLWkl6ZEUzVEtjNmRpemlVd1J0c2tfVG9rZW46Q0hYUWJxUFY2bzZTa3B4OEpaMGNxVWxFbk1oXzE3MDY1NDIzNzc6MTcwNjU0NTk3N19WNA)

根据上述描述的获取成员变量的步骤进行。

> 注：由于String在c语言中没有直接映射的类型，只能通过相关函数转换为以'\0'结尾的字符数组。

### 3.3.4 生成动态链接库

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=YTU3ZWQ4ZmQxYmYzZDU5MTZiMDM2ZTM1MmI0ZDdkOTNfYldWQWJzQTh1RVNmSnpLcmtpY1pQU3BnanhsaHNTM0lfVG9rZW46RTA2UGIwcWplb2VST3Z4OVNPV2M4Mm5ybmVrXzE3MDY1NDIzNzc6MTcwNjU0NTk3N19WNA)

> 注这里直接指定库名为libdemo3.so，至于为何要加前缀**lib**，请看前文

### 3.3.5 执行Java程序

![img](https://p2onpu7kg4.feishu.cn/space/api/box/stream/download/asynccode/?code=ZmRhNWUzYzZmNDg3NDMxNDg2MzMwNDliYjc3OTJlZTBfNHk5aDBRZHRhT3dDRnYwTk1IV0pxVTJ6SDl1WjdvbjBfVG9rZW46Qnlaa2JGenAxb0NudmR4UFNROWN6cFhOblZIXzE3MDY1NDIzNzc6MTcwNjU0NTk3N19WNA)

# 4.可能发生的问题

## 4.1 无法从系统中找到jni.h头文件

**问题原因:**这是因为无法从系统中找到`jni.h`头文件，这里我们可以手动导入`jni.h`到项目中，开头说了，jni是java的特征，所以`jni.h`文件在jdk当中

**解决办法:**

- windows去本地jdk安装目中找`<jdk安装目录>/include/jni.h`和`<jdk安装目录>/include/win32/jni_md.h`
- ubuntu去本地jdk安装目录找`<jdk安装目录>/include/jni.h`和`<jdk安装目录>/include/linux/jni_md.h`

将这两个文件拷贝到项目根目录中，然后将`#include <jni.h>`改为`#include "jni.h"`，尖括号表示从系统中查找，双引号表示从当前项目中查找。

或者在GCC生成动态连接库时,使用``-I``命令声明路径,直接引用这两个文件所在的绝对路径进行编译.

## 4.2 找不到"demo2"动态库

通过java命令的-D选项设定运行时库路径，但是仍然提示"找不到"demo2动态库。

经查阅，发现，在Linux系统中，共享库（也就是放入java.library.path路径下的动态库）必须符合这样的规范：

- Java代码：System.loadLibrary("XXXX");
- 库文件名：**lib**XXXX.so

**在Linux系统下共享库必须有lib作为前缀**

## 4.3 无法找到“hellojni”的类文件

**问题原因：**您的类有一个包,并且您尝试使用类文件而不是包根目录运行该命令

**解决办法：**在Src根目录下，使用PackageName.className的格式运行javah文件

**参考文档：**https://qa.1r1g.com/sf/ask/1339604101/

## 4.4 如何找到Linux下的JDK目录

```C
readlink -f $(which java) | sed "s:/bin/java::"
```

## 4.5 Exception in thread main java.lang.UnsatisfiedLinkError

**问题原因：**主要是由于在Java程序中调用了一个未被定义（在c文件中）的方法，可能是由于你声明的native函数和C代码中的函数头不一致导致的。

**解决方法：**查看java中声明的native方法和c中实现的方法是否一致，可以尝试看以下.h文件中生成的方法头。