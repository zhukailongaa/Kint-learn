Kint使用方法
============

1.Kint工作在LLVM字节码上，为了分析软件，第一步是生成LLVM字节码。Kint提供了一个脚本‘kint-build’，它同时调用了gcc(或g++)和clang对源代码进行编译，存储在.ll文件中。在源代码文件夹中执行以下指令。

    $ kint/build/bin/kint-build make

该脚本调用过程，kint-build -\> kint-gcc -\> kint-cc1 -\> clang & opt -\> gcc .

2.为了发现整数溢出，首先可以执行Kint在LLVM字节码上的全局分析，生成一些全局约束，减少后续分析步骤的误报。该步骤是可选的，如果不工作（例如出现bug）可以跳过执行。

    $ find . -name "\*.ll" \> bitcode.lst

    $ kint/build/bin/intglobal \@bitcode.lst

3.最后，在源代码文件夹中执行以下指令进行整数溢出检查。

    $ /home/john/program/kint/build/bin/pintck

最终的bug结构保存在‘pintck.txt’中。

代码结构及核心算法
==================

主要包括两个方面内容IntGlobal和Intck。

1.IntGlobal
===========

定义了入口函数main，main函数主要实现了以下功能：

1. 遍历输入文件，每个文件为一个模块，建立模块链表。

2. 调用AnnotationPass，利用元数据对指令进行标注。

3. 调用CallGraphPass进行调用图分析。

4. 调用TaintPass进行污点传播。

5. 调用RangePass进行范围分析。

维护全局上下文来记录上述分析结果，并最终以元数据形式标注到llvm代码中。

    GlobalContext {

    // 全局函数名字到定义的映射\<StringRef,Function\*\>

    FuncMap Funcs;

    // 函数指针(IDs)到可能分配的映射\<string,FuncSet\>

    FuncPtrMap FuncPtrs;

    // 调用点到所有潜在被调指令的映射\<CallInst\*,FuncSet\>

    CalleeMap Callees;

    // 污点信息：全局污点GTS，本地污点VTS

    TaintMap Taints;

    // 全局范围信息\<id, CRange\>

    RangeMap IntRanges;

    };

1.1 AnnotationPass
------------------

遍历指令，使用元数据对指令进行标注。

| 标注内容  | 元数据类型 | 说明                                                                                                                                                                                                                                                                  |
|-----------|------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Arguments | id         | 利用函数调用代替整数参数，call kint\_arg.i[bitwidth]，并替换所有使用                                                                                                                                                                                                  |
| Load      | id         | 使用load的第一个参数标注，仅标注全局变量和结构体类型                                                                                                                                                                                                                  |
| Store     | id         | 使用store的第二个参数标注，仅标注全局变量和结构体类型                                                                                                                                                                                                                 |
| Call      | taint\_src | 1.被调函数以kint\_arg.i开头，并且本函数以sys\_开头 2.被调函数为\_\_kint\_\_taint，用户自定义污点函数                                                                                                                                                                  |
| Call      | sink       | 被调函数是指定的alloc函数：*dma\_alloc\_from\_coherent, \_\_kmalloc, kmalloc, \_\_kmalloc\_node, kmalloc\_node, kzalloc, kcalloc, kcalloc, kmemdup, memdup\_user, pci\_alloc\_consistent, \_\_vmalloc, vmalloc, vmalloc\_user, vmalloc\_node, vzalloc, vzalloc\_node* |

1.2 CallGraphPass
-----------------

实现对代码中调用图的分析，主要通过维护全局上下文中三个映射。

    // 全局函数名字到定义的映射\<StringRef,Function\*\>

    FuncMap Funcs;

    // 函数指针(IDs)到可能分配的映射\<string,FuncSet\>

    FuncPtrMap FuncPtrs; 存储的是什么？

    // 调用点到所有潜在被调指令的映射\<CallInst\*,FuncSet\>

    CalleeMap Callees; 实验中都是单个被调函数

·FuncPrtrs：

1. 初始化时，收集函数指针分配，包括全局函数指针、结构体中的函数指针、结构体数组中的函数指针。

2. store指令，操作数是函数指针，则加入到FuncPrtrs。

3. return指令，返回时函数指针，加入到FuncPrtrs。

4. call指令，被调函数的函数指针参数，加入到FuncPrtrs。

·Funcs：

1. 初始化时，收集全局函数定义。

·Callees：

1. call指令，将被调函数加入到Callees。

1.3 TaintPass
-------------

实现污点传播过程，主要通过维护全局上下文中的Taints映射。

    // 污点信息：全局污点GTS，本地污点VTS

    TaintMap Taints;

Taints中包含两个映射：全局映射GTS，局部映射VTS。污点传播算法：

1. 在AnnotationPass中被标记为taint的指令，添加到GTS和VTS中。

2. 对于call指令，若被调函数的参数为污点，则添加“被调函数的参数”到GTS。（通过函数调用传播）

3. 如果指令的操作数被标记为污点，则将指令添加到VTS。

4. 对于store指令，若操作数被标记为污点，则添加store的“目的操作数”到GTS。

5. 对于return指令，若操作数被标记为污点，则添加“return所在函数”到GTS。

6. 迭代上述过程直到没有新的污点产生。

7. 最后，将GTS和VTS中的污点标记以“taint”元数据标记到llvm代码中。

