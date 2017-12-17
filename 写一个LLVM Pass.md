**1 介绍：什么是Pass**
======================

LLVM
的Pass框架是LLVM系统的一个很重要的部分。每个Pass都是做优化或者转变的工作，LLVM的优化和转换工作就是由很多个Pass来一起完成的。

Pass就是LLVM系统转化和优化的工作的一个节点，每个节点做一些工作，这些工作加起来就构成了LLVM整个系统的优化和转化。Pass架构这么做的话，可重用性非常好，可以选择已有的一些Pass，定制优化和转化效果。每个Pass都可以独立存在，所以新建Pass并不用考虑LLVM之前的优化和转化是怎么做的。

所有的Pass都是LLVM提供的Pass的子类，通过重写其虚函数实现自定义Pass。根据需求，还需要继承ModulePass、CallGraphSCCPass、FunctionPass、LoopPass、RegionPass或者BasicBlockPass
classes这些类。

**2 快速入门：编写hello world**
===============================

写一个“Hello
world”的Pass，不修改程序，仅打印非扩展函数的名称。源码在lib/Transforms/Hello中。

2.1 构建build环境
-----------------

安装配置完成LLVM，然后在LLVM源代码中创建一个目录。例如，lib/Transforms/Hello。最后，必须写一个脚本编译新的pass。可以复制以下代码到lib/Transforms/Hello/CMakeList.txt:

——————————————————————

add\_llvm\_loadable\_module( LLVMHello

Hello.cpp

PLUGIN\_TOOL

opt

)

——————————————————————

然后，复制下句话到lib/Transforms/CMakeLists.txt：

——————————————————————

add\_subdirectory(Hello)

——————————————————————

（注意，已经有一个名为Hello“Hello”示例的目录，您可以使用它，在这种情况下，您不需要修改任何
CMakeLists.txt文件，或者如果要从头开始创建所有内容，请使用其他名称。）

此构建脚本将Hello.cpp编译并链接到\$(LEVEL)/lib/LLVMHello.so，可由opt工具通过其-load
选项动态加载。

现在已经建立了build脚本，只需要编写pass本身代码。

\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

以下是来自3.2的doc

\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

安装配置完成LLVM，然后在LLVM源代码中创建一个目录。例如，lib/Transforms/Hello。最后，必须写一个脚本编译新的pass。可以复制以下代码到lib/Transforms/Hello/Makefile:

——————————————————————

\# Makefile for hello pass

\# Path to top level of LLVM hierarchy

LEVEL = ../../..

\# Name of the library to build

LIBRARYNAME = LLVMHello

\# Make the shared library become a loadable module so the tools can

\# dlopen/dlsym on the resulting library.

LOADABLE\_MODULE = 1

\# Include the makefile implementation stuff

include \$(LEVEL)/Makefile.common

——————————————————————

这个Makefile指明了本目录下的所有.cpp文件将被编译连接到\$(LEVEL)/Debug+Asserts/lib/LLVMHello.so，可由opt工具通过其-load
选项动态加载。

\*\*编写时发现：需要修改lib/Transforms/Makefile，在’ PARALLEL\_DIRS = Utils
Instrumentation Scalar InstCombine IPO Vectorize
Hello’后面添加新增的文件夹。\*\*

现在已经建立了build脚本，只需要编写pass本身代码。

2.2 需要的基本代码
------------------

Hello world代码如下：

————————————————————————————————————————————————

//添加必须的头文件

\#include "llvm/Pass.h"

\#include "llvm/Function.h"

\#include "llvm/Support/raw\_ostream.h"

//使用llvm命名空间

using namespace llvm;

namespace {

struct Hello : public FunctionPass {//声明pass，是FunctionPass的子类

static char ID;//声明pass的ID，唯一标识一个pass

Hello() : FunctionPass(ID) {}

virtual bool runOnFunction(Function &F)
{//重写父类FunctionPass的runOnFunction函数

errs() \<\< "Hello: ";

errs().write\_escaped(F.getName()) \<\< '\\n';

return false;

}

}; // end of struct Hello

} // end of anonymous namespace

char Hello::ID = 0;//初始化pass的ID，初始化什么值不重要。

static RegisterPass\<Hello\> X("hello", "Hello World Pass",

false /\* Only looks at CFG \*/,

false /\* Analysis Pass \*/);

