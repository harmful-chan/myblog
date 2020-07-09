以前看过下韦东山老师TQ2440linux开发的视频**https://www.bilibili.com/video/BV1pW411L7UX**，现在也忘得七七八八了。不得不说他讲的思路清晰得不得了，不尽让人佩服得五体投地。然后最近在学网络驱动回顾了一下以前看过的教程，发现自己当时弃坑简直英明┐(´∇｀)┌。不然头发保不住了。因为韦哥讲的是arm架构的嘛，如果学x86架构建议和孟宁老师的《linux内核分析》使用更佳**https://www.bilibili.com/video/BV1GJ411g7Gn?from=search&seid=6044318456043555330**。

以下文章用的环境是 **CentOS7.6  x86_64**，**系统内核4.4.224-1.el7.elrepo.x86_64**，**linux3.16.84源码**，**gcc4.8.5**，**gdb7.16.1**，**VSCode**。其中gcc和gdb用版本把系统自带的版本更到最新就行。下面开始搭建环境参考http://rainlin.top/archives/201但他是ubuntu+macos和本机开发的，所以看着来吧。

## 一、用qemu跑一个自己编译的linux内核镜像并运行自己的hello world

### 1. 安装qemu，gcc，及其依赖

首先我们需要升级一下我们的自带软件和系统内核，可以参考这个**https://www.cnblogs.com/harmful-chan/p/12992873.html**，其实最好是可以把内核升到3.18.6这样可以省去很多包依赖的麻烦，但其他版本也没坏。

升级好内核之后，安装qemu，一款软件模拟硬件参数的虚拟机，其实他的效率不怎么样，但是它简单啊。运行`yum -y install qemu`如果说系统找不到这个软件，运行`yum -y install epel-release && yum clean all && yum makecache`导入epel的源，里面才是大部分我们白嫖党用到的软件。然后`qemu-system-x86_64 --version`有输出就安装好了。（qemu系列说明在最后）gcc的话用`yum -y install gcc make` 安装。

