编译安装llvm和kint
==================

因为kint是2013年的工具，很久没更新，直接安装最新版本的llvm会导致兼容性问题，所以需要手动编译安装llvm-3.2。

1. 安装llvm-3.2
===============

1 下载llvm所需文件
------------------

从llvm官网(http://releases.llvm.org/download.html\#3.2)下载三个文件：LLVM source
code, Clang source code, Compiler RT source code。

2 构造目录结构
--------------

解压llvm-3.2.src.tar.gz，重命名为llvm。

解压clang-3.2.src.tar.gz，重命名为clang，并将clang移到llvm/tools中。

解压compiler-rt-3.2.src.tar.gz，重命名为compiler-rt，并compiler-rt移动到llvm/projects

3 编译
------

需要安装g++：

\$ sudo apt-get install g++

创建build文件夹，进行编译：

\$ cd llvm

\$ mkdir build

\$ cd build

\$ ../configure --enable-optimized --enable-targets=host --enable-bindings=none
--enable-shared --enable-debug-symbols

使用make进行编译，可以选择多核编译。编译需要较长时间。

\$ make -j4

4 安装
------

\$ sudo make install

2. 安装kint
===========

1 下载kint源码
--------------

从github上下载源码，并解压到kint。

Kint地址：<https://github.com/CRYPTOlab/kint>.

2 编译
------

安装编译需要的工具：

\$ sudo apt-get install autoconf automake libtool

执行以下命令进行编译：

\$ cd kint

\$ autoreconf -fvi

\$ mkdir build

\$ cd build

\$ ../configure

\$ make

编译好的文件在build/bin目录下。
