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

GCC中的AST数据结构表示: union tree_node  
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
[![1CrMgf.md.jpg](https://s2.ax1x.com/2020/01/19/1CrMgf.md.jpg)](https://imgchr.com/i/1CrMgf)

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