//注册自定义类，命令行参数“hello”，Pass名字“Hello World Pass”

//最后两行描述了行为：如果pass遍历CFG而不修改，第三个参数是true

//如果pass是一个分析pass，如dominator tree pass，第四个参数是true

————————————————————————————————————————————————

编写完成，在build目录下使用make指令编译这个文件，将得到lib/LLVMHello.so文件。

2.3 用opt执行pass
-----------------

首先，用c写一个hello world（略）hello.c，使用LLVM编译hello.c生成hello.bc。

———————————————————————————————————————

\$ opt -load lib/LLVMHello.so -hello \< hello.bc \> /dev/null

Hello: \_\_main

Hello: puts

Hello: main

———————————————————————————————————————

\-load指定opt加载的自定义链接库，已经注册了‘hello’，可以使用‘-hello’命令行参数。

查看注册的字符串，可以使用opt –help

———————————————————————————————————————

\$ opt -load lib/LLVMHello.so -help

OVERVIEW: llvm .bc -\> .bc modular optimizer and analysis printer

USAGE: opt [subcommand] [options] \<input bitcode file\>

OPTIONS:

Optimizations available:

...

\-guard-widening - Widen guards

\-gvn - Global Value Numbering

\-gvn-hoist - Early GVN Hoisting of Expressions

\-hello - Hello World Pass

\-indvars - Induction Variable Simplification

\-inferattrs - Infer set function attributes

...

———————————————————————————————————————

Pass名称将作为自定义pass的信息字符串，提供给opt的用户。

PassManager提供了一些指令帮助获取pass执行时信息。

例如，--time-passes获取执行时间相关信息

———————————————————————————————————————

\$ opt -load lib/LLVMHello.so -hello -time-passes \< hello.bc \> /dev/null

Hello: \_\_main

Hello: puts

Hello: main

===-------------------------------------------------------------------------===

... Pass execution timing report ...

===-------------------------------------------------------------------------===

Total Execution Time: 0.0007 seconds (0.0005 wall clock)

\---User Time--- --User+System-- ---Wall Time--- --- Name ---

0.0004 ( 55.3%) 0.0004 ( 55.3%) 0.0004 ( 75.7%) Bitcode Writer

0.0003 ( 44.7%) 0.0003 ( 44.7%) 0.0001 ( 13.6%) Hello World Pass

0.0000 ( 0.0%) 0.0000 ( 0.0%) 0.0001 ( 10.7%) Module Verifier

0.0007 (100.0%) 0.0007 (100.0%) 0.0005 (100.0%) Total

———————————————————————————————————————

**3. Pass相关类和需求**
=======================

在设计一个pass之前应该选择属于什么。例如HelloWorld例子是FunctionPass的子类。下面这些类是从一般到具体。当使用时，应选择最具体的类。

3.1 ImmutablePass 类
--------------------

该类很不常用，一个无聊的类，这种类型的pass不必运行，不改变状态，不需要更新。它不是通常的转换或者分析类型，但是可以提供当前编译器的配置信息。

3.2 ModulePass类
----------------

最一般的类，从ModulePass派生意味着pass使用整个程序作为一个单元，以不可预知的顺序引用函数体，添加删除函数。因为对子类的一无所知，所以没有优化操作。

ModulePass可以使用函数级的pass，使用getAnalysis接口来提供检索分析结果的函数。

使用时从ModulePass继承并且重写runOnModule方法。

virtual bool runOnModule(Module &M) = 0;

如果模块被修改runOnModule返回true。

3.3 CallGraphSCCPass类
----------------------

该类可以用来自底向上遍历调用图（callees before
callers）。该类提供了建立和遍历调用图的机制，也允许系统优化CallGraphSCCPasses的执行。

doInitialization(CallGraph &)

runOnSCC

doFinalization(CallGraph &)

3.4 FunctionPass类
------------------

与ModulePass相反，FunctionPass具有可预期的本地行为。所有
FunctionPass程序中的所有功能都独立于程序中的所有其他功能执行。FunctionPasses不要求它们按特定顺序执行，FunctionPasses也不要修改外部函数。

doInitialization(Module &)

runOnFunction

doFinalization(Module &)

3.5 LoopPass类
--------------

3.6 RegionPass类
----------------

3.7 BasicBlockPass类
--------------------

3.8 MachineFunctionPass类
-------------------------
