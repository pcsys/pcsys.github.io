---
title: 在ubuntu中构建x86_64-linux for windows
tags: pcsys
---

## 构建

本篇博文介绍如何在Ubuntu上构建一个在Windows中运行（`*.exe`）并为Linux编译可执行文件（`*.elf`）的交叉编译器。这个交叉编译器称为“x86_64-linux for windows”。

### 为什么要构建x86_64-linux for windows

在搭建操作系统的开发环境中，于渊的《Orange S: 一个操作系统的实现》给出了很详细的建议和方案。总结一点：Linux系统是开发操作系统的首选平台。但是，由于无法完全抛弃Windows，所以通常将Linux系统安装在虚拟机中来实现跨系统访问的。于是，这让操作系统的开发过程变成这样：

    1. 首先，在Windows中编写程序；
    2. 然后，切换到虚拟机中的Linux中编译程序；
    3. 最后，返回到Windows调试程序。

可以看到，整个开发过程是非常繁琐的很浪费时间。为了简化开发操作，需要所用到的开发工具的Windows版本，以求在Windows完成所有的操作，而不用切换到虚拟机中的linux系统。很幸运，大部分的工具都找到了对应的Windows版本。其中，最重要的gcc编译器的Windows版本是[MinGW-w64](https://sourceforge.net/projects/MinGW-w64)。

MinGW-w64只是一个编译器，它不同于[cygwin](http://cygwin.com/)和[msys2](https://www.msys2.org/)还附带了一个庞大的类UNIX模拟环境。这即是它的缺点，但也是它的优点！

在cmd或PowerShell中，使用MinGW-w64可以很方便地编译产生一个在Windows上运行的可执行文件（`*.exe`），操作如同在Linux中一样方便，并且可以将编译命令写入一个bat脚本来一步搞定！是的，一切都是那么自然！

在[kos docs](http://kos.enix.org/docs.php)（http://kos.enix.org/docs.php）网站中的[plainbin.pdf.gz](http://kos.enix.org/pub/plainbin.pdf.gz)文件提供了如何制作平坦纯二进制文件的教程。当我希望使用它编译链接产生一个无格式的平坦纯二进制文件（`*.bin`）时，却总是出现下列错误提示：

```bash
# 不能在非PE输出文件'main.bin'上执行PE操作
ld.exe: cannot perform PE operations on non PE output file 'main.bin'
```

这是没办法的，MinGW-w64只能编译产生在Windows上运行的可执行文件（`*.exe`），不能编译产生在Linux上运行的可执行文件（`*.elf`）。要想使用gcc制作平坦纯二进制文件，只能通过elf格式的可执行文件 。

在[OSDev Wiki](https://wiki.osdev.org/)中提供了一篇[GCC_Cross-Compiler](https://wiki.osdev.org/GCC_Cross-Compiler)教程，从中找到了一个解决方案，那就是**构建一个交叉编译器**。在这个教程的底部提供了许多为每个系统预构建的交叉编译器。其中，为Windows主机构建的是[i686-/x86_64-elf 7.1.0 target + GDB](https://github.com/lordmilko/i686-elf-tools)。打开链接，进入它的github。根据它所提供的脚本，可以成功地构建了一个在Windows中运行并为Linux编译可执行文件的交叉编译器。但是，这个编译器是Freestanding版本的（不支持头、库和运行时文件）。

在网上，也有很多教程演示了如何构建一个Hosted版本的交叉编译器。比较全面的教程是[How to Build a GCC Cross-Compiler](https://preshing.com/20141119/how-to-build-a-gcc-cross-compiler/)（网上有翻译的[中文版](https://blog.csdn.net/fickyou/article/details/51671006)）和[gcc 9.2 交叉编译器构建过程](https://www.cnblogs.com/summitzhou/p/12503647.html)。（这里，感谢开发者的无私分享！让我从中找到了很多有用的信息。）

当我认为它能完全胜任我的工作时，又发现了一个问题：不支持多库。也就是，不能使用`gcc -m32`选项编译产生32位代码。不能产生16位代码也就算了，连32位代码也不能产生，这如何能在保护模式大展拳脚？！

为了构建一个支持多库的宿主交叉编译器，尝试了很多构建方法。在处理了无数个莫名奇妙的构建错误之后，终于，还是在崩溃的边缘构建成功了！为了方便记忆，记录下这篇笔记！

### 在虚拟机中安装Linux系统

首先需要在虚拟机中安装一个基于Debian的Linux发行版，推荐`Ubuntu + workstation pro`。

- [Ubuntu Linux](https://cn.ubuntu.com/)：如果要构建x86_64-linux for windows时，下载64位Ubuntu；如果要构建i686-linux for windows时，下载32位Ubuntu。
- [Workstation Pro](https://www.vmware.com/cn/products/workstation-pro/workstation-pro-evaluation.html)：选择Workstation Pro的一个主要原因是它支持简易安装。这可以省去很多安装步骤。

在Workstation Pro中安装Ubuntu是很简单的，直接点击菜单“新建虚拟机”，根据提示操作即可。唯一需要注意的是，为了避免空间不足，硬盘的大小至少需要30G以上（推荐50G），因为构建过程将会占用几十G的磁盘空间。

当安装好Ubuntu之后，只需要进行一些简单的配置就可以了。

**第一步，重置root密码。**

在桌面空白处右击，选择“Open in Terminal”打开终端，并执行下列命令来重置root密码：

```bash
sudo passwd root
```

**第二步，安装VMware Tools。**

点击菜单“虚拟机→安装VMware Tools”，然后，在Ubuntu的文件管理器中解压VMware-tools并将vmware-install.pl拖拽到终端中，以root用户运行，第一个提示输入`yes`，之后的输入全部`回车`即可，直到安装完成。

注销或重启Ubuntu之后，就能在主机和虚拟机之间执行复制粘贴操作了。

**第三步，将默认软件源修改成国内镜像站。**

首先，进入root，将sources.list备份，并用gedit文本编辑器打开。

```bash
# 进入root
su root

# 备份sources.list
cp -v /etc/apt/sources.list /etc/apt/sources.list.backup

# 打开sources.list
gedit /etc/apt/sources.list
```

然后，进入[清华大学开源软件镜像站](https://mirrors.tuna.tsinghua.edu.cn/)，找到Ubuntu镜像，点击旁边的问号打开[Ubuntu镜像使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)。选择你的Ubuntu版本之后将软件源代码复制到sources.list中。

```bash
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-proposed main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ groovy-proposed main restricted universe multiverse
```

注意，软件源中的“groovy”字符表示的是Ubuntu 20.10的系统代号。对于其他版本 ，需要执行`lsb_release -c`命令查看当前系统的系统代号。然后，将软件源中的所有“focal”字符替换成获得的系统代号。

最后，执行下列命令更新软件源。

```bash
# 更新的软件包列表信息
apt-get update
```

这样，就可以安装软件了。

**第四步，禁用屏幕自动黑屏。**

在桌面空白处右击，选择“settings”打开设置，找到Power（电源），将Blank Screen（黑屏）设置成“Never（从不）”。

### 安装构建工具和依赖包

gcc和glibc都是高度依赖的软件，它们需要配合许多其他工具和依赖包才能构建。根据[Installing GCC](https://gcc.gnu.org/install/)中的“[1.Prerequisites](https://gcc.gnu.org/install/prerequisites.html)”和[glibc manual](https://www.gnu.org/software/libc/manual/)中的“[C.3 Recommended Tools for Compilation](https://www.gnu.org/software/libc/manual/html_node/Tools-for-Compilation.html)”章节所述，大致需要安装下列软件、库和依赖包：

```bash
apt-get install \
    autoconf \
    autogen \
    automake \
    bash \
    binutils \
    bison \
    bzip2 \
    dejagnu \
    diffutils \
    expect \
    flex \
    g++ \
    g++-multilib \
    gawk \
    gcc \
    gettext \
    gfortran \
    git \
    gnat \
    gperf \
    guile-3.0 \
    gzip \
    libc6-dev-i386 \
    libgmp-dev \
    libisl-dev \
    libmpc-dev \
    libmpfr-dev \
    libtool \
    libzstd-dev \
    m4 \
    make \
    MinGW-w64 \
    patch \
    perl \
    sphinxsearch \
    ssh \
    tar \
    tcl \
    texinfo \
    texlive \
    unzip \
    wget -y

apt-get install build-essential -y
```

当然，这些工具并不会全部都用到，但是全部安装可以避免日后想增添某些功能时，不会出现因工具缺失而再次产生某些构建错误。

#### MinGW-w64：gcc for windows

[MinGW-w64](http://www.mingw-w64.org/doku.php)是一个免费开源的交叉编译器，用于编译产生在Windows上运行的可执行文件。MinGW-w64主要由binutils、gcc和gdb组成，并包含一套Windows专用的系统头文件和静态导入库以允许使用Windows API。

MinGW-w64支持2种目标三元组：

- `i686-w64-mingw32`：用于编译产生32位Windows可执行文件
- `x86_64-w64-mingw32`：用于编译产生64位Windows可执行文件

当`--target=x86_64-linux`时，使用`x86_64-w64-mingw32`； 当`--target=i686-linux`时，使用`i686-w64-mingw32`。

在[MXE](https://mxe.cc/)官网中的[List of Packages](https://mxe.cc/#packages)中给出了一个列表，可以查看哪些软件能够使用MinGW-w64移植到Windows中。

### 导入系统变量

执行下列命令将多次重复用到的选项和参数当做作变量导入到系统环境中，以方便批量修改。这些变量主要包括：源代码的下载目录（也是构建目录）和版本号、以及系统类型和安装目录等，它们的值都是根据需求自定义的。

```bash
# 构建目录
export srcdir=$HOME/src

# 版本号 (注意, 推荐当前系统相同或发布时间接近的版本号, 以保证最大的构建成功率)
export kernel_ver=5.8                    # uname -a
export binutils_ver=2.35.1
export gcc_ver=10.2.0
export glibc_ver=2.32                    # ldd --version
export pthread_ver=2.5

# 系统类型
export host=x86_64-w64-mingw32           # 主机系统
export target=x86_64-linux               # 目标系统

# 安装目录
export sysdir=/usr/cross                 # x86_64-linux for linux 
export optdir=/opt/cross                 # x86_64-linux for windows

# 系统根目录
export sysroot=$sysdir/$target           # x86_64-linux for linux 
export optroot=$optdir/$target           # x86_64-linux for windows

# 系统路径
export PATH=$sysdir/bin:$PATH
```

一个支持多库的x86_64-linux for windows交叉编译器主要由[sysheader](https://www.kernel.org/)（来自Linux Kernel）、[binutils](http://www.gnu.org/software/binutils/)和[glibc](https://www.gnu.org/software/libc/)和[gcc](https://gcc.gnu.org/)四部分组成。可选地，包含[gdb](http://www.gnu.org/software/gdb/)用于交叉调试。

大多数情况下，它们之间并不是所有的版本都能正常工作，只有彼此发布时间相近的版本才能更容易构建成功。因此，最好是选择与当前构建系统最接近或相同的版本号。下列命令可以查看它们的版本号：

```bash
cat /proc/version   # 查看Linux, gcc, binutils的版本
ldd --version       # 查看glibc的版本
```

`$HOME`是系统预设的系统变量。在普通用户中，`$HOME=/home/user`（user是系统的登陆用户名）；在管理用户中，`$HOME=/root`。

### 下载源代码

因此，执行下列命令将所有需要的源代码下载到`$srcdir`中，并解压。

```bash
# 新建源代码目录
rm -rf $srcdir
mkdir -p $srcdir

# 下载
cd $srcdir
wget https://mirrors.tuna.tsinghua.edu.cn/kernel/v5.x/linux-$kernel_ver.tar.gz
wget https://mirrors.tuna.tsinghua.edu.cn/gnu/binutils/binutils-$binutils_ver.tar.gz
wget https://mirrors.tuna.tsinghua.edu.cn/gnu/gcc/gcc-$gcc_ver/gcc-$gcc_ver.tar.gz
wget https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-$glibc_ver.tar.gz
wget https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-linuxthreads-$pthread_ver.tar.bz2

# 解压
cd $srcdir
tar -xf linux-$kernel_ver.tar.gz
tar -xf binutils-$binutils_ver.tar.gz
tar -xf gcc-$gcc_ver.tar.gz
tar -xf glibc-$glibc_ver.tar.gz

# 解压glibc-linuxthreads
cd $srcdir/glibc-$glibc_ver
tar -xf ../glibc-linuxthreads-$pthread_ver.tar.bz2
```

所有的GNU软件都可以在其官网（[ftp://ftp.gnu.org/gnu](ftp://ftp.gnu.org/gnu)）中下载。由于国内速度原因，可以使用http://ftpmirror.gnu.org/会自动选择最近和最新的镜像站进行下载。如果这两个地址都下载失败，也可以从国内的某些开源镜像站下载，这里推荐[清华大学开源软件镜像站](https://mirrors.tuna.tsinghua.edu.cn/)。

#### 下载gcc依赖库

根据[Installing GCC](https://gcc.gnu.org/install/)中的“[Downloading the source](https://gcc.gnu.org/install/download.html)”所述，gcc都会在构建过程中自动执行源代码目录中的/contrib/download_prerequisites脚本来下载它依赖的[gmp](http://gmplib.org/)、[mpfr](https://www.mpfr.org/)、[mpc](http://www.multiprecision.org/mpc/)、[isl](http://isl.gforge.inria.fr/)和[cloog](http://www.cloog.org/)库（cloog库不是必须的）。如你所见，成功与否完全取决于国内网速。

所以，在这里，将它们的源代码从其官网下载到`$srcdir`目录之后，制作一个符号链接到gcc源码目录，来手动完成download_prerequisites脚本的工作。需要注意的是，下载的gmp、mpfr、mpc和isl版本应该与download_prerequisites脚本中指定的相同，以保证最大的版本兼容性和构建成功率。

```bash
# 导入版本号
export gmp_ver=6.1.0
export mpfr_ver=3.1.4
export mpc_ver=1.0.3
export isl_ver=0.18 
export cloog_ver=0.18.1

# 下载
cd $srcdir
wget https://gcc.gnu.org/pub/gcc/infrastructure/gmp-$gmp_ver.tar.bz2
wget https://gcc.gnu.org/pub/gcc/infrastructure/mpfr-$mpfr_ver.tar.bz2
wget https://gcc.gnu.org/pub/gcc/infrastructure/mpc-$mpc_ver.tar.gz
wget https://gcc.gnu.org/pub/gcc/infrastructure/isl-$isl_ver.tar.bz2
wget https://gcc.gnu.org/pub/gcc/infrastructure/cloog-$cloog_ver.tar.gz

# 解压
cd $srcdir
tar -xf gmp-$gmp_ver.tar.bz2
tar -xf mpfr-$mpfr_ver.tar.bz2
tar -xf mpc-$mpc_ver.tar.gz
tar -xf isl-$isl_ver.tar.bz2
tar -xf cloog-$cloog_ver.tar.gz

# 符号链接
cd $srcdir/gcc-$gcc_ver
ln -s ../mpfr-$mpfr_ver mpfr
ln -s ../gmp-$gmp_ver gmp
ln -s ../mpc-$mpc_ver mpc
ln -s ../isl-$isl_ver isl
ln -s ../cloog-$cloog_ver cloog
```

### 构建x86_64-linux-gcc

根据[维基百科](https://en.wikipedia.org/wiki/Main_Page)中的[Cross compiler](https://en.wikipedia.org/wiki/Cross_compiler)词条所述，如果某个编译器在平台A（构建系统）上构建，但在平台B（主机系统）上运行，并为平台C（目标系统）生成可执行文件，那么这个编译器称为“[Canadian Cross](https://en.wikipedia.org/wiki/Cross_compiler#Canadian_Cross)”。

为了构建一个加拿大交叉编译器，需要先构建分别这三个平台生成可执行文件的编译器：

1. 在构建系统运行，为**构建系统**生成可执行文件的本地编译器。
2. 在构建系统运行，为**主机系统**生成可执行文件的交叉编译器。
3. 在构建系统运行，为**目标系统**生成可执行文件的交叉编译器。

根据需求，这三个编译器的名称应该是：

1. gcc：用于为构建系统（Linux）生成可执行文件。

2. x86_64-w64-mingw32-gcc：用于为主机系统（Windows）生成可执行文件。

3. x86_64-linux-gcc：用于为目标系统（Linux）生成可执行文件。


由于前两个已经被安装好了，所以在构建x86_64-linux for windows之前，只需要构建x86_64-linux-gcc。

因为x86_64-linux-gcc将在Linux中运行并为Linux生成可执行文件，所以它的主机和目标系统类型为：

- `--host=x86_64-pc-linux-gnu`
- `--target=x86_64-linux`

根据[crosstool-ng文档](http://crosstool-ng.github.io/docs/)中的[9.How a toolchain is constructed](http://crosstool-ng.github.io/docs/toolchain-construction/)章节所述，构建x86_64-linux for linux的步骤一般是：

1. 构建sysheader
2. 构建binutils
3. 构建bootstrap gcc
4. 构建glibc
5. 构建gcc

这里，由于x86_64-linux是x86_64-pc-linux-gnu的别名。所有，可以直接指定`--target=x86_64-pc-linux-gnu`；来跳过x86_64-linux for linux的构建。

#### 构建sysheader

执行下列命令从Linux Kernel源码中将x86架构的系统头（sysheader）文件安装到`$sysroot/usr`目录中。

```bash
cd $srcdir/linux-$kernel_ver
make headers_install ARCH=x86 INSTALL_HDR_PATH=$sysroot/usr
```

根据[headers_install.rst](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/kbuild/headers_install.rst)文档所述，安装sysheader文件最简单的方法是在内核源码的根目录中运行`make headers_install`命令。这个命令接受两个可选参数：`ARCH`和`INSTALL_HDR_PATH`。

`ARCH`选项指定要为哪个架构生成头文件。在大多数情况下，架构的名称与/arch目录中的子目录名相同。这里应该指定为x86。对于x86架构，还支持两个别名：i386（32位）和x86_64（64位）。

`INSTALL_HDR_PATH`变量用于指定Linux内核头文件的安装目录，所有的Linux内核头文件都被安装到该目录下的`/include`子目录中。对于交叉构建，该变量的值应该指定为系统根目录下的/usr子目录（`$sysroot/usr`）。

#### 构建binutils

执行下列命令来构建binutils并将其安装到`$sysdir`目录中。

```bash
cd $srcdir
rm -rf build-binutils
mkdir -p build-binutils
cd $srcdir/build-binutils
../binutils-$binutils_ver/configure --prefix=$sysdir \
--target=$target \
--{with-sysroot,with-build-sysroot}=$sysroot
make
make install
```

`--with-sysroot`选项指定目标系统的系统根目录，目标系统的头、库和运行时对象文件将在这里进行搜索。这个目录通常指定为安装目录下与目标三元组同名的目录（`$sysroot`）。

`--with-build-sysroot`选项用于指定在构建过程中gcc使用的系统根目录。这通常和`--with-sysroot`选项指定的目录相同。

#### 构建bootstrap gcc

根据[Installing GCC](https://gcc.gnu.org/install/)中的“[4.Building](https://gcc.gnu.org/install/build.html)”所述，为了构建一个交叉编译器，建议先构建并安装一个本地编译器。然后，使用这个本地编译器构建交叉编译器。

执行下列命令来构建一个bootstrap gcc并将其安装到`$sysdir`目录中。这个bootstrap gcc用于构建glibc和完整版的gcc。

```bash
cd $srcdir 
rm -rf build-gcc
mkdir -p build-gcc
cd $srcdir/build-gcc
../gcc-$gcc_ver/configure --prefix=$sysdir \
--target=$target \
--with-newlib --without-headers \
--enable-languages=c,c++ \
--enable-multilib --with-multilib-list=m32,m64,mx32
make all-gcc
make all-target-libgcc
make install-gcc
make install-target-libgcc
```

当不需要多库目标时，使用`-disable-multilib`选项。当需要多库目标时，使用`--enable-multilib`和`--with-multilib-list=m32,m64,mx32`。对于多库编译器，需要为每个gcc都添加这两个选项。安装之后，安装目录中将会出现5个文件夹：lib、lib32、lib64、libx32和libexec。

因为目标系统（x86_64-linux）是构建系统（x86_64-pc-linux-gnu）的别名，gcc可以使用系统自带的gcc来构建glibc，所以可以省略这个步骤。但是，作为标准的构建，需要构建bootstrap gcc。

#### 构建glibc

x86架构具有三个glibc目标库：[i686 glibc](https://stackoverflow.com/questions/8004241/how-to-compile-glibc-32bit-on-an-x86-64-machine)（`--host=i686-linux`）、[x86_64 glibc](http://www.sourceware.org/glibc/wiki/Testing/Builds)（`--host=x86_64-linux`）、[x32 glibc](http://www.sourceware.org/glibc/wiki/x32)（`--host=x86_64-x32-linux`）。它们分别使用目标编译器（x86_64-linux-gcc/g++）的`-m32`、`-m64`和`-mx32`选项来构建。

执行下列命令来构建3个glibc并将其安装到系统根目录中（`$sysroot`）。(  --libdir=/usr/lib32 --slibdir=/lib32 )

```bash
# 1) 构建i686 glibc 
cd $srcdir
rm -rf build-glibc-i686
mkdir -p build-glibc-i686
cd $srcdir/build-glibc-i686
echo "libdir=/usr/lib32" > configparms
echo "slibdir=/lib32" >> configparms
../glibc-$glibc_ver/configure --prefix=/usr \
--host=i686-linux \
--with-headers=$sysroot/usr/include \
--enable-add-ons \
CC='x86_64-linux-gcc -m32' CXX='x86_64-linux-g++ -m32'
make
make install DESTDIR=$sysroot

# 2) 构建x86_64 glibc
cd $srcdir
rm -rf build-glibc-x86_64
mkdir -p build-glibc-x86_64
cd $srcdir/build-glibc-x86_64
echo "libdir=/usr/lib" > configparms
echo "slibdir=/lib" >> configparms
../glibc-$glibc_ver/configure --prefix=/usr \
--host=x86_64-linux \
--with-headers=$sysroot/usr/include \
--enable-add-ons \
CC='x86_64-linux-gcc -m64' CXX='x86_64-linux-g++ -m64'
make
make install DESTDIR=$sysroot

# 3) 构建x32 glibc
cd $srcdir
rm -rf build-glibc-x32
mkdir -p build-glibc-x32
cd $srcdir/build-glibc-x32
echo "libdir=/usr/libx32" > configparms
echo "slibdir=/libx32" >> configparms
../glibc-$glibc_ver/configure --prefix=/usr \
--host=x86_64-x32-linux \
--with-headers=$sysroot/usr/include \
--enable-add-ons \
CC='x86_64-linux-gcc -mx32' CXX='x86_64-linux-g++ -mx32'
make
make install DESTDIR=$sysroot
```

`--prefix`选项用于指定glibc的相对安装位置，在构建glibc时，必须使用`--prefix=/usr`来配置，并使用`make install DESTDIR=$sysroot`命令将其安装到系统根目录中。这使三个glibc安装到系统根目录中能够共存。

`--with-headers`选项用于指定LinuxKernelHeader文件的真实路径。这应该指定为LinuxKernelHeader安装目录下的/include子目录（`$sysroot/include`）。`--enable-add-ons`选项用于使能构建glibc附加包。这里用于使能glibc-linuxthreads的构建。

根据[glibc wiki](https://sourceware.org/glibc/wiki)中的[ABIList](https://sourceware.org/glibc/wiki/ABIList)所述，glibc默认安装在安装目录中的三个子目录：

- i686 glibc：/lib/ld-linux.so.2（符号链接到/lib32/ld-2.32.so）
- x86_64 glibc：/lib64/ld-linux-x86-64.so.2（符号链接到/lib32/ld-2.32.so）
- x32 glibc：/libx32/ld-linux-x32.so.2

在构建`x86_64-linux-gcc`时，gcc总是在/lib目录搜索64位库，在/lib32目录搜索32位库，这导致gcc找不到对应的glibc库。根据[glibc manual](https://www.gnu.org/software/libc/manual/)中的“[C.1 Configuring and compiling the GNU C Library](https://www.gnu.org/software/libc/manual/html_node/Configuring-and-compiling.html)”章节所述，可以在glibc的**构建目录**下新建一个名为“configparms”脚本，并设置`libdir`和`slibdir`变量来分别重定位glibc基本库和扩展库的安装位置。例如，

```bash
echo "libdir=/usr/libx32" > configparms
echo "slibdir=/libx32" >> configparms
```

#### 构建gcc

执行下列命令来构建一个完整的gcc并将其安装到`$sysdir`目录中。

```bash
cd $srcdir
rm -rf build-gcc
mkdir -p build-gcc
cd $srcdir/build-gcc
../gcc-$gcc_ver/configure --prefix=$sysdir \
--target=$target \
--{with-sysroot,with-build-sysroot}=$sysroot \
--disable-bootstrap \
--enable-languages=c,c++ \
--enable-multilib --with-multilib-list=m32,m64,mx32
make
make install
```

`--with-sysroot`选项用于告诉gcc包含目标操作系统的根文件系统的根目录（sysroot），gcc将在这个目录中搜索目标系统的头（/usr/include）、库和运行时对象文件（gcc默认在系统根目录中的搜索系统头文件）。更具体地说，这就像将`--sysroot=dir`选项添加到构建编译器的默认选项中。该选项会影响用于构建目标库（运行在构建系统上）的编译器的系统根目录，以及使用make install新安装的编译器；它不会影响用于构建gcc自身的编译器。

`--with-build-sysroot`选项用于告诉在构建系统上运行的gcc在构建目标库时使用的系统根目录。这个选项只有当`--with-sysroot`选项被使用时才有用。该选项会影响用于构建目标库（运行在构建系统上）的编译器的系统根目录，以及使用make install新安装的编译器；它不会影响用于构建gcc自身的编译器。

`--disable-bootstrap`选项用于禁用gcc的三阶段引导过程。在构建交叉编译器时，通常是不可能对gcc执行三阶段引导的（3-stage bootstrap）。如果没有禁用这个过程，将在第二阶段引导时出现构建错误。

`--enable-languages`选项指定gcc支持的编程语言。当前支持的值有：`all,default,ada,c,c++,d,fortran,go,jit,lto,objc,obj-c++`。这里仅指定c和c++。

如果出现下列错误，可以降低gcc和/或glibc的版本号。最新的版本存在一些功能依赖问题。

```bash
../../../../gcc-10.1.0/libgcc/config/i386/cpuinfo.c:390:14: error: ‘bit_AVX512BF16’ undeclared (first use in this function); did you mean ‘bit_AVX512F’?
  390 |    if (eax & bit_AVX512BF16)
      |              ^~~~~~~~~~~~~~
      |              bit_AVX512F
../../../../gcc-10.1.0/libgcc/config/i386/cpuinfo.c:390:14: note: each undeclared identifier is reported only once for each function it appears in
make[4]: *** [../../../../gcc-10.1.0/libgcc/shared-object.mk:14: cpuinfo.o] Error 1
```

### 导入系统路径

将上一步构建的x86_64-linux for linux的可执行文件的路径导入到系统路径（PATH）上：

```bash
export PATH=$PATH:$sysdir/bin
```

执行下列命令来检查一下x86_64-linux for linux中的ld和gcc是否可用：

```bash
$target-ld -v
$target-gcc -v
```

这时，我们的构建系统中有了三个gcc，分别是：

1. 为**构建系统**编译可执行程序的本地编译器：x86_64-linux-gnu-gcc；
2. 为**主机系统**编译可执行程序的交叉编译器：x86_64-w64-mingw32-gcc；
3. 为**目标系统**编译可执行程序的交叉编译器：x86_64-linux-gcc。

在构建最终的x86_64-linux for windows交叉编译器时，会调用这三个gcc来产生各自的可执行程序。

### 构建x86_64-linux for windows

当有了gcc，x86_64-w64-mingw32-gcc，以及x86_64-linux-gcc之后，就可以构建x86_64-linux for windows。这个交叉编译器将在Windows中运行为Linux产生可执行文件。因此，它的主机和目标系统类型分别为：

* `--host=x86_64-w64-mingw32`（主机Windows）
* `--target=x86_64-linux`（目标Linux）

x86_64-linux for windows的构建过程与x86_64-linux for linux是相似的。不同之处在于安装目录（`--prefix=$optdir`）和系统类型（`--host=$host`和-`-target=$target`）。


#### 构建sysheader

执行下列命令从Linux Kernel源码中将x86架构的sysheader文件安装到`$optroot/usr`目录中。

```bash
cd $srcdir/linux-$kernel_ver
make headers_install ARCH=x86 INSTALL_HDR_PATH=$optroot/usr
```

#### 构建glibc

执行下列命令来构建i686、x86_64和x32 glibc，并将其安装到`$optroot`目录中。

```bash
# 1) 构建i686 glibc
cd $srcdir
rm -rf build-glibc-i686
mkdir -p build-glibc-i686
cd $srcdir/build-glibc-i686
echo "libdir=/usr/lib32" > configparms
echo "slibdir=/lib32" >> configparms
../glibc-$glibc_ver/configure --prefix=/usr \
--host=i686-linux \
--with-headers=$optroot/usr/include \
--enable-add-ons \
CC='x86_64-linux-gcc -m32' CXX='x86_64-linux-g++ -m32'
make
make install DESTDIR=$optroot

# 2) 构建x86_64 glibc
cd $srcdir
rm -rf build-glibc-x86_64
mkdir -p build-glibc-x86_64
cd $srcdir/build-glibc-x86_64
echo "libdir=/usr/lib" > configparms
echo "slibdir=/lib" >> configparms
../glibc-$glibc_ver/configure --prefix=/usr \
--host=x86_64-linux \
--with-headers=$optroot/usr/include \
--enable-add-ons \
CC='x86_64-linux-gcc -m64' CXX='x86_64-linux-g++ -m64'
make
make install DESTDIR=$optroot

# 3) 构建x32 glibc
cd $srcdir
rm -rf build-glibc-x32
mkdir -p build-glibc-x32
cd $srcdir/build-glibc-x32
echo "libdir=/usr/libx32" > configparms
echo "slibdir=/libx32" >> configparms
../glibc-$glibc_ver/configure --prefix=/usr \
--host=x86_64-x32-linux \
--with-headers=$optroot/usr/include \
--enable-add-ons \
CC='x86_64-linux-gcc -mx32' CXX='x86_64-linux-g++ -mx32'
make
make install DESTDIR=$optroot
```

#### 构建binutils

执行下列命令来构建binutils并将其安装到`$optdir`目录中。

```bash
# 构建binutils
cd $srcdir
rm -rf build-binutils
mkdir -p build-binutils
cd $srcdir/build-binutils
../binutils-$binutils_ver/configure --prefix=$optdir \
--host=$host --target=$target \
--{with-sysroot,with-build-sysroot}=$optroot
make
make install
```

#### 构建gcc

执行下列命令来构建gcc并将其安装到`$optdir`目录中。其配置选项应该与x86_64-linux for linux匹配。

```bash
# 构建gcc
cd $srcdir
rm -rf build-gcc
mkdir -p build-gcc
cd $srcdir/build-gcc
../gcc-$gcc_ver/configure --prefix=$optdir \
--host=$host --target=$target \
--{with-sysroot,with-build-sysroot}=$optroot \
--enable-languages=c,c++ \
--enable-multilib --with-multilib-list=m32,m64,mx32
make
make install
```

### 构建相关工具

#### 构建gdb

如果有特殊需要，也可以构建一个交叉调试器（x86_64-linux-gdb）。这个gdb无法在本机调试目标代码，因为它需要模拟。通常，模拟操作是通过远程连接目标机来实现的。这对于嵌入式开发有一定用处。

执行下列命令来构建expat、gdb和gdbserver并将其安装到`$optdir`目录中。

```bash
# 导入版本号
export expat_ver=2.3.0
export gdb_ver=9.2

# 下载解压
cd $srcdir
wget https://mirrors.tuna.tsinghua.edu.cn/gnu/gdb/gdb-$gdb_ver.tar.gz
wget https://sourceforge.net/projects/expat/files/expat/$expat_ver/expat-$expat_ver.tar.gz
tar -xf gdb-$gdb_ver.tar.gz
tar -xf expat-$expat_ver.tar.gz

# 构建expat
cd $srcdir
rm -rf build-expat
mkdir -p build-expat
cd $srcdir/build-expat
../expat-$expat_ver/configure --prefix=$optdir \
--host=$host \
--with-sysroot=$optroot
make
make install 

# 构建gdb
cd $srcdir
rm -rf build-gdb
mkdir -p build-gdb
cd $srcdir/build-gdb
../gdb-$gdb_ver/configure --prefix=$optdir \
--host=$host --target=$target \
--{with-sysroot,with-build-sysroot}=$optroot \
--with-expat \
--with-system-gdbinit=$optroot/etc/gdbinit 
make
make install

# 构建gdbserver
cd $srcdir
rm -rf build-gdbserver
mkdir -p build-gdbserver
cd $srcdir/build-gdbserver
../gdb-$gdb_ver/gdb/gdbserver/configure --prefix=$optdir \
--host=$target \
--program-prefix=$target-
make
make install
```

根据[gdb wiki](http://sourceware.org/gdb/wiki/)中的[Building GDB and GDBserver for cross debugging](https://sourceware.org/gdb/wiki/BuildingCrossGDBandGDBserver)所述，对于交叉调试，gdb是运行在主机系统上的，而gdbserver是运行在目标系统上的。它们之间通过远程进行连接。所以，

- gdb的`--host`选项应该指定为`x86_64-w64-mingw32`，`--target`选项应该指定为`x86_64-linux` ；
- gdbserver的`--host`选项都应该指定为``x86_64-linux``。`--target`选项对于gdbserver没有意义，因为它不是一个“交叉”工具（它知道如何在自己的系统上调试程序）。

`--{with-sysroot,with-build-sysroot}`用于指定gdb的系统根目录。该选项主要是用于告诉gdb在远程调试时，应该使用在主机系统（而不是目标系统）安装的运行时库来加快调试速度。

`--program-prefix=$target-`选项用于给gdbserver可执行文件添加一个前缀。由于Linux中一般都安装了gdb，所有它附带了一个gdbserver，为了避免同名，需要使用该选项来进行区分两个gdbserver。

根据 [gdb manual](https://www.sourceware.org/gdb/documentation/)中的[Appendix C Installing GDB](https://sourceware.org/gdb/current/onlinedocs/gdb/Installing-GDB.html#Installing-GDB)所述，为了远程调试的功能，需要构建expat库。expat库是一个XML解析库，用于读取由gdb提供的XML文件。如果不可用，那么基于XML文件的某些功能（远程协议内存映射，目标描述，远程共享库列表，Windows系统共享库，跟踪框架信息，以及分支追踪）将在gdb中不可用。

expat仅仅用于解析XML，它不需要关心目标是什么，因此`--host`选项应该指定为`x86_64-w64-mingw32`。当expat构建之后，给gdb指定`--with-expat`选项来使能[expat](https://sourceforge.net/projects/expat/)库。如果libexpat没有安装在与gdb相同的安装目录中，那么还需要使用`--with-libexpat-prefix`选项指定expat的安装位置。

#### 构建make

执行下列命令来构建[make](https://www.gnu.org/software/make/)并将其安装到`$optdir`目录中。

```bash
# 导入版本号
export make_ver=4.2.1

# 下载解压
cd $srcdir
wget https://mirrors.tuna.tsinghua.edu.cn/gnu/make/make-$make_ver.tar.gz
tar -xf make-$make_ver.tar.gz

# 构建make
cd $srcdir
rm -rf build-make
mkdir -p build-make
cd $srcdir/build-make
../make-$make_ver/configure --prefix=$optdir \
--host=$host \
--program-prefix=$target-
make
make install
```

#### 构建nasm

执行下列命令来构建[nasm](https://www.nasm.us/)并将其安装到`$optdir`目录中。

```bash
# 导入版本号
export nasm_ver=2.15.05

# 下载解压
cd $srcdir
wget https://www.nasm.us/pub/nasm/stable/nasm-$nasm_ver.tar.gz
tar -xf nasm-$nasm_ver.tar.gz

# 构建nasm
cd $srcdir
rm -rf build-nasm
mkdir -p build-nasm
cd $srcdir/build-nasm
../nasm-$nasm_ver/configure --prefix=$optdir \
--host=$host \
--program-prefix=$target-
make
make install
```

### 构建常用的开源库

构建一些常用的开源库，以便x86_64-linux-gcc能够在Windows上直接编译产生大多数Linux应用程序。这些库可以安装在`/lib32`或`/usr/lib32`（推荐），并且支持多库编译（m32，m64，mx32）。

http://www.linuxfromscratch.org/~dj/lfs-systemd-multilib/index.html

#### 构建ncurses

执行下列命令来构建[ncurses](https://www.gnu.org/software/ncurses/)并将其安装到`$optroot`目录中。

```bash
# 导入版本号
export ncurses_ver=6.2

# 下载解压
cd $srcdir
wget https://mirrors.tuna.tsinghua.edu.cn/gnu/ncurses/ncurses-$ncurses_ver.tar.gz
tar -xf ncurses-$ncurses_ver.tar.gz

# 构建libncurses
cd $srcdir
rm -rf build-ncurses
mkdir -p build-ncurses
cd $srcdir/build-ncurses
../ncurses-$ncurses_ver/configure --prefix=/usr --libdir=/usr/lib \
--host=x86_64-linux \
--without-ada --without-manpages \
CC='x86_64-linux-gcc -m64' CXX='x86_64-linux-g++ -m64' CFLAGS="-fPIC"
make
make install DESTDIR=$optroot

# 构建lib32ncurses
cd $srcdir
rm -rf build-ncurses32
mkdir -p build-ncurses32
cd $srcdir/build-ncurses32
../ncurses-$ncurses_ver/configure --prefix=/usr --libdir=/usr/lib32 \
--host=i686-linux \
--without-ada --without-manpages \
CC='x86_64-linux-gcc -m32' CXX='x86_64-linux-g++ -m32' CFLAGS="-fPIC" 
make
make install DESTDIR=$optroot

# 构建libx32ncurses
cd $srcdir
rm -rf build-ncursesx32
mkdir -p build-ncursesx32
cd $srcdir/build-ncursesx32
../ncurses-$ncurses_ver/configure --prefix=/usr --libdir=/usr/libx32 \
--host=x86_64-x32-linux \
--without-ada --without-manpages \
CC='x86_64-linux-gcc -mx32' CXX='x86_64-linux-g++ -mx32' CFLAGS="-fPIC" 
make
make install DESTDIR=$optroot


# 将libncurses, lib32ncurses和libx32ncurses安装到$sysroot
cd $srcdir/build-ncurses
make install DESTDIR=$sysroot

cd $srcdir/build-ncurses32
make install DESTDIR=$sysroot

cd $srcdir/build-ncursesx32
make install DESTDIR=$sysroot
```

`--without-ada`选项用于禁止构建Ada95绑定。`--without-manpages`选项用于禁止安装手册页。在构建ncurses时，需要指定`CFLAGS=-fPIC`标志来编译共享库。`--without-progs`选择用于禁止构建可执行程序（例如，`tic`）。这些程序只能在Linux中运行，对于我们来说没有用。`--enable-widec`选项用于使能宽字符支持，这将会在库名后面添加一个字符“w”（例如，libncursesw.so）。

创建一个测试ncurses的C程序（n.c）：

```c
#include <string.h>
#include <ncurses.h>
int main(int argc,char* argv[]){
    initscr();
    raw();
    noecho();
    curs_set(0);
 
    char* c = "Hello, World!";
 
    mvprintw(LINES/2,(COLS-strlen(c))/2,c);
    refresh();
 
    getch();
    endwin();
 
    return 0;
}
```

分别执行下列命令编译，是否能够成功编译：

```
xgcc -m32 n.c -o n.out -lncursesw
xgcc -m64 n.c -o n.out -lncursesw
xgcc -mx32 n.c -o n.out -lncursesw
```

#### 构建readline

执行下列命令来构建[readline](https://tiswww.case.edu/php/chet/readline/rltop.html)并将其安装到`$optroot`目录中。

```bash
# 导入版本号
export readline_ver=8.0

# 下载解压
cd $srcdir
wget https://mirrors.tuna.tsinghua.edu.cn/gnu/readline/readline-$readline_ver.tar.gz
tar -xf readline-$readline_ver.tar.gz

# 构建libreadline
cd $srcdir
rm -rf build-readline
mkdir -p build-readline
cd $srcdir/build-readline
../readline-$readline_ver/configure --prefix=/usr --libdir=/usr/lib \
--host=x86_64-linux \
--with-curses \
CC='x86_64-linux-gcc -m64' CXX='x86_64-linux-g++ -m64'
make
make install DESTDIR=$optroot

# 构建lib32readline
cd $srcdir
rm -rf build-readline32
mkdir -p build-readline32
cd $srcdir/build-readline32
../readline-$readline_ver/configure --prefix=/usr --libdir=/usr/lib32 \
--host=i686-linux \
--with-curses \
CC='x86_64-linux-gcc -m32' CXX='x86_64-linux-g++ -m32'
make
make install DESTDIR=$optroot

# 构建libx32readline
cd $srcdir
rm -rf build-readlinex32
mkdir -p build-readlinex32
cd $srcdir/build-readlinex32
../readline-$readline_ver/configure --prefix=/usr --libdir=/usr/libx32 \
--host=x86_64-x32-linux \
--with-curses \
CC='x86_64-linux-gcc -mx32' CXX='x86_64-linux-g++ -mx32'
make
make install DESTDIR=$optroot


# 将libreadline, lib32readline和libx32readline安装到$sysroot
cd $srcdir/build-readline
make install DESTDIR=$sysroot

cd $srcdir/build-readline32
make install DESTDIR=$sysroot

cd $srcdir/build-readlinex32
make install DESTDIR=$sysroot
```

`--with-curses`选项用于告诉configure脚本可以在curses库中找到termcap库函数（tgetent等），而不是在单独的termcap库中。

创建一个c程序：

```c
#include <stdio.h>
#include <readline/readline.h>
#include <readline/history.h>

int main() {
    // 设置readline为按tab键自动完成路径
    rl_bind_key('\t', rl_complete);

    while (1) {
        // 显示提示和读取输入
        char* input = readline("prompt> ");
       
        // 检查EOF
        if (!input)
            break;
       
        // 添加输入到readline历史
        add_history(input);
        
        // Do stuff...
        
        // 由readline分配的空闲缓冲区
        free(input);
    }
    return 0;
}
```

执行下列命令编译：

```bash
x86_64-linux-gcc -m32 r.c -o r.out -lreadline -lncursesw --static
x86_64-linux-gcc -m64 r.c -o r.out -lreadline -lncursesw --static
x86_64-linux-gcc -mx32 r.c -o r.out -lreadline -lncursesw --static
```

#### 构建libiconv

执行下列命令来构建[libiconv](https://www.gnu.org/software/libiconv/)并将其安装到`$optroot`目录中。

```bash
# 导入版本号
export iconv_ver=1.16

# 下载解压
cd $srcdir
wget https://mirrors.tuna.tsinghua.edu.cn/gnu/libiconv/libiconv-$iconv_ver.tar.gz
tar -xf libiconv-$iconv_ver.tar.gz

# 构建libiconv
cd $srcdir
rm -rf build-libiconv
mkdir -p build-libiconv
cd $srcdir/build-libiconv
../libiconv-$iconv_ver/configure --prefix=/usr --libdir=/usr/lib \
--host=x86_64-linux \
CC='x86_64-linux-gcc -m64' CXX='x86_64-linux-g++ -m64'
make
make install DESTDIR=$optroot

# 构建lib32iconv
cd $srcdir
rm -rf build-libiconv32
mkdir -p build-libiconv32
cd $srcdir/build-libiconv32
../libiconv-$iconv_ver/configure --prefix=/usr --libdir=/usr/lib32 \
--host=i686-linux \
CC='x86_64-linux-gcc -m32' CXX='x86_64-linux-g++ -m32'
make
make install DESTDIR=$optroot

# 构建libx32iconv
cd $srcdir
rm -rf build-libiconvx32
mkdir -p build-libiconvx32
cd $srcdir/build-libiconvx32
../libiconv-$iconv_ver/configure --prefix=/usr --libdir=/usr/libx32 \
--host=x86_64-x32-linux \
CC='x86_64-linux-gcc -mx32' CXX='x86_64-linux-g++ -mx32'
make
make install DESTDIR=$optroot
```

#### 构建sqlite

执行下列命令来构建[sqlite](https://www.sqlite.org/)并将其安装到`$optroot`目录中。

```bash
# 导入版本号
export sqlite_ver=3350400

# 下载解压
cd $srcdir
wget https://www.sqlite.org/2021/sqlite-autoconf-$sqlite_ver.tar.gz
tar -xf sqlite-autoconf-$sqlite_ver.tar.gz

# 构建libsqlite
cd $srcdir
rm -rf build-sqlite
mkdir -p build-sqlite
cd $srcdir/build-sqlite
../sqlite-autoconf-$sqlite_ver/configure --prefix=/usr --libdir=/usr/lib \
--host=x86_64-linux \
CC='x86_64-linux-gcc -m64' CXX='x86_64-linux-g++ -m64'
make
make install DESTDIR=$optroot

# 构建lib32sqlite
cd $srcdir
rm -rf build-sqlite32
mkdir -p build-sqlite32
cd $srcdir/build-sqlite32
../sqlite-autoconf-$sqlite_ver/configure --prefix=/usr --libdir=/usr/lib32 \
--host=i686-linux \
CC='x86_64-linux-gcc -m32' CXX='x86_64-linux-g++ -m32'
make
make install DESTDIR=$optroot

# 构建libx32sqlite
cd $srcdir
rm -rf build-sqlitex32
mkdir -p build-sqlitex32
cd $srcdir/build-sqlitex32
../sqlite-autoconf-$sqlite_ver/configure --prefix=/usr --libdir=/usr/libx32 \
--host=x86_64-x32-linux \
CC='x86_64-linux-gcc -mx32' CXX='x86_64-linux-g++ -mx32'
make
make install DESTDIR=$optroot
```



```
apt-cache search sqlite3
apt-get install sqlite sqlite3 libsqlite3-dev
gcc s.c -o s.out -lsqlite3 -m32
gcc s.c -o s.out -lsqlite3 -m64
gcc s.c -o s.out -lsqlite3 -mx32

x86_64-linux-gcc s.c -o s.out -lsqlite3 -m32
x86_64-linux-gcc s.c -o s.out -lsqlite3 -m64
x86_64-linux-gcc s.c -o s.out -lsqlite3 -mx32
```

#### 构建openssl

执行下列命令来构建[openssl](https://www.openssl.org/)并将其安装到`$optroot`目录中。

```bash
# 导入版本号
export openssl_ver=1.1.1k

# 下载解压
cd $srcdir
wget https://www.openssl.org/source/openssl-$openssl_ver.tar.gz
tar -xf openssl-$openssl_ver.tar.gz

# 构建libopenssl
cd $srcdir
rm -rf build-openssl
mkdir -p build-openssl
cd $srcdir/build-openssl
../openssl-$openssl_ver/Configure --prefix=/usr --libdir=/usr/lib \
linux-x86_64 \
CC='x86_64-linux-gcc -m64' CXX='x86_64-linux-g++ -m64'
make
make install_sw DESTDIR=$optroot


# 构建lib32openssl
cd $srcdir
rm -rf build-openssl32
mkdir -p build-openssl32
cd $srcdir/build-openssl32
../openssl-$openssl_ver/Configure --prefix=/usr --libdir=/usr/lib32 \
linux-x86 \
CC='x86_64-linux-gcc -m32' CXX='x86_64-linux-g++ -m32'
make
make install_sw DESTDIR=$optroot

# 构建libx32sqlite
cd $srcdir
rm -rf build-opensslx32
mkdir -p build-opensslx32
cd $srcdir/build-opensslx32
../openssl-$openssl_ver/Configure --prefix=/usr --libdir=/usr/libx32 \
linux-x32 \
CC='x86_64-linux-gcc -mx32' CXX='x86_64-linux-g++ -mx32'
make
make install_sw DESTDIR=$optroot
```

#### 构建curl

```bash
# 导入版本号
export curl_ver=7.76.0

# 下载解压
cd $srcdir
wget https://curl.se/download/curl-$curl_ver.tar.gz
tar -xf curl-$curl_ver.tar.gz

# 构建libcurl
cd $srcdir
rm -rf build-curl
mkdir -p build-curl
cd $srcdir/build-curl
../curl-$curl_ver/configure --prefix=/usr --libdir=/usr/lib \
--host=x86_64-linux \
CC='x86_64-linux-gcc -m64' CXX='x86_64-linux-g++ -m64'
make
make install DESTDIR=$optroot

# 构建lib32curl
cd $srcdir
rm -rf build-curl32
mkdir -p build-curl32
cd $srcdir/build-curl32
../curl-$curl_ver/configure --prefix=/usr --libdir=/usr/lib32 \
--host=i686-linux \
CC='x86_64-linux-gcc -m32' CXX='x86_64-linux-g++ -m32'
make
make install DESTDIR=$optroot

# 构建libx32curl
cd $srcdir
rm -rf build-curlx32
mkdir -p build-curlx32
cd $srcdir/build-curlx32
../curl-$curl_ver/configure --prefix=/usr --libdir=/usr/libx32 \
--host=x86_64-x32-linux \
CC='x86_64-linux-gcc -mx32' CXX='x86_64-linux-g++ -mx32'
make
make install DESTDIR=$optroot
```

#### 构建zlib

执行下列命令来构建[zlib](http://www.zlib.net/)并将其安装到`$optroot`目录中。参考[zlib构建](http://www.linuxfromscratch.org/~dj/lfs-systemd-multilib/chapter05/zlib.html)。

```bash
# 导入版本号
export zlib_ver=1.2.11

# 下载解压
cd $srcdir
wget http://www.zlib.net/zlib-$zlib_ver.tar.gz
tar -xf zlib-$zlib_ver.tar.gz

# 构建libz
cd $srcdir
rm -rf build-zlib
mkdir -p build-zlib
cd $srcdir/build-zlib
CC='x86_64-linux-gcc -m64' \
../zlib-$zlib_ver/configure --prefix=/usr --libdir=/usr/lib
make
make install DESTDIR=$optroot

# 构建lib32z
cd $srcdir
rm -rf build-zlib32
mkdir -p build-zlib32
cd $srcdir/build-zlib32
CC='x86_64-linux-gcc -m32' \
../zlib-$zlib_ver/configure --prefix=/usr --libdir=/usr/lib32
make
make install DESTDIR=$optroot

# 构建libx32z
cd $srcdir
rm -rf build-zlibx32
mkdir -p build-zlibx32
cd $srcdir/build-zlibx32
CC='x86_64-linux-gcc -mx32' \
../zlib-$zlib_ver/configure --prefix=/usr --libdir=/usr/libx32
make
make install DESTDIR=$optroot
```

#### 构建gdbm

执行下列命令来构建[gdbm](https://www.gnu.org.ua/software/gdbm/)并将其安装到`$optroot`目录中。

```bash
# 导入版本号
export gdbm_ver=1.19

# 下载解压
cd $srcdir
wget https://mirrors.tuna.tsinghua.edu.cn/gnu/gdbm/gdbm-$gdbm_ver.tar.gz
tar -xf gdbm-$gdbm_ver.tar.gz


# 构建libgdbm
cd $srcdir
rm -rf build-gdbm
mkdir -p build-gdbm
cd $srcdir/build-gdbm
../gdbm-$gdbm_ver/configure --prefix=/usr --libdir=/usr/lib \
--host=x86_64-linux \
--with-readline \
CC='x86_64-linux-gcc -m64' CXX='x86_64-linux-g++ -m64'
make
make install DESTDIR=$optroot


# 构建lib32gdbm
cd $srcdir
rm -rf build-gdbm32
mkdir -p build-gdbm32
cd $srcdir/build-gdbm32
../gdbm-$gdbm_ver/configure --prefix=/usr --libdir=/usr/lib \
--host=i686-linux \
--with-readline \
CC='x86_64-linux-gcc -m32' CXX='x86_64-linux-g++ -m32'
make
make install DESTDIR=$optroot


# 构建libx32gdbm
cd $srcdir
rm -rf build-gdbmx32
mkdir -p build-gdbmx32
cd $srcdir/build-gdbmx32
../gdbm-$gdbm_ver/configure --prefix=/usr --libdir=/usr/lib \
--host=x86_64-x32-linux \
--with-readline \
CC='x86_64-linux-gcc -mx32' CXX='x86_64-linux-g++ -mx32'
make
make install DESTDIR=$optroot
```

### 压缩打包

执行下列命令将安装目录（`$optdir`）打包，以方便复制到Windows中。

```bash
# 压缩
cd $optdir
rm -rf $target-tools.zip
zip -r $target-tools.zip *

# 解压
rm -rf $optdir
unzip -d /opt/cross /root/src/x86_64-linux-tools.zip

# 删除
rm -rf $optdir
rm -rf $sysdir
rm -rf $srcdir
```

zip命令中的`*`表示压缩当前目录中的所有文件和目录。`-r`选项表示递归到子目录。

所有的可执行文件的文件名都以`x86_64-linux-`为前缀。例如，gcc的文件名为`x86_64-linux-gcc`。

## 使用

将**x86_64-linux-tools.zip**复制到Windows中，将其解压到某个目录（例如**D:\osdev**）就可以使用了。在解压时，会出现有重复文件的提示。这是因为Windows的文件系统不区分文件名大小写而造成的。在解压时，可以选择重命名或覆盖，都不会影响工具的使用。

下面编写一个程序来检查一下x86_64-linux for windows编译器是否可用。在**D:\os**目录下，新建一个文本文档，将其命名为**main.c**。然后，添加下列代码：

```c
#include "stdio.h"
int main(){
        printf("Hello World! \n");
}
```

这是所有c语言入门教程都会用到的Hello World程序。

然后，打开cmd，输入下列命令编译这个C程序：

```bash
set path=%path%;D:\osdev\x86_64-linux-tools\bin;

cd /D D:\os
x86_64-linux-gcc -m32 main.c

:: x86_64-linux-gcc -m64 main.c
:: x86_64-linux-gcc -mx32 main.c
```

首先，将编译器的可执行文件路径（D:\osdev\x86_64-linux-tools\bin）添加到系统路径变量（path）上。然后，进入D:\os目录，并使用`x86_64-linux-gcc`命令来编译main.c。

这将编译产生一个名为`a.out`的可执行文件。但是，这个可执行文件不能在Windows（主机）中运行，必须将它复制到Linux（目标）中运行。

### 在git中使用x86_64-linux for windows

当使用git for windows的终端环境时，也需要一些配置。在`C:\Program Files\Git\etc\profile.d`目录中新建一个脚本文件（例如，myinit.sh），并输入下列代码以UTF-8编码保存：

```bash
#!/usr/bin/bash

# 1) 设置提示符: user@host /path/to (master) $
MSYS2_PS1='\n\[\e[1;32m\]\u\[\e[36m\]@\[\e[34m\]\h \[\e[33m\]\w\[\033[36m\]`__git_ps1` \[\e[31m\]\$ \[\e[00m\]'

# \n	换行符
# \u	当前用户名
# \h	主机名，直到第一个点字符（.）
# \w	当前工作目录, $HOME被缩写为波浪号(~)
# `__git_ps1`	git的分支名称. 主分支的名称为: (master)
# \[\e[A;B;Cm\]	指定之后的字符颜色. 格式是: \[\e[A;B;Cm\]. 其中, A表示前景色; B表示背景色; C表示颜色属性
 	
#  A		B		C
# 30=黑色	40=黑色	1=高亮显示
# 31=红色	41=红色	
# 32=绿色	42=绿色	
# 33=黄色	43=黄色	4=显示下划线
# 34=蓝色	44=蓝色	5=闪烁显示
# 35=紫色	45=紫色	
# 36=青色	46=青色	7=反白显示
# 37=白色	47=白色	8=颜色不可见

# 2) 添加需要的命令路径到系统路径(请使用自己的实际路径)
export PATH="$PATH:/d/apps/mingw64/bin"              # MinGW-w64
export PATH="$PATH:/d/apps/x86_64-linux-tools/bin"   # x86_64-linux
export PATH="$PATH:/d/apps/nasm"                     # nasm

# 3) 设置命令别名
# x86_64-linux-tools alias
alias xaddr2line="x86_64-linux-addr2line"
alias xar="x86_64-linux-ar"
alias xas="x86_64-linux-as"
alias xc++="x86_64-linux-c++"
alias xc++filt="x86_64-linux-c++filt"
alias xcpp="x86_64-linux-cpp"
alias xelfedit="x86_64-linux-elfedit"
alias xg++="x86_64-linux-g++"
alias xgcc="x86_64-linux-gcc"                        # gcc
alias xgcov="x86_64-linux-gcov"
alias xgcov-dump="x86_64-linux-gcov-dump"
alias xgcov-tool="x86_64-linux-gcov-tool"
alias xgdb="x86_64-linux-gdb"                        # gdb
alias xgprof="x86_64-linux-gprof"
alias xld.bfd="x86_64-linux-ld.bfd"
alias xld="x86_64-linux-ld"
alias xmake="x86_64-linux-make"                      # make
alias xnm="x86_64-linux-nm"
alias xobjcopy="x86_64-linux-objcopy"
alias xobjdump="x86_64-linux-objdump"
alias xranlib="x86_64-linux-ranlib"
alias xreadelf="x86_64-linux-readelf"
alias xsize="x86_64-linux-size"
alias xstrings="x86_64-linux-strings"
alias xstrip="x86_64-linux-strip"
# MinGW-w64 alias
alias gsc="g++ -static -static-libgcc -static-libstdc++"   # gsc <-- gcc static compilation

```

设置好之后，如果在某个git仓库中，右击打开Git Bash， 命令提示符将显示成如下格式：


```bash
niki@niki-PC /e/gitdir (master) $
```

如果需要汉化Git GUI，可以从https://github.com/stayor/git-gui-zh下载zh_cn.msg文件，然后，将zh_cn.msg复制到C:\Program Files\Git\mingw64\share\git-gui\lib\msgs目录中。

### 在cmd中使用x86_64-linux for windows

#### 让cmd自动执行bat脚本

为了让cmd在启动时自动运行某个bat脚本，只需要将bat脚本的文件路径添加到注册表中的AutoRun键即可。AutoRun的路径有两个：

```cmd
HKEY_LOCAL_MACHINE\Software\Microsoft\Command Processor (系统)
    和/或
HKEY_CURRENT_USER\Software\Microsoft\Command Processor  (用户)
```

这里，如果两个都有值，那么先执行HKEY_LOCAL_MACHINE中的AutoRun，然后执行HKEY_CURRENT_USER中的AutoRun。

首先，在个人文件夹（%USERPROFILE%）新建一个文本文档，重命名为`autoexec.bat`，并输入下列代码：

```cmd
::关闭回显
@echo off
echo Executing [%USERPROFILE%\autoexec.bat] ...
::prompt $E[1;32m$P$S$G$S$E[1;37m

:: 添加命令路径
set path=%path%;d:\apps\nasm
set path=%path%;d:\apps\mingw64\bin
set path=%path%;d:\apps\x86_64-linux-tools\bin
set path=%path%;D:\apps\ansicon\x64
set path=%path%;D:\apps\ColorTool

:: 设置命令别名
:: x86_64-linux-tools alias
doskey xaddr2line=x86_64-linux-addr2line $*
doskey xar=x86_64-linux-ar $*
doskey xas=x86_64-linux-as $*
doskey xc++=x86_64-linux-c++ $*
doskey xc++filt=x86_64-linux-c++filt $*
doskey xcpp=x86_64-linux-cpp $*
doskey xelfedit=x86_64-linux-elfedit $*
doskey xg++=x86_64-linux-g++ $*
doskey xgcc=x86_64-linux-gcc $*
doskey xgcov=x86_64-linux-gcov $*
doskey xgcov-dump=x86_64-linux-gcov-dump $*
doskey xgcov-tool=x86_64-linux-gcov-tool $*
doskey xgdb=x86_64-linux-gdb $*
doskey xgprof=x86_64-linux-gprof $*
doskey xld.bfd=x86_64-linux-ld.bfd $*
doskey xld=x86_64-linux-ld $*
doskey xmake=x86_64-linux-make $*
doskey xnm=x86_64-linux-nm $*
doskey xobjcopy=x86_64-linux-objcopy $*
doskey xobjdump=x86_64-linux-objdump $*
doskey xranlib=x86_64-linux-ranlib $*
doskey xreadelf=x86_64-linux-readelf $*
doskey xsize=x86_64-linux-size $*
doskey xstrings=x86_64-linux-strings $*
doskey xstrip=x86_64-linux-strip $*

:: MinGW-w64 alias
doskey gsc=g++ -static-libgcc -static-libstdc++  $*

:: cmd alias
doskey ls=dir /b $*
doskey ll=dir /od/p/q/tw $*
```

`doskey`命令用于设置别名（相当于Linux中的alias命令），使等号左边的是其右边的别名，`$*`表示这个命令可能还有其他选项。`set PATH`命令用于将命令的可执行文件路径临时添加到系统路径上。

打开注册表（regedit），在`[HKEY_CURRENT_USER\Software\Microsoft\Command Processor]`项目中新建一个字符串值：`AutoRun` = `%USERPROFILE%\autoexec.bat`。

```cmd
%USERPROFILE%\autoexec.bat
```

这样启动cmd时，会。这时，重新启动cmd之后，就会自动执行autoexec.bat文件：

```cmd
xgcc -v
```

#### 让cmd支持ANSI escape sequences

为了让cmd支持ANSI escape sequences功能，需要安装Ansicon。（Windows 10默认支持ANSI escape sequences。）

首先，从 Ansicon官网（https://github.com/adoxa/ansicon）下载[ansi189-bin.zip](https://github.com/adoxa/ansicon/releases/download/v1.89/ansi189-bin.zip)并解压。然后，打开cmd，进入Ansicon解压目录中的/x64子目录（D:\apps\ansicon\x64），输入下列命令来安装Ansicon：

```cmd
cd D:\apps\ansicon\x64
ansicon -i        # 安装, 仅用户
ansicon -u        # 卸载, 仅用户

ansicon -I        # 安装, 用户和系统(需要以管理员身份运行cmd)
ansicon -U        # 卸载, 用户和系统(需要以管理员身份运行cmd)
```

安装成功之后，就可以cmd中使用ANSI escape sequences了。例如：

```cmd
echo ^[[31mHello                 # 输出红色的Hello
prompt ^[[1;32m$P$S$G$S^[[1;37m  # 将提示符文本颜色更改成绿色(查看帮助prompt /?)
```

上述命令中的`^[`是一个通过`Ctrl+[`或`Alt+2+7`（27是`ESC`的ASCII码）得到的字符。`^[`表示是`ESC`控制字符，用于指示一个ANSI escape sequences的开始。

`prompt`命令支持使用`$E`来表示`ESC`控制字符，可以将`^[`替换成`$E`。因此，上述的`prompt`命令等效于

```cmd
prompt $E[1;32m$P$S$G$S$E[1;37m  # 将提示符文本颜色更改成绿色(查看帮助prompt /?)
```

在C/C++语言中，使用`\E` 或`033`（八进制）来表示`ESC`控制字符。例如：

```c
printf("\E[31mHello, Word!");
printf("\033[31mHello, Word!");
```

Ansicon支持的ANSI escape sequences，请参考Ansicon解压目录中的readme.txt和sequences.txt，或[ANSI escape code的维基百科词条](https://en.wikipedia.org/wiki/ANSI_escape_code)（https://en.wikipedia.org/wiki/ANSI_escape_code）。

#### 让cmd主题颜色更好看

首先，从 ColorTool官网（https://github.com/microsoft/terminal/tree/main/src/tools/ColorTool）下载[ColorTool.zip](https://github.com/microsoft/terminal/releases/download/1904.29002/ColorTool.zip)并解压。然后，打开cmd，进入ColorTool解压目录（D:\apps\ColorTool），输入下列命令更改 cmd 的主题颜色：

```
cd D:\apps\ColorTool
colortool -b OneHalfDark.itermcolors   (推荐主题)
colortool -b cmd-legacy.ini            (默认主题)
```

最后，在 cmd 窗口的标题栏上右击→属性→确定，将颜色更改保存下来。这时，cmd 窗口看起来柔和自然一些，没有那么刺眼。

 ColorTool自带的主题很少，只有OneHalfDark.itermcolors主题看着舒服。如果需要，可以在iTerm2-Color-Schemes官网（https://github.com/mbadolato/iTerm2-Color-Schemes）中，将/schemes目录中的.itermcolors文件，复制到ColorTool解压目录中的/schemes子目录中，然后，执行`colortool -b`命令更改主题颜色，例如：

```
colortool -b breeze.itermcolors
colortool -b The_Hulk.itermcolors
```

#### 让cmd支持Bash命令行编辑

从[Clink](https://mridgers.github.io/clink/)官网下载[clink_0.4.9_setup.exe](https://github.com/mridgers/clink/releases/download/0.4.9/clink_0.4.9_setup.exe)并安装。一路默认即可。

Clink将cmd与 GNU Readline 库强大的命令行编辑功能相结合，提供了丰富的新快捷键、上下文敏感完成、历史记录永久保存和行编辑功能。

- 与Bash相同的行编辑（来自 GNU Readline库） 。
- 会话之间的历史记录。
- 上下文敏感完成：可执行文件和别名、目录命令、环境变量、第三方工具：Git，Mercurial，SVN，Go和P4。
- 新的键盘快捷键：
  - 粘贴（Ctrl-V）、撤消（Ctrl-Z） 、自动`cd..`（Ctrl-PgUp）。
  - 增量历史记录搜索（Ctrl-R/Ctrl-S）。
  - 强大的完成（TAB）。
  - 环境变量扩展（Ctrl-Alt-E）。
  - 按Alt-H查看更多。
- 与Lua一起完成脚本化。
- 彩色和可脚本化的提示。
- 自动回答”Terminate batch job?“。

更多帮助信息，请查看Clink安装目录中的clink.html（C:\Program Files (x86)\clink\0.4.9\clink.html）。

## 调试

本节是对“在Ubuntu中构建x86_64-linux for windows”笔记的补充说明，用于解释如何使用x86_64-linux for windows工具链中的`x86_64-linux-gdb/gdbserver`命令来执行远程调试。

对于x86_64-linux for windows来说，这种远程调试方式没有一点用处。因为直接使用虚拟机中gdb来进行调试更简单更方便。或许，对其他工具链（例如，arm-linux）来说，还是有用的。这里仅仅是为了解释远程调试的使用方法。

### 配置局域网

执行远程调试之前，需要将主机和虚拟机配置到同一局域网中。这里的配置环境仍然是：

- 主机：Windows
- 虚拟机：Ubuntu

**第一步，查看主机的网络连接信息**

在桌面点击，选择“网络→网络和共享中心→更改适配器设置”，找到显示已连接的网络右击，选择“状态→详细信息”，得到如下信息：

- **IPv4地址：192.168.0.101**
- **子网掩码：255.255.255.0**
- **默认网关：192.168.0.1**
- **DNS：192.168.0.1**

**第二步，将虚拟机的网络连接改成桥接模式**

在VMware中，点击菜单“虚拟机→设置→网络适配器”，将“网络连接”改成“桥接模式”。

**第三步，设置虚拟机的静态ip地址**

打开VMware中的Ubuntu，在桌面右击，选择“设置（Settings）→网络（Network）→有线（Wired）”，在打开对话框中，根据Windows的网络连接信息配置如下：

- **IPv4 Method**：manual（将IPv4方式设置成手动）
  - **Address**：**192.168.0.106**（将地址设置成与主机相同的网段上）。
  - **Netmask**：255.255.255.0（子网掩码与主机一样）。
  - **Gateway**：192.168.0.1（网关与主机一样）。
- **DNS**：192.168.0.1（DNS与主机一样）。

为了使设置更改生效，需要对该网络执行一次关闭和打开操作。

#### 使用netplan配置Ubuntu网络

在Linux中，通过命令行来配置网络显得高级一点。配置方法如下：

```bash
# (1) 打开网卡的配置文件(文件格式为*.yaml)
cd /etc/netplan
ls
sudo gedit 01-network-manager-all.yaml

# (2) 将下列内容添加到配置文件中
network:
    ethernets:
        ens33:
            dhcp4: false
            addresses: [192.168.0.106/24]
            gateway4: 192.168.0.1
            nameservers:
                addresses: [192.168.0.1]
    version: 2
    renderer: NetworkManager

# (3) 应用配置,使网络配置生效
sudo netplan apply
```

### 检查网络配置是否有效（可选）

当局域网配置完成之后，可以使用`ping`命令检查一下主机和虚拟机能不能相互访问：

```bash
# 在主机windows中, ping虚拟机的ip
ping 192.168.0.106

# 在虚拟机Ubuntu中, ping主机的ip
ping 192.168.0.101
```

### 设置共享文件夹

为了简化调试流程，设置一个共享文件夹用于在主机和虚拟机之间共享文件是非常有必要的。

在VMware中，点击菜单“虚拟机→设置“，在打开虚拟机设置对话框，点击”选项→共享文件夹”，添加一个共享文件夹（例如，E:\os）。该文件夹将以相同名挂载到/mnt/hgfs目录下。

这样，编译调试都将在这个文件夹中执行，无需在主机和虚拟机之间通过复制粘贴来传输文件了。

### 启动远程监听

在启动远程监听之前，需要将安装目录中的`x86_64-linux-gdbserver`可执行文件复制到共享文件夹中（/mnt/hgfs/os）。

然后，在虚拟机中，执行下列命令启动gdbserver进行远程监听。它监听的是主机的ip地址（192.168.0.101），端口号可以是任意值，这里是6666。

```bash
export PATH=/mnt/hgfs/os:$PATH
x86_64-linux-gdbserver 192.168.0.101:6666 a.out
```

### 远程调试

启动远程监听之后，就可以在主机中进行编译调试操作了。

在主机中，使用x86_64-linux-gcc编译一个main.c程序并启动gdb：

```bat
cd /D E:\os
xgcc main.c
xgdb a.out
```

在gdb中，使用下列类似的命令连接虚拟机。这里，连接的是虚拟机的ip地址（192.168.0.106），端口号是x86_64-linux-gdbserver监听的端口。

```bash
target remote 192.168.0.106:6666
```

如果连接成功，虚拟机中的x86_64-linux-gdbserver 将会显示来自主机的远程调试：

```bash
Remote debugging from host 192.168.0.101
```

接下来就可以开始远程调试了。

下列是一些常用的远程调试命令：

```bash
b main       # 设置断点
info b       # 显示断点
c            # 运行至断点处
l            # 列出源代码(要求源代码文件也在可执行文件的目录中)
s            # 单步执行
n            # 单步执行
print a      # 打印变量的值
q            # 退出gdb
```

调试的操作在主机中，而程序执行的结果任然显示在虚拟机中。[![pcsys](https://img.shields.io/github/stars/kitian616/jekyll-TeXt-theme.svg?label=Stars&style=social)](https://github.com/pcsys)

