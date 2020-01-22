# gcc internals
## 总体结构
依然是三段式的架构,代码主要目录结构:  
- 与gcc编译配置有关的config*文件
- lib*目录: 各种库文件,既包括通用的库文件,也包括一些语言相关的库文件
- gcc目录: gcc核心代码  
    - 各种语言相关的词法,语法等前端分析程序, e.g. gcc/cp, gcc/fortran, gcc/java...(C语言是默认的处理前端语言,部分代码在gcc目录下)
    - 与各种目标机器相关的机器描述文件, 主要在gcc/config目录下, e.g. gcc/config/alpha, gcc/config/arm...
    - 与前端语言和后端目标无关的核心处理代码  

gcc的逻辑结构及工作流程  
[![19gpxP.md.png](https://s2.ax1x.com/2020/01/19/19gpxP.md.png)](https://imgchr.com/i/19gpxP)  
图片上半部分表示gcc代码的逻辑结构,下半部分表示gcc(实际是cc1,未显示binutils等相关工具)编译器内部运行流程
- gcc代码逻辑结构(图片上半部分):
    - HLL Specific code: 主要集中在${GCC_SOURCE}/${Language}目录, 完成lexer, parser等功能,输出为语言对应的AST,并完成其Genericize(规范化)操作
    - Language & Machine Independent Generic code: 主要包括${GCC_SOURCE}/目录下的代码, 完成GIMPLE/RTL的生成,以及大量基于GIMPLE/RTL的处理及优化工作
    - Machine Descriptions: ${GCC_SOURCE}/config/${target}的子目录, 存放机器描述代码及其相应的.h,.c文件
    - Machine Dependent Generator code: ${GCC_SOURCE}/gen*.[ch], 主要功能为根据机器描述文件***生成***与目标机器相关的部分源代码,这部分生成的代码和其他gcc源代码一起编译生成machine dependent compiler(cc1).这么做的原因主要是因为在gcc设计阶段的代码是不完整的,缺少的部分主要包括目标机器相关的rtl构造及目标代码(目标机器的汇编)生成等部分的源代码,因此抽象出一层generator,方便扩展不同的目标机器.  
因此,参与编译,生成目标机器编译器的源代码主要包括图中build Time所在行的语言相关代码、通用代码、根据机器描述文件由机器相关代码生成器生成的代码三部分
- gcc(准确来说是cc1,gcc只是负责调用cc1)工作流程(图片下半部分)
    - parser: 词法分析->语法分析->生成AST->AST规范化
    - Gimplifier: AST->GIMPLE
    - Tree SSA Optimizer: GIMPLE->SSA->SSA optimizer
    - RTL Generator: SSA->RTL
    - Optimizer: RTL based optimize
    - Code Generator: 生成目标机器汇编代码  
parser这块对应于HLL Specific code  
Gimplifier、Tree SSA Optimizer、Optimizer对应于Language & Machine Independent Generic code  
RTL Generator和Code Generator对应于目标机器相关代码生成器根据机器描述生成的代码  

GCC build Flow  
- native build情况下采用bootstrapping的技术,整个gcc代码编译过程主要分为三个阶段:  
    - stage1: 使用一个现有的编译器编译gcc源码,生成一个新的编译器new_gcc1(xgcc);
    - stage2: 使用new_gcc1重新编译gcc源码,生成另外一个新的编译器new_gcc2;
    - stage3: 使用new_gcc2重新编译gcc源码,生成另外一个新的编译器new_gcc3;  
比较new_gcc2和new_gcc3,以此判断gcc是否编译成功
- cross build和Canadian cross build较为复杂,但是不需要通常的3-stage,相对来说速度较快.主要涉及目标机器的binutils和c库的提前编译,交叉编译如果目标机器OS是windows,可能会需要先编译一遍linux的编译器,然后再以此编译windows版本(和makefile flow有关,不知道后续会不会改进)
## HLL->AST/GENERIC
GCC的三种IR: AST(Language-specific), GIMPLE, RTL(target-specific)  
AST: 往往是语言相关的,一般包含了部分语言相关的AST节点  
GENERIC: 规范化的AST.一般来说,如果一种前端语言的AST均能用gcc/tree.h中的节点表示,那么该AST即为规范的AST(GENERIC)  

### GCC中的AST数据结构表示: union tree_node  
- 定义形式为: #define DEFTREECODE(SYM, NAME, TYPE, LEN)  
e.g. DEFTREECODE(ERROR_MARK, "error_mark", tcc_exceptional, 0)  
表示了一种树节点, 其TREE_CODE为ERROR_MARK,名称为"error_makr", TREE_CODECLASS为tcc_exceptional, 操作数的个数为0  
- 树节点基本信息:  
    - TREE_CODE: DEFTREECODE宏定义中的SYM参数,描述该节点代表了什么样的节点(语义描述)
    - NAME: DEFTREECODE宏定义中的NAME参数, 表示树节点名称,主要用来进行AST中间结果表示
    - TREE_CODECLASS(TCC): DEFTREECODE宏定义中的TYPE参数, 描述该树节点的TREE_CODE所属类型
    - LEN: DEFTREECODE宏定义中的LEN参数, 描述该树节点所包含的操作数的数目  

核心数据结构(GCC4.4.0代码,比较旧,仅供参考):
```C
union tree_node{
    struct tree_base base; //其他具体节点的基类
    struct tree_common common; //其他具体节点的基类
    /*
    各种具体的树节点e.g. 常量节点,声明结点,类型节点,列表节点...
    */
};
struct tree_base{
    ENUM_BITFIELD(tree_code) code: 16; //TREE_CODE
    /*
    各种flag e.g. 常量标志,无符号标志,只读标志...
    */
};
struct tree_common{
    struct tree_base base;
    tree chain;
    tree type;
};
```
关于struct tree_common中的两个字段chain, type
- tree chain: 将有一定关系的树节点连接成一个链表, e.g. 同一作用域的变量声明结点通过chain字段连接起来
常用宏TREE_CHAIN(node)来访问node节点的chain字段
- tree type: 在不同节点种表示含义不同,但基本含义表示该树节点关联的类型节点
常用宏TREE_TYPE(node)宏来访问node节点的type字段

各种AST节点中,声明结点最复杂,这里记录下声明结点的层次结构  
声明结点的两个基类
```C
struct tree_decl_minimal{
    struct tree_common common;
    location_t locus; //该声明在源文件的位置信息,包括文件名称和行位置
    unsigned int uid; //声明的一个数字标志
    tree name; //该声明名称所对应的标识符节点指针
    tree context; //该声明所在上下文信息
};
struct tree_decl_common{
    struct tree_decl_minimal common; //基本声明信息
    tree size; //该声明的位数
    ENUM_BITFIELD(machine_mode) mode:8; //机器模式
    /*
    misc,部分字段根据该节点的TREE_CODE不同而不同
    */
};
```
除了上述两个基类声明节点,还有三个基类节点也是以层级结构构成其他声明结点的基类  
[![1FsGDK.md.jpg](https://s2.ax1x.com/2020/01/21/1FsGDK.md.jpg)](https://imgchr.com/i/1FsGDK)

```C
struct tree_decl_with_rtl{
    struct tree_decl_common common;
    rtx rtl; //描述该声明对象所对应的RTX信息
};
struct tree_decl_with_vis{ 
    //变量声明和函数声明结构体的基类, 描述一些与变量及函数声明相关的标志和特殊用法,主要涉及该声明的一些visibility
    struct tree_decl_with_rtl common;
    tree assembler_name; //描述变量声明对应的汇编名称
    tree section_name; //描述该变量的section属性信息
    /*
    misc flag
    */
}
struct decl_non_common{
    //可以看作函数和类型声明的基类,主要定义了一些函数的变量、参数、返回值以及函数体等内容
    struct tree_decl_with_vis common;
    /*
    misc
    */
}
```
GCC提供了一些dump选项,可以输出GCC处理源代码过程中的AST/GIMPLE IR信息,e.g. -fdump-tree-original, -fdump-tree-all等  

### AST生成过程简介(以C语言为例):
- 词法分析: 字符流->token流, token结构体一般包括:  
    - 符号类型: CPP_EQ, CPP_NOT, CPPEOF.....
    - 标识符类型: If this token is a CPP_NAME, this value indicates whether also declared as some kind of type. Otherwise, it is C_ID_NONE.
    - 关键字标识: If this token is a keyword, this value indicates which keyword. Otherwise, this value is RID_MAX.
    - PRAGMA类型: If this token is a CPP_PRAGMA, this indicates the pragma that was seen.  Otherwise it is PRAGMA_NONE.
    - value: 对于一些词法符号来讲,不仅需要关注类型,还要关注值.e.g. 字符串常量符号, 其词法符号类型为CPP_STRING, 字符串的值由value指向的树给出
    - location: 位置信息  
早期GCC使用Lex/Flex工具进行词法分析,现在则为handcode(gcc/c-lex.c)
- 语法分析: 对token流进行语法推导,生成AST
早期GCC使用Yacc/Bison进行C语言的语法分析,现在为handcode(gcc/c-parser.c)  
GCC对C语言进行语法分析采用的是LL(2)递归下降算法,且C语言的词法分析嵌入在语法分析过程中,  
函数c_parse_file()是C语法分析的入口函数,该函数首先对当前的词法符号进行判断,根据当前符号类型来决定是预处理还是进行语法推导  
- AST GENERIC(optional): 不是所有语言的前端都会转换成GENERIC AST
- 语义分析: TODO
## AST/GENERIC->GIMPLE
AST是语言相关的, GENERIC是语言无关规范化的AST(所有AST节点都可以用gcc/tree.h中的树节点表示)  
但事实上,GCC很多语言的前端处理并不包含AST的GENERIC转换,而是直接将AST(language-specific)转换成GIMPLE(language-independent)  

AST/GENERIC和GIMPLE的区别:  
- AST/GENERIC语言相关, GIMPLE语言无关
- AST/GENERIC为树形结构, GIMPLE线性三地址码形式的中间表示序列,更方便后续的编译优化(GIMPLE语句中的操作数依然会大量使用树节点)
- AST/GENERIC的属性节点类型非常多, GIMPLE语句类型较少  

AST和GIMPLE形式上的区别:  
- 在GIMPLE中通过引入临时变量保存中间结果,将AST表达式拆分成不超过三个operand的tuples.
- AST中的控制结构(if-else, for, while)在GIMPLE中被转换为条件跳转语句
- AST中的Lexical Scopes在低级GIMPLE中被取消
- AST中的Exceptional Region被转换成一个单独的Exception Region Tree.

AST->GIMPLE过程:  
AST->High-Level GIMPLE->Low-Level GIMPLE, High-Level GIMPLE到Low-Level GIMPLE的转换pass为*pass_lower_cf*  
High-Level GIMPLE和Low-Level GIMPLE的区别主要是抽象层次的高低(HL中含有一些表示作用域的语句,嵌套表达式...)

gcc/gimple.def文件声明了各种GIMPLE语句  
e.g. DEFGSCODE(GIMPLE_COND, "gimple_cond", struct gimple_statement_with_ops)  
声明主要包括:
- GIMPLE_CODE, 第一个参数,描述GIMPLE语句的语义,例子的GIMPLE语句为条件语句
- 名称, 第二个参数,打印名称时使用
- GIMPLE语句操作数的偏移量, 第三个参数, 以DEFGSCODE宏定义中使用的结构体大小来计算,该GIMPLE语句存储时, 使用的结构体为struct gimple_statement_with_ops
通过该结构体的相关信息,可以计算GIMPLE_COND语句操作数的偏移量,从而访问其操作数  

GIMPLE的表示与存储类似于AST节点,都是类似于面向对象的那种由基类分别派生出子类的结构  
GIMPLE分层主要是因为有些GIMPLE语句中带有操作数, GIMPLE结构体只有第一个操作数的指针(tree op[1]),后续操作数的结点指针将被连续存放在tree op[1]之后的地址中

AST/GENERIC经过转换将形成一系列的GIMPLE语句,GCC将这些GIMPLE语句组织成一种线性的序列,通过线性序列的起始节点就可以逐一遍历  
为了方便对所有GIMPLE语句序列进行操作,GCC还定义了一个GIMPLE序列的描述节点,该序列节点包括三个字段  
分别指向每个GIMPLE序列的第一个语句节点、最后一个语句节点、以及下一个空闲的序列节点
### GIMPLE的生成(gcc/c-gimplify.c)
一般而言***GIMPLE生成以函数为单位***进行.GCC前端完成一个函数的词法/语法分析->函数AST, 调用函数  
```C
c_genericize(tree fndecl)  
```
对函数fndecl的AST进行规范化处理,在函数c_genericize(tree fndecl)中,会进一步调用  
```C
gimplify_function_tree(fndecl)
```
将函数AST->GIMPLE序列,参数***fndecl***就是将要转换的函数声明节点,框架如下:
```C
void c_genericize(tree fndecl){
    /*
    ...
    */
    gimplify_function_tree(fndecl); //以函数为单位,完成AST/GENERIC到GIMPLE的转换
    /*
    ...
    */
}
```
## GIMPLE PASS
pass数据结构一般包括:
- pass的类型: GCC中pass分为四大类
    - GIMPLE_PASS: 以GIMPLE中间标识为处理对象
    - RTL_PASS: 处理RTL中间表示
    - SIMPLE_IPA_PASS: 
    - IPA_PASS: 和SIMPLE_IPA_PASS均处理GIMPLE中间标识,但是功能主要集中在IPA(Inter-Procedural Analysis)的处理上  
    即函数间的变量传递和调用关系等
- 名称
- 执行条件: bool gate (function *fun);
- 执行函数
- 静态编号
- sub指针: 子链
- next指针
- misc

几个重要的基于GIMPLE的pass:
- pass_lower_cf: 主要功能为将Hign-Level GIMPLE转换成Low-Level GIMPLE
- pass_build_cfg: 对函数的GIMPLE序列进行分析,完成基本快的划分,根据GIMPLE语义构造基本块之间的跳转关系
- pass_build_cgraph_edges: 构造函数调用图,调用关系可以使用有向图的形式进行描述
- pass_build_ssa: 将GIMPLE转换成SSA形式
- pass_all_optimizations: 完成GCC预定义的all_passes链中的基于GIMPLE和RTL的各种优化处理  
GCC预定义的pass链如下(partly):
    - all_passes: 完成基于GIMPLE和RTL的各种优化及其相关处理
    - all_ipa_passes: IPA优化
    - all_lowering_passes: 完成函数GIMPLE序列的低级化处理  
    这三个分别包含了不同数量和内容的pass,是否执行一般由该pass中的gate()函数决定,同时也依赖于gcc编译时所使用的优化选项
- pass_expand: 将GIMPLE转换为RTL
## RTL(Register Transfer Language)
RTL采用了类似LISP的前缀表达式形式,描述了每一条指定的语义动作,根据作用分为两大类:
- Internal Form: 由GIMPLE转换而来,也是一种IR,可以成为IR-RTL(Intermediate Representation RTL)
- Textual Form: 用于Machine Description文件中, 进行机器描述时所采用的RTL形式, 称为MD-RTL(Machine Description RTL)

GIMPLE->IR-RTL:  
按照GCC设计时所定义的规则,将每个GIMPLE语句转换成具有某个SPN(Standard Pattern Name)对应的RTL,这个转换规则是机器无关的.  
从MD-RTL来看,某个SPN所定义的指令模板则是与机器相关的,对应于不同的机器,其实现内容也是各不相同的  
从IR-RTL来看,这些SPN所对应的操作语义则是***与机器无关的***.   
通过SPN来将GIMPLE_CODE和MD-RTL中具有SPN的指令模板进行匹配,实现机器无关的GIMPLE表示到机器相关的RTL之间的转换.  
因此,在使用MD-RTL描述目标机器特性时,必须定义这些SPN所对应的指令模板  

GIMPLE->IR-RTL->Asm
[![1FNw38.md.png](https://s2.ax1x.com/2020/01/21/1FNw38.md.png)](https://imgchr.com/i/1FNw38)  
MD-RTL主要用来描述机器的指令模板,其中具有SPN的指令模板用来指导IR-RTL的构造,从而实现机器无关的GIMPLE表示到机器相关的IR-RTL之间的转换.  
IR-RTL->Asm: 进一步根据MD-RTL中所定义的所有指令模板, 完成IR-RTL到指令模板的匹配,根据匹配指令模板中的汇编代码输出格式生成汇编代码  
***SPN(核心)***解耦了机器相关的RTL/Asm与机器无关的的GIMPLE.  

RTL中的对象类型
IR-RTL共五种Object Type: Expression, Integer, Wide Integer, String, Vector  
- Integer: 类型为int的简单类型
- Wide Integer: 数据类型为HOST_WIDE_INT
- String: 类似于C
- Vector: 包含任意数量的RTX表达式
- Expression(重点): 也称为RTX(RTL eX pression),是RTL最重要的一类对象.
 
### RTX
宏定义声明(gcc/rtl.def):  
DEF_RTL_EXPR(RTL_CODE, NAME, PRINT_FORMAT, RTX_CLASS)  
包括四个部分:  
- RTL_CODE: RTX的语义标识, 与TREE_CODE, GIMPLE_CODE类似.***注意:RTX_CODE所表达的语义是机器无关的.***
- NAME: 输出该RTX时的字符串表示
- PRINT_FORMAT: 描述了该RTX操作数的输出格式, 同时也描述了这些操作数的类型及操作数的个数
[![1FyDJJ.md.png](https://s2.ax1x.com/2020/01/21/1FyDJJ.md.png)](https://imgchr.com/i/1FyDJJ)  
- RTX_CLASS: 描述了RTX的分类类型  
(e.g. RTX_CONST_OBJ(常量), RTX_OBJ(寄存器和内存), RTX_COMPARE(比较运算), 算术运算...)  

有一类RTX比较特殊,与表示"值"的RTX不同,这些特殊的RTX表示"动作", 这些RTX所描述的"动作"称之为RTX的Side Effect.  
一般来说, 机器指令insn(描述程序代码信息的IR-RTL)的body部分通常是具有side effect的RTX, 用来表示一些"动作", 其余的RTX通常只是作为这些具有side effect的RTX的操作数出现.  
常见的side effect RTX:  
- (set lval x): 将值存储到lval对应的RTX中
- (return)
- (call function nargs): 函数调用, function时一个mem表达式,值为被调函数的地址
- (clobber x): x的值可能会被修改
- (use): 需要使用x的值

RTX定义: e.g. DEF_RTL_EXPR(GE, "ge", "ee", RTX_COMPARE)  
该定义表明了一个表示"大于或等于"语义的RTX, 输出字符串名称为"ge", 类型为RTX_COMPARE, "ee"为操作数的输出格式(两个操作数)
对于RTX来说,有些RTX只能出现在IR-RTL中,有些只能用于MD-RTL,有些both

### RTX MACHINE MODES(gcc/machmode.def)  
机器模式表示在机器层次上数据的大小及其格式,***每个RTX均有其机器模式的描述***  
GCC也支持定义与特定目标机器相关的机器模式(config/${target}/${target}-modes.def)
gcc internals原文:  
A machine mode describes a size of data object and the representation used for it. Each RTL expression has room for a machine mode and so do certain kinds of tree expressions (declarations and types, to be precise). In debugging dumps and machine descriptions, the machine mode of an RTL expression is written after the expression code with a colon to separate them. The letters ‘mode’ which appear at the end of each machine mode name are omitted. For example, (reg:SI 38) is a reg expression with machine mode SImode. If the mode is VOIDmode, it is not written at all.

### RTX的存储
RTX使用结构体rtx_def进行存储(gcc/rtl.h),主要包括两部分:
```C
struct rtx_def{
    //Header
    ENUM_BITFIELD(rtx_code) code: 16;
    ENUM_BITFIELD(machine_mode) mode : 8;
    /*
    misc flag
    */

    //RTX的第0操作数
    union u{
        /*
        各种不同类型的操作数
        */
    }
}
```
- RTX Header: 所有RTX首部长度相同,描述了RTX_CODE, 机器模式, RTX flag....
- RTX的第0操作数. RTX的操作数使用union u进行存储,表示各种不同类型的操作数  
由于各种RTX包含的操作数数目和类型都不相同,所以每种RTX的实际存储大小也不尽相同
### IR-RTL
RTL可以用来进行机器描述(MD-RTL),也可以描述由GIMPLE转换而来的程序代码信息,GCC中描述程序代码信息的RTL也被称为insn  
用来表述程序中的算术运算、程序跳转、标号等,也可以用来表示各种说明信息  
[![1Fskj0.md.png](https://s2.ax1x.com/2020/01/21/1Fskj0.md.png)](https://imgchr.com/i/1Fskj0)  
所有的insn被一个双向链表所连接(这6种insn的前三个操作数都一样,第二和第三个操作数uu用来连接前后insn)  

以RTX_CODE为INSN(非跳转、非函数调用的指令)的insn举例
e.g. DEF_RTL_EXPR(INSN, "insn", "iuuBieie", RTX_INSN)  
- RTX_CODE: INSN
- RTX_类型: RTX_INSN
- 输出格式: iuuBieie(第0,1,2操作数所有insn都一样,分别是insn的UID值,insn的前驱节点,后继节点)
    - 第3操作数"B": 基本快信息指针(basic bolck)
    - 第4操作数"i": 描述了该INSN对应的源代码的行数
    - 第5操作数"e": INSN的主体(称为insn的pattern或者body部分), 描述了该INSN的指令模板
    - 第6操作数"i": ***INSN_CODE, 即该INSN操作所对应的指令模板索引值***
    - 第7操作数"e": 未使用

INSN实例(JUMP_INSN, CALL_INSN, BARRER, CODE_LABEL, NOTE都类似,偷个懒🤭)  
[![1F6hn0.md.png](https://s2.ax1x.com/2020/01/21/1F6hn0.md.png)](https://imgchr.com/i/1F6hn0)  

## Machine Descriptions(MD-RTL)
机器描述主要包括两部分:
- Machine Descriptions文件(${target}.md),主要描述目标机器所支持的每条指令的Instruction Pattern
- 机器描述的头文件及c文件(${target}.[ch]),主要描述了与机器相关的变量声明与函数实现  
同时配合md文件,实现其指令模板里变量的定义及其函数代码的实现等  

整个GCC编译器的编译过程主要包括:  
[![1FXuIe.md.jpg](https://s2.ax1x.com/2020/01/21/1FXuIe.md.jpg)](https://imgchr.com/i/1FXuIe)  

机器描述就是使用规范的RTL语言(MD-RTL)对目标机器的特性进行描述的文件,尤其是其中指令模板的定义,指导了GIMPLE IR向IR-RTL转换的具体形式  
主要包含:  
Insn Pattern定义、Constang定义、Attribute定义、User-Defined Predicate、User-Defined Constraint  
Iterator定义、Pipeline声明、Peephole Optimization定义  
[![1FvFnx.md.jpg](https://s2.ax1x.com/2020/01/21/1FvFnx.md.jpg)](https://imgchr.com/i/1FvFnx)  
### 指令模板(define_insn)
MD文件包含了目标机器所支持的每一条指令的指令模板,形式如下:  
```Lisp
(define_insn 指令模板名称
    RTL模板
    条件
    输出模板
    属性
) 
```
指令模板定义中的五部分:  
- 指令模板名称: 由字符串给出, 唯一的标识了该指令模板
- RTL模板: 提供了一个不完整的RTX向量, 用来描述指令的RTX表示形式  
如果RTX向量只有一个元素,表示普通的指令模板,如果由多个元素,表示并行(Parallel)的指令模板
- Condition: C表达式,判断某个insn是否与该指令模板匹配的最后条件
- Output Template: 描述该指令模板转换成目标汇编代码时的输出格式
- Attribute: 设置指令的一些属性值,可忽略  

mips的一条指令模板的具体实例(gcc/config/mips/mips.md)如下:  
[![1FxfJg.md.jpg](https://s2.ax1x.com/2020/01/21/1FxfJg.md.jpg)](https://imgchr.com/i/1FxfJg)  
#### 1.模板名称
SPN(Standard Pattern Name)是GCC中预定义的一些指令模板的名称  
GIMPLE->IR-RTL的转换可以看做两个阶段:
1. GIMPLE->SPN: 每个SPN只是某个IR-RTL模板的抽象代表,某个GIMPLE语句到SPN的mapping是GCC事先预定好的(Hard-coded),***与机器无关***
2. SPN->IR-RTL: 根据SPN从MD文件中提取与SPN相匹配的指令模板, 根据该模板构造出对应的IR-RTL.***与机器紧密相关***
注: 只有命名的define_insn/define_expand指令模板才会用来构造insn
#### 2.RTL模板
[![1kSCcj.md.jpg](https://s2.ax1x.com/2020/01/21/1kSCcj.md.jpg)](https://imgchr.com/i/1kSCcj)  
RTL模板的功能主要有两种:  
- Matching: 用来描述包含该RTL模板的指令模板是否可以和特定的IR-RTL完成匹配,以及如何获得该IR-RTL的操作数.对于匹配操作,IR-RTL的形式及其操作数应该满足该RTL模板所要求的匹配条件.匹配发生在insn已经完成,需要进行insn后续操作(利用insn生成目标汇编)
- Construction: 对于命名的指令模板, RTL模板描述了如何利用给定的操作数实现该指令模板对应的insn的构造.对于构造来说,即从GIMPLE->insn时,需要从GIMPLE语句中提取相应的操作数,并替换RTL模板中的操作数占位符.构造发生在insn的生成过程(GIMPLE生成RTL时)中.
    ##### Predicate(define_predicate)
    断言是RTL模板中操作数是否满足某种条件的判定过程,通常由一个描述断言函数名称的字符串来表示  
    GCC中的断言可分为Machine-Independent Predicates和Machine-Dependent Predicates
    ##### Constraint(define_constraint)
    约束对断言所允许的操作数进行更详细的描述

#### 3.Conditions
指令模板中的条件是一个C表达式, 判断该指令模板是否匹配的最后条件
#### 4.输出模板
表示该指令模板匹配后如何输出目标汇编代码
#### 5.属性(define_attr)
对指令属性进行设置, 通常在指令模板及流水线优化中有较多的使用

### 定义RTL序列(define_expand)
***在某些特定的机器上***,一个具有SPN的指令模板所生成的指令不能被一条insn所表示,  
但是可以用一系列的insn表示,此时就可以利用define_expand来描述如何生成一系列的insn
与define_insn的不同点: 
- define_ expand只在RTL构造时使用
- define_expand定义中的准备语句也是define_insn没有的:  
包含0个或者多个C语句的字符串,这些C语句在从RTL模板生成RTL代码之前执行.通常这些语句会给RTL生成准备一些临时的reg,作为RTL模板中的内部操作数使用  
当然这些语句也可以通过调用一些例程(emit_insn)等直接生成insn, 这些insn将插入在由RTL模板生成的insn之前
### 指令拆分(define_split)
***在某些特定的目标机器中***,可以对一些复杂的指令模板进行分解,即将一个复杂的指令模板分解成多个简单的指令模板,从而生成多个简单的insn
### Iterator
在机器描述文件中,通常对针对不同的机器模式或者RTX_CODE书写大致类似的指令模板.为了避免书写复杂  
通常可以使用Iterator进行指令模板的简化, 分为:  
- mode枚举器(define_mode_iterator): 机器模式枚举器
- code枚举器(define_code_iterator): 主要用于对RTX_CODE的扩展
### Peephool Optimization
窥孔优化是一种局部优化方式,可以认为是一个滑动窗口,只分析窗口内的指令,看能否合并成一条指令
在机器描述文件中,有两种窥孔优化方式:
- define_peephole: RTL to Text Peephole Optimizers
- define_peephole2: RTL to RTL Peephole Optimizers

## 机器描述文件${target}.[ch]
{target}.md使用MD-RTL对目标机器的指令生成进行了详细描述,但是还有一些如:  
寄存器信息、storage layout以及一些与硬件相关的函数实现等无法使用RTL进行描述,因此需要${target}.[ch]  
[![1kVHmD.md.jpg](https://s2.ax1x.com/2020/01/21/1kVHmD.md.jpg)](https://imgchr.com/i/1kVHmD)  
- targetm(target.h):  
```C
struct gcc_target targetm;
//描述目标机器的结构体,异常复杂,主要包含汇编代码输出、指令调度、向量化、函数参数传递、函数返回以及其他大量与目标机器相关的信息
```
- 编译驱动及选项:  
GCC本质上是一个编译驱动程序,它通过调用一系列其他程序来完成compile、汇编、链接等工作  
GCC需要对命令行参数进行解析,从而根据命令行参数判断需要调用哪些程序,以及向这些程序传递什么样的命令行参数  
这些工作由GCC内部定义的SPEC字符串完成
- SPEC语言及SPEC文件:  
一种用来描述GCC传递给各个处理程序的命令行参数的DSL
- storage layout:  
主要定义目标机器中数据存储的格式、大小、对齐方式...
- 寄存器使用:主要定义目标机器中的寄存器用法,包括  
目标机器物理寄存器的数量(本质上只是微架构暴露出来的接口,并非物理寄存器,真正的物理寄存器会在寄存renaming中使用,对程序员透明)  
寄存器的初始化、寄存器分配顺序、寄存器在函数调用时是否需要保存、寄存器名称、寄存器类型...
- 堆栈及函数调用规范描述
通常要对堆栈的增长方向、堆栈的布局、以及一些描述堆栈地址的寄存器进行设置
- 寻址方式
- 汇编代码分区
- 定义输出的汇编语言
- 机器描述信息的提取: 
GCC源码中包含了一些名称为gcc/gen*的文件,这些文件的主要功能就是读取、机器描述文件,并生成与目标机器相关的源代码.  
这些以gen开头的文件被称为Machine-Dependent Generator Code(MDGC),  
其编译生成的可执行程序就用来从目标机器的描述文件中提取信息,并生成与目标系统相关的源代码.  
生成的目标机器相关源代码将与GCC的其他源代码一起,编译生成目标机器上的compiler
## GIMPLE->RTL
准确来讲,是GIMPLE->IR-RTL的过程,即GIMPLE转换成insn序列的过程
众多基于GIMPLE的pass中,pass_expand完成了GIMPLE向RTL的转换  
在以函数为单位进行RTL生成时(GIMPLE也是以函数为单位转换的),需要对当前函数的RTL信息进行维护,这个主要由struct rtl_data来描述  
pass_expand主要处理框架(以函数为单位):  
- 变量展开: 对当前函数中所有变量进行分析,在虚拟寄存器或者堆栈为其分配空间,生成对应的RTX
- 参数和返回值的处理: 对函数的参数和返回值进行处理,生成对应的RTX
- 初始块的处理: 创建初始块,并修正函数的CFG.
- 基本块的展开: 对函数体中每个基本块所包含的GIMPLE语句序列逐个进行展开(RTL生成的主要部分)
- 退出块的处理: 创建退出块, 生成函数退出的RTL, 并修正函数的CFG
- MISC: 重建基本块、异常处理...

GIMPLE语句转换成RTL 
一般来讲,一条GIMPLE语句生成RTL时,通常先将GIMPLE语句转换成树的存储形式,  
再根据树中表达式节点的TREE_CODE值,调用相应的函数生成对应的insn表示,详情如下:  
1. GIMPLE语句转换成树形结构  
这个阶段GIMPLE_CODE与TREE_CODE的映射关系是与目标机器无关的  
关于GIMPLE为什么不直接转换成RTL,而是需要中间树结构作为媒介,GCC代码(4.4.0)注释如下:  
RTL expansion has traditionally been done on trees, so the transition to doing it on GIMPLE tuples is very invasive to the RTL expander. To facilitate the transition, this function takes a GIMPLE tuple STMT and returns the same statement in the form of a tree.

2. 从树形结构生成RTL(两个关键步骤):  
    - 选择构造insn的指令模板:  
    因为目标机器上指令模板的内容千差万别,很难直接定义树节点的TREE_CODE和指令模板的对应关系, 因此GCC引入了***SPN(Standard Pattern Name)***.  
    作为TREE_CODE(表示某种语义操作)和对应指令模板之间的"映射中介",某种TREE_CODE和SPN的对应关系是由GCC系统确定,且和目标机器无关的.  
    SPN在MD文件中表现为指令模板的名称,通过定义具有SPN的指令模板的索引号,就可以通过SPN作为纽带, 根据TREE_CODE查找出对应的指令模板索引号   
    从而完成TREE_CODE到指令模板的映射.
    - 根据模板中的操作数信息,从树形结构中提取操作数,并利用指令模板中提供的构造函数来构造insn的主体,从而生成insn.  
    提取指令模板中RTX构造函数,利用构造函数和相应的树结构中提取的操作数来完成insn的构造
## RTL处理及优化
在GCC预定义的all_passes链中,执行了pass_expand过程之后,GCC的中间标识就已经转换成RTL形式,此后所有的pass都是基于RTL的处理过程(RTL_PASS)  
所有的RTL处理***都包含在***pass_rest_of_compilation的处理过程及其子过程中,主要包括了对pass_expand所生成的insn序列进行进一步的处理,  
包括循环优化、寄存器分配、窥孔优化...  
RTL重要pass简介:  
### pass_instantiate_virtual_regs: 
特殊寄存器的实例化,将虚拟寄存器实例化成目标机器上特定的物理寄存器(硬件寄存器)  
- 注:特殊寄存器实例化之后的物理寄存器在给定的目标机器上并不一定都是存在的,  
因此在后续的寄存器分配过程中,可以使用寄存器消除的方法将这些寄存器替换成真正存在的物理寄存
### 指令调度(Instruction Scheduling):  
对当前函数的insn序列进行重新排列,更充分的利用目标机器的硬件资源,提高指令的执行效率.  
指定调度主要考虑因素包括Data Dependency、Control Dependency、Structural Hazard、指令延迟或者指令cost...  
通常指令调度与目标机器中的流水线设置紧密相关  
- Data Dependency: True Dependenc(RAW)、Anti-dependence(WAR)、Output Dependence(WAW)、Input Dependence(RAR)  
    WAR和WAW没有逻辑上的相关性,只是由于共用了同一个寄存器而存在相关性(ISA暴露的寄存器太少),被称为伪相关(可通过CPU内部的寄存器重命名解决)
- Control Dependency: 条件跳转等指令的锅(CPU内部可通过Speculative Execution解决,其实就是预测)
- Structural Hazard: 通常是指由于目标机器的硬件资源限制而导致的相关性冲突  
- GCC中的指令调度主要包括两个Pass,即
    - pass_sched: 在寄存器分配之前进行
    - pass_sched2: 在寄存器分配之后进行  

- GCC的指令调度算法(List Scheduling为默认算法):  
[![1AN2Ct.md.jpg](https://s2.ax1x.com/2020/01/22/1AN2Ct.md.jpg)](https://imgchr.com/i/1AN2Ct)
### Register Allocation(太复杂了,偷懒🤭)
寄存器分配主要有两种算法:
- 图着色算法(GCC4.4.0使用的这个,不知道最新版本情况)
- 线性扫描算法(貌似LLVM用的就是这个)
### 汇编代码生成: pass_final