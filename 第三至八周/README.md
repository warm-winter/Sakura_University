# 傻哭拉大学的第三至八周学习生涯
对于cs143 编译原理(compiler的学习)

### 1.Lexer-词法分析

+ 在此任务中，我们主要完成对cool.flex的编写，将其输出为对应的c++源码，对其编译就得到lexer

+ cool.flex中，主要编写符合cool语法的正则表达式的匹配规则，通过正则匹配对应的字符串，返回token。

+ 词法分析过程中，运用了状态机的思想，主要有四种自定义的状态

  ```
  %x COMMENT
  %x S_LINE_COMMENT
  %x STRING
  %x STRING_ERR
  ```

  分别对应了cool语法中的注释(其中分为多行注释和单行注释) 和 字符串

+ 可以滤出的错误是
  - 过长的字符串(超出1025)
  - 注释符合，引号未闭合

### 2.Parser - 语法分析

+ 语法分析使用的是bison
+ 该任务的主要工作是构建一个AST抽象语法树，但是其只是token的组合，还需要进行处理
+ 其是一个逐渐归约的过程，将一个个token压入栈中，与归约条件进行匹配，符合则将两个或两个以上符合归约条件的token规约成一个新token

### 3.Semant - 语义分析

+ 此任务主要编写semant.cc，对Parser构成的AST抽象语法树进行二次处理
+ 通过设定Environment(建立各种符号表)来约束各个类，方法的作用域
+ 类型检测，在继承等前提下，对属性，方法的定义和使用是否存在重定义，是否构成cyclic，返回类型是否匹配定义，推断类型是否符合等一系列判断