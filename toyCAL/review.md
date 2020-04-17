# 自己动手构造编译系统
## 编译系统
- 编译器: 输入为源文件(\*.c),输出为汇编文件(\*.s)  
  词法分析->语法分析->符号表管理->语义分析->编译优化->代码生成  
  现代编译器结构如图:  
  [![JEHfht.md.png](https://s1.ax1x.com/2020/04/17/JEHfht.md.png)](https://imgchr.com/i/JEHfht)  
  [![JEbV9x.md.png](https://s1.ax1x.com/2020/04/17/JEbV9x.md.png)](https://imgchr.com/i/JEbV9x)
- 汇编器: 输入为源文件(\*.s),输出为可重定位文件(\*.o)   
  词法分析->语法分析->表信息生成(ELF文件中的段表,符号表,重定位表)->指令生成  
  汇编器结构如图:  
  [![JEbaDg.png](https://s1.ax1x.com/2020/04/17/JEbaDg.png)](https://imgchr.com/i/JEbaDg)  
- 链接器:输入为可重定位文件(\*.o),输出为可执行文件ELF  
  地址空间分配->符号解析->重定位  
  链接器结构如图:  
  [![JEqiM8.md.png](https://s1.ax1x.com/2020/04/17/JEqiM8.md.png)](https://imgchr.com/i/JEqiM8)  
- ELF文件格式  
  linux下ELF主要有四种:可执行文件,可重定位目标文件,共享目标文件(动态链接器相关),核心转储文件(coreDump,错误现场相关)  
  ELF格式如图:  
  [![JELCk9.png](https://s1.ax1x.com/2020/04/17/JELCk9.png)](https://imgchr.com/i/JELCk9)