举例

    int foo1(int a){ //假设a已经是污点

    int b,c;

    b = a+1; //则根据3，b被标记为污点

    c = foo2(b);
    //根据2，因为b为污点，所以foo2的参数被记为全局污点，污点传播到函数foo2中

    return c; //根据5，若c为污点，则foo1被标记为污点，传播到调用foo1函数的地方

    }

1.4 RangePass
-------------

实现\<值-范围\>分析。通过维护全局上下文中的全局\<值-范围\>，每个基本块维护基本块内的局部\<值-范围\>。范围使用“上下限”表示：{Lower，
Upper}。

    // 全局范围信息\<id, CRange\>

    RangeMap IntRanges;

范围传递算法：

1. 初始化时，收集全局变量的初始值添加到IntRanges，包括全局整型常数，结构体中的整数域，数组中的整数域。

2. 基本块之间\<值-范围\>传递：

根据前继块的\<值-范围\>和终结符约束求得“更新范围”，所有前继块“更新范围”的并集作为本块的\<值-范围\>。

3. 基本块内和函数间\<值-范围\>传递：

>a. 对于store指令，同时更新“store目标地址”的“全局范围”和“局部范围”。

>b. 对于return指令，更新“return所属函数”的“全局范围”。

>c. 对于call指令，更新被“调函数参数”的“全局范围”；同时利用返回值，更新“call指令”的“全局范围”。

>d. 对于二元操作指令(加、减、乘、除、移位、与、或、异或等)，转换指令(cast)，选择指令，PHI指令，Load指令，call指令，更新相关值的“局部范围”。

4. 遍历函数，迭代上述2-3步骤，直到没有新的范围更新或超过最大迭代次数。

5. 最后，利用“intrange”元数据，将“load和call”指令的范围标注到llvm代码中。

注：因为只有这两个指令会给函数内引入外部未知的范围信息，标注着两个指令帮助完成最终的约束收集。

2.整数检查intck
===============

libintck.so提供的Opt优化pass

| 序号 | 源代码              | 生成的Opt可选优化              | 描述                                              | 描述                          |
|------|---------------------|--------------------------------|---------------------------------------------------|-------------------------------|
| 1    | LoadRewrite.cc      | \-load-rewrite                 | Rewrite load instructions                         | 重写load指令                  |
| 2    | OverflowIdiom.cc    | \-overflow-idiom               | Rewrite overflow checking idioms using intrinsics | 使用intrinsic重写溢出检查语句 |
| 3    | OverflowSimplify.cc | \-overflow-simplify            | Canonicalize overflow intrinsics                  | 规范化溢出内联函数            |
| 4    | IntLibcalls.cc      | \-int-libcalls                 | Rewrite well-known library calls                  | 重写’有名’的函数调用          |
| 5    | IntRewrite.cc       | \-int-rewrite                  | Insert int.sat calls                              | 插入int.sat调用               |
| 6    | IntSat.cc           | \-int-sat                      | Check int.sat for satisfiability                  | 检查int.sat满足性             |
| 7    | IntRewrite.cc       | \-fwrapv                       | Use two's complement for signed integers          | 对符号整数使用二进制补码      |
| 8    | IntSat.cc           | \-smt-model                    | Output SMT model                                  | 输出SMT模型                   |
| 9    | SMTSolver.cc        | \-smt-timeout=\<milliseconds\> | Specify a timeout for SMT solver                  | 指定SMT求解器超时时间         |

黄色代表intck中选择的opt选项

2.1 LoadRewirte
---------------

2.2 OverflowIdiom
-----------------

2.3 OverflowSimplify
--------------------

2.4 IntLibcalls
---------------

2.5 IntRewrite
--------------

在需要进行整数检查的位置插入“int.sat”的函数调用，并使用“bug”元数据标注。

插入点：所有二元操作指令和数组指针指令。

1. 对于加，减，乘指令，在指令前插入内建检查函数：llvm.[s\|u][add\|sub\|mul].with.overflow.\*；并插入call
int.sat，检查内建函数是否能满足。

2. 对于除法指令，插入断言(R==NULL) or (L==Smin and
R==MinusOne)，L是被除数，R是除数，Smin是有符号的最小值，MinusOne是各bit全为1的值。并插入call
int.sat，对该断言满足性进行检查。

3. 对于移位指令，插入断言：移位值是否大于上限。并插入call
int.sat，对该断言满足性进行检查。

4. 对于结构体类型中指针元素，插入断言：Idx \> n。并插入call
int.sat，对该断言满足性进行检查。

注：如果上述指令被标记为污点或者sink点，则call int.sat也被标记为污点或者sink点。

2.6 IntSat
----------

针对IntRewrite中插入的int.sat，对需要检查的断言收集约束，并调用SMT求解器求解可满足性。

1. 对于有“bug”元数据标注的call int.sat，获取其需要检查的断言。

2. 根据断言收集约束，约束包括“值约束”和“路径约束”。

>a. “值约束”包括范围约束；

>b. “路径约束”仅包含函数内路径，即从函数入口块到本块的路径约束。算法：前继块的路径约束，前继块到本块的条件约束，PHIGuard约束求交集，所有前继块的这些交集求并集即为本块的路径约束。

3. 根据约束生成SMT表达式，调用SMT约束求解器对SMT表达式进行求解，求解结果可能的状态如下：

        SMTStatus {

        SMT\_TIMEOUT = -1,

        SMT\_UNDEF,

        SMT\_UNSAT,

        SMT\_SAT,

        };

4. SMT\_SAT表示可满足，将其求解结果输出。
