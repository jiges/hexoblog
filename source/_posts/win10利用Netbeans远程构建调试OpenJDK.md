---
title: win10利用Netbeans远程构建调试OpenJDK
date: 2017-11-10 08:25:31
tags: [Netbeans,OpenJDK]
categories: OpenJDK
---
# NetBeans远程开发
在开发`linux`下的`C/C++`程序的时候，不可避免的会遇到在`windows`下进行开发，而在`linux`上编译和调试的场景。常见的解决方式是在`windows`上安装一个`linux`虚拟机，然后在`Linux`上编写代码，编译和调试。如果虚拟机不装图形界面会导致开发效率低，而如果安装了图形界面又会大量的占用系统资源。安装无界面的`linux`虚拟机，同时使用`NetBeans`的远程开发功能在`windows`下进行开发，可以很好的解决这个问题。
需要了解`NetBeans`远程开发请参考：[https://netbeans.org/kb/docs/cnd/remotedev-tutorial_zh_CN.html](https://netbeans.org/kb/docs/cnd/remotedev-tutorial_zh_CN.html) 
众所周知，在`windows`上编译及调试`openJDK`极为复杂，会遇到各种各样的问题，而且网络上的教程很少且大部分是以前的版本。而在`Linux`上编译调试`OpenJDK`相对简单的多。利用这一点，读者可在`windows`上安装`NetBeans`，利用其远程开发的功能，实现在`windows`上使用`Linux`环境对`OpenJdk`进行调试。
 <!-- more -->
# 准备`Linux`环境编译`openJDK`
既然是使用`Linux`环境，那`Linux`环境自然需要配置成能对`OpenJdk`进行编译。`Linux`环境编译`OpenJDK`网上已经有很多教程，在此只是简单的说下步骤
## 准备`Linux`环境。
我选择`Ubuntu17`版本在`VirtualBox`上安装，`make`版本原本是`4.1`版本，`gcc`和`g++`版本是`6.3.0`，但在编译过程中出现一些问题，后来将版本降至`4.7`，为了能实现远程调试，环境必须有`GDB`。`bootjdk` jdk7版本。
```c
$ make -v 
GNU Make 4.1
$ gcc -v
gcc version 4.7.4
$ g++ -v
gcc version 4.7.4
$ gdb -v
GNU gdb 7.12.50.20170314-git
```  

## 获取OpenJDK源码。
这里我选择`openjdk-8-src-b132-03_mar_2014.zip`
## 配置`config`（预构建）
```c
bash ./configure  --with-debug-level=slowdebug --with-target-bits=64 --with-boot-jdk=/home/ccr/jdk1.7.0_80
```  
在配置的过程中，会提示环境缺少的各种包，按照提示一次安装即可，在提示安装 `libX11-dev`依赖时，`ubuntu`怎么也安装不上，后来发现`x`大写了，改成小写就行了，配置完成后如下图：
![配置完成](3433091-cf63f9d61aff491c.png)
## `make all`(构建)
在构建时有可能遇到以下问题，解决方法也在下面：
* ### `This OS is not supported`:
在`/openjdk/hotspot/make/linux/Makefile`文件的228行，写明了该版本`openJDK`所支持的`OS`版本，通过`uname -r`命令查出系统的版本，添加到228行后面重新编译即可（`make clean`）。
![uname-r](3433091-949599abc82796ee.png)
* ### `invalid option -- '/'`
![clipboard.png](3433091-34c85d83d29b3642.png)
解决办法：删除66-67行之间的那段
```c
$ vim hotspot/make/linux/makefiles/adjust-mflags.sh
63 MFLAGS=`
64         echo "$MFLAGS" \
65         | sed '
66                 s/^-/ -/
                     s/ -\([^        ][^     ]*\)j/ -\1 -j/
67                 s/ -j[0-9][0-9]*/ -j/
68                 s/ -j\([^       ]\)/ -j -\1/
69                 s/ -j/ -j'${HOTSPOT_BUILD_JOBS:-${default_build_jobs}}'/
70         ' `
```  
* ### `invalid suffix on literal`
![clipboard.png](3433091-49695fd38841bd8c.png)
![clipboard.png](3433091-f66ab57efafab79e.png)
这是因为`gcc`和`g++`版本过高导致，降低`gcc`版本至`4.7`.
```c
//1 安装
$ sudo apt-get install -y gcc-4.7
$ sudo apt-get install -y g++-4.7
//2 重新建立软连接
$ cd /usr/bin
$ sudo rm -r gcc
$ sudo ln -sf gcc-4.7 gcc
$ sudo rm -r g++
$ sudo ln -sf g++-4.7 g++
```  
* ### `-Werror=deprecated-declarations`
![clipboard.png](3433091-a2b772bd714d673f.png)
解决办法
```c
$ vim hotspot/make/linux/makefiles/gcc.make
# Compiler warnings are treated as errors
# WARNINGS_ARE_ERRORS = -Werror #注释这一行
```  
构建完成后出现如下图所示内容，说明构建成功
![clipboard.png](3433091-0df8893f2ea2b223.png)
# `windows` `NetBeans`构建调试
远程调试一般有两种方式，一种是将本地源码通过`sftp`的方式复制到目标机器，目标机器进行编译后的结果在复制到本地，这种方式针对小型项目效果还是不错的，但是对`OpenJDK`这种大型项目，复制无疑是非常耗时的，有时还会出现异常。另一种方式是将`windows`的文件夹共享至网络中，`linux`通过`mount`命令将网络中的文件夹挂在至`Linux`系统，并且赋予读写和执行权限，这样`windows`和`Linux`就能共同操作统一文件夹。毫无疑问本教程使用第二种方式。
## 创建文件夹
在D盘新建`jvm`文件夹（作为共享文件夹），将`OpenJDK`解压至该文件夹。安装`NetBeans`（我选择最新版本8.2）。
## 分享文件夹
分享`jvm`文件夹。在`windows`上操作如下，选择一个账户进行共享，共享结束后，右键->属性，查看共享状态。
![clipboard.png](3433091-b5b4b9bbc0ef5f7e.png)
![clipboard.png](3433091-79cfd9a78d692fe5.png)
![clipboard.png](3433091-4e0b8dd48584c661.png)
## 挂载
在`Linux`上使用`mount`命令挂载
```c
$ id ccr
uid=1000(ccr) gid=1000(ccr) groups=1000(ccr)
$ sudo mount //192.168.2.176/jvm /home/ccr/jvm/ -o username=ccr,password=ccr,gid=1000,uid=1000
```  
参数依次是资源文件夹的`IP`和文件夹，挂载到目标机器的目录（必须先创建），`username`文件夹共享的`windows`账户，`password`密码，`gid`和`uid`是控制挂载后控制该目录的所有者，可以通过`id username`来获取。挂载成功后使用`ll`命令到`jvm`文件夹查看结果。
![clipboard.png](http://upload-images.jianshu.io/upload_images/3433091-c0d5b0f94e934ff9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/620)
## `NetBeans`构建主机。
打开`NetBeans`，窗口->服务->右键  `c\c++`构建主机  添加主机。填写目标机器的IP及操作用户（`Linux`上必须安装`ssh`），我这里使用`ccr`用户。 最后项目文件的访问方式一定要选择【系统级别文件共享（`NFS`,`Samba`等）】，添加主机后 位主机设置路径映射。
![clipboard.png](3433091-e4c0fa4fcda389b6.png)
![clipboard.png](3433091-26cea2b1b64425d5.png)
![clipboard.png](3433091-c6a24ce48352da6a.png)
![QQ截图20171110151648.png](3433091-01f33f5c624cb110.png)
![QQ截图20171110151807.png](3433091-214d6e2b941518d0.png)
## `NetBeans`构建项目。
* 文件->新建项目，选择，`c/c++`，基于现有源代码的`c/c++`项目
![clipboard.png](3433091-f64aac29218fb0d5.png)
* 选择`openJdk`目录，选择构建的主机，选择定制
![clipboard.png](3433091-6fdd756df1f4f842.png)
* 写上预定以参数，这个参数和之前编译时的参数一直就行了
```c
./configure  --with-debug-level=slowdebug --with-target-bits=64 --with-boot-jdk=/home/ccr/jdk1.7.0_80
```  
![clipboard.png]( 3433091-b161f3c3a41bca4a.png)
* 一直点下一步，直到最后一步，项目名称改成`openjdk8u`
![clipboard.png](3433091-ac9205fd316af1ed.png)
* 点击完成后，项目开始配置和构建，在构建过程中遇到的问题，可以参考上面的编译过程去解决，然后重新清理和构建，直到构建成功为止。构建大概30分钟。
## 错误处理
构建过程中出现错误。`failed to create symbolic link： Operation not supported`
因为由`windows`共享的方式，在`Linux`上挂载时使用的`cifs`文件系统，可以用`df -T`来查看，而`cifs`文件系统是创建不了软连接的（`symbolic link`），所以`windows`的共享方式需要改成`NFS`的方式（读者可以先去了解`HFS`和`cifs`的区别）。
![clipboard.png](3433091-4960ac9fcea91965.png)
`windows`要想通过`nfs`的方式共享文件，需要安装`nfs`服务，这里我选择用`haneWin NFS`。下载和文档链接如下：
 [https://www.hanewin.net/nfs-e.htm](https://www.hanewin.net/nfs-e.htm) 
 [https://www.hanewin.net/doc/nfs/nfsd.htm](https://www.hanewin.net/doc/nfs/nfsd.htm) 
软件需要破解，可在网上找到注册机。下图是配置图。
![clipboard.png](3433091-5ef1ad52e670fc56.png)
![clipboard.png](3433091-2152e47cb6e14e17.png)
![clipboard.png](3433091-2b90dbc679214b9f.png)
上面的参数中`-mappall:1000:1000 -exec` 是必须的为客户端设置文件夹所属账户以及执行权限，设置完成后重启服务（如果是最新版本直接点击重启服务是可以的，或者到`windows`的服务管理重启）
重启后可在命令行中 `showcount -e` 进行查看。
客户端挂载
```c
//先安装nfs客户端在挂载，原来的挂载取消
$ sudo apt-get install nfs-common
$ sudo mount -t nfs 192.168.2.176:/jvm /home/ccr/jvm/
//挂载成功后，进入目录测试一下能否创建软连接
$ touch foo  --创建文件
$ ln -s bar foo
```  
最后运行成功
![clipboard.png](3433091-125985cd43b66e47.png)
# `NetBeans`运行项目。
* 准备`Hello World class`文件（很简单的`javaHelloWorld`，用`openjdk`编译好的`javac`去编译，或者用该版本以下版本去编译。），放到共享文件夹内。我的放在`jvm/javatest`目录下。
* 右键项目->属性->运行->编辑运行命令->输入类路径和类名
![clipboard.png](3433091-5148f13e2c1de5a2.png)
![clipboard.png](3433091-de8b30d01ef478e5.png)
* 好了现在可以运行项目了，点击运行->运行项目，`netbeans`会询问你用什么可执行文件来运行，选择`java`即可
![clipboard.png](3433091-8573cd61828ff74c.png)
运行结果如下：
![clipboard.png](3433091-ba558ae92a1e7273.png)
# NetBeans调试项目。
`System.out.println(...)`实际上调用的是`jdk/src/share/native/java/io/io_util.c` 文件中的 `writeBytes` 方法，定位到这个方法打上断点。点击调试项目，如果出现`SIGSEGV` 警告，忽略往前继续。最终执行到改断点，你就能看到`jvm`正在干什么。
![clipboard.png](3433091-b24ad588d9cef967.png)
# 总结
好了，以上就是利用`NetBeans`远程开发和`Linux`虚拟机进行的`OpenJDK`的构建和调试。构建过程中有很多错误，读者要有耐心，利用虚拟机的快照功能对虚拟机进行备份。尽量参考官方的文档。