![image-20200530180200879](http://chf.cool:9000/myblog/img/image-20200530180200879.png)

### 2. 编译x86_64架构linux内核及运行

首先我们需要下载linux源码**http://ftp.sjtu.edu.cn/sites/ftp.kernel.org/pub/linux/kernel/v3.x/linux-3.18.84.tar.xz**这个好像是清华的源下的快点，需要不同版本把linux的版本改下直接下载就行。解压`tar -xvf linux-3.16.84.tar.xz `然后`cd linux-3.16.84 && make help`能查看官方提供的配置，里面就包含`1386_defconfig和x86_64_defconfig`分别对应两种不同的架构的模块配置。为了确保依赖`yum -y install  glibc glibc-utils glibc-devel`安装编译C程序基本的库，如果等下编译内核错误，应该是漏了这几个。然后就可以进到内核目录`make x86_64_defconfig && make -j2`（j后面跟的是用于编译的cpu线程数量），然后就过漫长的等待，`ls -la linux-3.16.84/arch/x86_64/boot`查看编译好的内核。**如果编译失败，请先执行make mrproper清理全部生成文件，再从配置开始**

![image-20200530175940690](http://chf.cool:9000/myblog/img/image-20200530175940690.png)

### 3. 编写一个hello程序用于测试

`cd ~/ && mkdir hello && vi hello.c`然后输入一下内容，

<details>
    <summary>hello.c</summary>
<pre><code>
#include "stdio.h"
int main()
{
    while(1)
    {
          printf("hello world\n");
          sleep(1);
    }
    return 0;
} </code><pre></details>

然后保存退出。安装静态链接库用于把库都打包成一个独立的文件`yum -y install glibc-static`。执行编译命令把程序生成一个名为init的可执行文件`cd ~/ && gcc -o hello/init hello/hello.c -pthread -static `，然后把init打包起来`find hello/init | cpio -o -Hnewc | gzip -9 > rootfs.img`。**在x86_64的系统中不建议用-m32参数编译成32位程序，其他linux发行版还好，centos真的很少32位的库，不然找依赖会找到你怀疑人生**

最好如果没报错的话就用qemu把他们都运行起来` qemu-system-x86_64 -kernel linux-3.16.84/arch/x86/boot/bzImage -initrd hello/rootfs.img -nographic --append "console=ttyS0"`-kernel执行驱动内核，-initrd指定启动的第一个程序，-nographic不弹出新窗口，--append指定追加额外参数，console=ttyS0指定用本窗口的串口设备输出加载信息。然后看到hello world输出那你就成功了第一步了。退出qemu用`（ctrl + a ）松开马上按x `

![image-20200530182113235](http://chf.cool:9000/myblog/img/image-20200530182113235.png)

## 二、给内核添加调试信息，gdb调试内核

### 1. 生成development版内核

上一部分所生成的bzImage相当于一个release版本的内核，下面我们要生成一个带调试信息的development版内核。先安装配置菜单的包`yum -y install ncurses-devel`然后到我们的linux目录下清理现场，然后运行配置菜单`cd linux-3.16.84 && make mrproper && make menuconfig `然后会弹出一蓝蓝的框。

![image-20200530185100472](http://chf.cool:9000/myblog/img/image-20200530185100472.png)

然后把添加调试信息项打上星号，光标移动到对应位置按空格就行（默认已经打上了，位置在kernel hacking --> Compile-time checks and compiler options -->[*]compile the kernel with debug info）然后SAVE再EXIT就行。配置之后`make -j2`。上一步我们已经成功生成内核的话，这一步就没多大问题，只是时间比之前长5倍左右...

### 2. 本地安装gdb调试

由于系统下载的GDB在调试内核时会出现“Remote ‘g’ packet reply is too long”的错误，我们需要修改GDB的源码然后编译gdb。我们先按安装官方的gdb`yum -y install gdb`不为别的，只图yum会帮我们把gdb依赖的库都会下载下来哈哈(￣▽￣)~*，然后再安装g++，在centos有点特别要这样`yum install gcc-c++`

gdb源码`wget https://mirror.bjtu.edu.cn/gnu/gdb/gdb-7.6.1.tar.gz `版本可以自己改，然后解压`tar -xvf gdb-7.6.1.tar.gz `，修改gdb-7.6.1/gdb目录下remote.c`vi gdb-7.6.1/gdb/remote.c`修改如下参考

<details>
    <summary>gdb-7.6.1/gdb/remote.c</summary>
<pre><code>
    /* Further sanity checks, with knowledge of the architecture.  */
    // if (buf_len > 2 * rsa->sizeof_g_packet)
    //   error (_("Remote 'g' packet reply is too long (expected %ld bytes, got %d "
    //      "bytes): %s"),
    //    rsa->sizeof_g_packet, buf_len / 2,
    //    rs->buf.data ());
    if (buf_len > 2 * rsa->sizeof_g_packet) {
    rsa->sizeof_g_packet = buf_len;
    for (i = 0; i < gdbarch_num_regs (gdbarch); i++){
            if (rsa->regs[i].pnum == -1)
                continue;
            if (rsa->regs[i].offset >= rsa->sizeof_g_packet)
                rsa->regs[i].in_g_packet = 0;
            else
                rsa->regs[i].in_g_packet = 1;
        }
    }
</code><pre></details>

修改之后`cd gdb-7.6.1 && ./configure && make -j2`配置并编译，这是可能会遇到很多编译问题要自己查了，我查了很久按我上面方法是最快的。编译好删除官方的gdb`yum remove gdb`，然后安装我们自己编译的`make install`会把可执行文件放到/usr/local/bin/中，gdb -v有输出就ok了。

![image-20200530191705995](http://chf.cool:9000/myblog/img/image-20200530191705995.png)

接下来就是用gdb命令行调试内核，先打开两个窗口，左运行gdb右运行linux。左`gdb linux-3.16.84/vmlinux`vmlinux是代码索引文件，右`qemu-system-x86_64 -kernel linux-3.16.84/arch/x86/boot/bzImage -initr d hello/rootfs.img -nographic --append "console=ttyS0" -s -S`-s -S 启动之后等待有控制信息再往下执行。

先运行虚拟机，然后运行gdb，在gdb窗口连接本地的1234端口，然后再start_kernel打断点然后顺序执行。

![running](http://chf.cool:9000/myblog/img/running.gif)

## 三、VSCode搭建远程调试环境

vscode上微软官网下载就行，然后我们需要两个插件VScode Remote就不说了，走ssh协议参考**https://www.jianshu.com/p/0f2fb935a9a1**，官方**https://code.visualstudio.com/docs?start=true**。然后C/C++语法提示和gdb debug。记得是安装再服务器上，小心选错。



![image-20200530194341225](http://chf.cool:9000/myblog/img/image-20200530194341225.png)

然后配置vscode的启动配置文件，在目录新建一个.vscode/launch.json文件，有时候vscode你按f5会帮你自动生成模板的，但快捷键我不知道哪个....写入如下信息。其实这个c++程序的调试模板，所以按着c++的程序配置就行。

<details>
    <summary>.vscode/launch.json</summary>
<pre><code>
    "version": "0.2.0",
    "configurations": [
        {
            "name": "gcc - Build and debug active file",
            "type": "cppdbg",    //c++程序
            "request": "launch",
            "miDebuggerServerAddress": "127.0.0.1:1234",    //运行内核的机器ip的1234端口
            "program": "${workspaceFolder}/vmlinux",    //索引文件目录linux-3.16.84/vmlinux
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "/usr/local/bin/gdb"    //gdb在远程机器上的绝对路径
        }
    ]
</code><pre></details>

![image-20200530194717853](http://chf.cool:9000/myblog/img/image-20200530194717853.png)

![debug](http://chf.cool:9000/myblog/img/debug.gif)



## 额外参考

**qemu系列的区别**http://blog.jcix.top/2016-11-02/qemu_commands/

**-pthread和-lpthread的区别**https://www.iteye.com/blog/chaoslawful-568602

**centos7.6编译32位程序**`yum install glibc-static &&  yum -y install glibc-devel.i686 libstdc++-devel.i686 && gcc -o init hello.c -pthread -static -m32 `







