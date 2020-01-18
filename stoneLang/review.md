# 两周自制脚本语言:
## 基础概念
汇编器: Asm->Binary, 直接由CPU执行二进制代码  
解释器: HLL->AST, 解析AST执行  
编译器: HLL->IR(AST, Linear)->Asm, 由汇编器翻译成二进制代码  

Lexer: base是RE,主要实现方式如下
1. 自动生成工具(输入词法描述,输出lexer): lex, Flex, JFlex...  
2. RE->NFA->DFA->minimalDFA->hand code

AST: ASTLeaf(终结符/叶节点) + ASTList(非终结符/非叶节点)  
语法树的生成过程:  
子树ASTList的生成会依次调用子节点的语法树生成函数,递归调用直到叶节点调用语法树生成函数,每一层可以用相关符号表示(括号较常用)

Parser:	常见的语法描述文法有
1. CFG: 通常用BNF描述, 弥补RE的缺陷(1.表述优先级2.无限的括号配对)  
2. PEG: parsing expression grammar, 不是很常见 

两者主要区别:  
A CFG grammar is non-deterministic, meaning that some input could result in two or more possible parse-trees. Though most CFG-based parser-generators have restrictions on the determinability of the grammar. It will give a warning or error if it has two or more choices.
A PEG grammar is deterministic, meaning that any input can only be parsed one way.

CFG描述的parser主要实现方式:  
1. 自动生成工具(输入语法描述, 输出parser): yacc, bison...  
2. handcode parser  
    - 组合子:  高阶函数,接收若干函数作为参数,返回它们的组合,原本为Haskell这类函数型语言中的一种程序设计技巧.  
        - Parser combinator: 组合简单语法分析函数, 获得一个可以parse复杂语法的函数  
        - Y-combinator(fixed-point combinator): 主要用于表述递归计算  
    - 递归下降算法(常用): handcode LL(1)

语法分析算法主要分为  
自顶向下: LL(left-to-right, leftmost derivation), 递归下降算法  
自底向上: LR(left-to-right, reverse rightmost derivation), LR算法一般不手工构造

DSL: domain-specific language(gcc的spec, llvm的tableGen)  
内部DSL/嵌入式DSL: 库(?)  
外部DSL: 严格意义上的DSL?(脚本语言?)  
DSA: domain-specific architecture(AI处理器)

AST的执行(区别于编译器的地方):  
直接在AST节点上解释执行,而非根据AST翻译成机器语言(AST->IR->Asm),通常每个节点都会有一个类似于eval的函数用来表示该节点的执行.  
eval通常有一个表示环境的参数来存取计算结果,非叶节点通常会递归调用子节点的eval方法,表示整颗子树的执行.  
语句块的eval通常以最后一条语句的计算结果作为返回值

函数的实现与调用  
除了扩展函数相关语法之外,还需要实现函数执行时的环境,因此有可能存在环境嵌套  
作用域(主要指函数内部访问自由变量的处理方式)  
1. 静态作用域(自由变量和该引用处最近的作用域绑定)
2. 动态作用域(自由变量和运行时最近一次创建的同名变量绑定)

闭包: 并非匿名函数  
一种特殊的函数,可以当作参数赋值给参数或变量,由于消除自由变量而得名  
自由变量的绑定方式有两种:  
1. 绑定全局变量:赋值将直接操作该全局变量
2. 绑定局部变量:->延长该自由变量的生命周期

实现新特性的主要步骤:  
1. 扩展语法
2. 添加eval

解释器为了实现输入输出操作,可以关联实现该解释器的语言所提供的IO函数  
新特性主要包括: 类的实现(参考闭包), 数组的实现 

虚拟机:
- 寄存器机器: 内存通常分为stack, heap, 代码区, 文字常量区
- 堆栈式机器: 中间代码简洁, codesize较小(JAVA, Smalltalk 80)

根据中间代码执行实际运算的程序称为中间代码解释器或者虚拟机  
引申: JAVA虚拟机会在执行过程中将部分hot中间代码翻译为机器码(动态编译), 并非所有中间代码都转换为机器码

Interpreter模式: 类似于面向对象的多态(不同子类调用不同的实现方法)  
Visitor模式: 抽象出一个抽象方法层,不同的方法只要实现自己的对应该层函数即可  
Java反射: 运行时动态查找函数定义(进一步省略visitor模式中必备的接口定义)

## 性能优化
- 针对eval方法的参数: 环境  
用数组取代hashmap, 以index代替hash操作,主要针对函数的局部变量,因为局部变量的数量与变量名在函数定义完之后就全部确定   
为了能够通过index来访问局部变量,需要为每个AST节点添加lookup方法  
参数是一个保存变量名与对应位置关系的哈希表,函数扫描并记录局部变量出现位置,方便eval方法能够直接从数组而非hash表中取数据  
全局变量仍需通过hash表来实现  

- 对象性能优化  
每个同一类的对象的字段名和方法都是相同的,可以单独提取出来减少内存使用  
由于动态语言的特性,虽然编译时不能直接确定引用字段所属的对象,但是对于this所指的对象,仍然可以操作(用数组代替hash查找),因为此时所属的类可以确定(即使是子类也没有关系,因为继承不会改变父类的方法名与字段名和其对应位置的关系)  

- 内联缓存:  
对于非this对象的字段或方法名的引用,因为只有运行时才能确定所属对象,  
因此可以采用缓存的办法,在第一次运行完之后将引用的字段或方法保存,下一次即可加速(缓存的一般用处)

- AST->中间代码: 本质上还是将运行前能做的先做好    
避免AST的遍历操作, 函数定义后只需单独编译一次,后续直接调用即可,避免重复eval 

- 添加静态数据类型->类型检查->类型推论(对未指定类型的变量做类型推导,往往和类型检查一起执行)
- 脚本语言->JAVA: 脚本语言->AST->遍历AST生成相应的JAVA程序,利用成熟的已知语言虚拟机加速执行