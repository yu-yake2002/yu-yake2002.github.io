---
layout: post
title:  "软件分析期末复习"
date:   2024-01-08 09:00:00 +0800
categories: SE chinese
---

## 软件分析概述

###  静态分析（Static Analysis）和动态测试（Dynamic Testing）的区别是什么？
- **静态分析（Static Analysis）**是指在实际运行程序$P$之前，通过分析静态程序$P$本身来推测程序的行为，并判断程序是否满足某些特定的性质（Property）$Q$。
- 动态测试是指通过运行程序$P$，收集程序的运行时信息来观察代码运行状况。

### 完全性（Soundness）、正确性（Completeness）、假积极（False Positives）和假消极（False Negatives）分别是什么含义？

一个静态分析$S$。我们定义程序P的关于$Q$的真实行为为真相（Truth）。
- **完全性（Soundness）**：真相一定包含在$S$给出的答案中；
- **正确性（Completeness）**：$S$给出的答案一定包含在真相中。

记这个静态分析程序给出的答案集合$A$，真相集合为$T$，则完美的静态分析满足：

$$T \subseteq A \wedge A \subseteq T \Leftrightarrow A = T$$

其中，$T \subseteq A$体现了完全性，$A \subseteq T$体现了正确性。

- 假积极：记程序$P$的关于性质$Q$的静态分析$S$有**假积极（False Positive）**问题，当且仅当$S$给出的答案$A$和$P$关于$Q$的真相$T$满足如下关系：

$$\exists a \in A, a \notin T$$

其中，$a$称为一个**假积极实例（False Positive Instance）**，其实是一个**消极实例（Negative Instance）**。

- 假消极：记程序$P$的关于性质$Q$的静态分析$S$有**假消极（False Negative）**问题，当且仅当$S$给出的答案$A$和$P$关于$Q$的真相$T$满足如下关系：

$$\exists a \notin A, a \in T$$

其中，$a$称为一个**假消极实例（False Negative Instance）**，其实是一个**积极实例（Positive Instance）**。

### 为什么静态分析通常需要尽可能保证完全性？

- Sound 的静态分析可以帮助我们有效的缩小 debug 的范围，我们最多只需要在$A$范围内暴力排查掉所有的假积极实例（False Positive Instance）就可以了，人工排查的代价是可控的。
- Complete 的静态分析做不到这一点，它不能够帮助我们有效缩小 debug 的范围。因为假消极实例（False Negative Instance）$a\notin A$，所以$a$的范围是$P−A$。这里注意的是，虽然假消极的理论范围是$T−A$，但因为我们并不知道$T$是什么，所以只能从$P−A$中排查。而$P−A$往往是比$A$大得多的，因此排查假消极的代价是很大的。

### 如何理解抽象（Abstraction）和过近似（Over-Approximation）？

- **抽象（Abstraction）**：当我们考虑程序$P$的性质$Q$时，程序$P$中的各种值，我们或许不一定非得事无巨细。比如说，当我们考虑除零错误（Zero Division Error）的时候，对于某个值，我们只需要判定其是否为$0$即可，至于它具体是多大，我们其实不关心，因为它和我们要研究的性质$Q$没有直接关联。这种将$P$中的值的，和我们需要研究的性质$Q$相关的性质提取出来，从而忽略其他细节的过程，就是一个抽象的过程。
- **过近似（Over-Approximation）**：软件分析中的转移函数与控制流，针对程序中的语句，对抽象域上的结果提供转换规则，达到soundness的目标。

## 程序的中间表示
### 编译器（Compiler）和静态分析器（Static Analyzer）的关系是什么？
静态分析器是现代编译器的一部分。编译器由源代码生成中间表示之后，静态分析器对中间表示进行处理，编译器根据静态分析器的输出做出优化，或者报出错误与警告。

### 三地址码（3-Address Code, 3AC）是什么，它的常用形式有哪些？
一些常见的三地址码形式如下：
```
x = y bop z
x = uop y
x = y
goto L
if x goto L
if x rop y goto L
```
- `x`、`y`、`z`是变量的地址，地址可能是
  - 名字（Name），包括，
    - 变量（Variable）
    - 标签（Label）：用于指示程序位置，方便跳转指令的书写
  - 字面常量（Literal Constant），
  - 编译器生成的临时量（Compiler-Generated Temporary）；
- `bop`是双目操作符（Binary Operator），可以是算数运算符，也可以是逻辑运算符；
- `uop`是单目操作符（Unary Operator），可能是取负、按位取反或者类型转换；
- `L`是标签（Label），是标记程序位置的助记符，本质上还是地址；
- `rop`是关系运算符（Relational Operator），运算结果一般为布尔值。
- `goto`是无条件跳转，`if... goto`是条件跳转。

### 如何在中间表示（Intermediate Representation, IR）的基础上构建基块（Basic Block, BB）？
基块就是满足如下性质的最长指令序列：
- 程序的控制流只能从首指令进入；
- 程序的控制流只能从尾指令流出。

一个基块的领导者就是这个基块的首指令，整个程序中的领导者有如下3种：
- 整个程序的首指令；
- 跳转指令（包括条件跳转和无条件跳转）的目标指令；
- 跳转指令（包括条件跳转和无条件跳转）紧接着的下一条指令。

从前到后扫一遍，获得所有的Leader。再从前到后扫一遍，按Leader分段即可。

### 如何在基块的基础上构建控制流图（Control Flow Graph, CFG）?
程序控制流的产生来源于两个地方：
- 天然的顺序执行，这是计算系统天然存在的一种控制流；
- 跳转指令，这是人为设计添加的一种控制流。

扫一遍所有的基块：
- 如果一个基块的末尾是跳转，就从这个基块向跳转目标连一条边；
- 只要一个基块的末尾不是无条件跳转，就从这个基块向下一个基块连一条边。

## 数据流分析
### 定义可达性（Reaching Definitions）分析、活跃变量（Live Variables）分析和可用表达式（Avaliable Expressions）分析分别是什么含义？
- 我们称在程序点$p$处的一个定义$d$**到达（Reach）**了程序点$q$，如果存在一条从$p$到$q$的“路径”（控制流），在这条路径上，定义$d$未被**覆盖（Kill）**。称分析每个程序点处能够到达的定义的过程为**定义可达性分析（Reaching Definition Analysis）**。定义可达性分析可以应用于检测程序中可能存在的未被定义的变量。
- 在程序点$p$处
  - 某个变量$v$的变量值（Variable Value）可能在之后的某条控制流中被用到，我们就称变量$v$是程序点$p$处的**活变量（Live Variable）**
  - 否则，我们就称变量$v$为程序点$p$处的**死变量（Dead Variable）**。

  分析在各个程序点处所有的变量是死是活的分析，称为**活跃变量分析（Live Variable Analysis）**。通过活跃变量分析，可以知道某个变量当前所储存的值在之后有没有可能被用到。
- 我们称一个表达式（Expression）`x op y`在程序点$p$处是**可用的（Avaliable）**，如果：
  - **所有**的从程序入口到程序点$p$的路径都**必须**经过`x op y`表达式的评估（Evaluation），并且
  - 在最后一次`x op y`的评估之后，没有$x$或者$y$的重定义（Redefinition）。
  
  对于程序中每个程序点处的可用表达式的分析，我们称之为 可用表达式分析（Avaliable Expression Analysis） 。

### 上述三种数据流分析（Data Flow Analysis）有哪些不同点？又有什么相似的地方？

### 如何理解数据流分析的迭代算法？数据流分析的迭代算法为什么最后能够终止？

### 如何从函数的角度来看待数据流分析的迭代算法？

### 格和全格的定义是什么？

### 如何理解不动点定理？

### 怎样使用格来总结可能性分析与必然性分析？

### 迭代算法提供的解决方案与 MOP 相比而言精确度如何？

### 什么是常量传播（Constant Propagation）分析？

### 数据流分析的工作表算法（Worklist Algorithm）是什么？

## 过程间分析
### 如何通过类层级结构分析（Class Hierarchy Analysis, CHA）来构建调用图（Call Graph）？

### 如何理解过程间控制流图（Interprocedural Control-Flow Graph, ICFG）的概念？

### 如何理解过程间数据流分析（Interprocedural Data-Flow Analysis, IDFA）的概念？

### 如何进行过程间常量传播（Interprocedural Constant Propagation）分析？

## 指针分析
### 什么是指针分析（Pointer Analysis）？

### 如何理解指针分析的关键因素（Key Factors）？

### 我们在指针分析的过程中具体都分析些什么？

### 指针分析的规则（Pointer Analysis Rules）是什么？

### 如何理解指针流图（Pointer Flow Graph）？

### 指针分析算法（Pointer Analysis Algorithms）的基本过程是什么？

### 如何理解方法调用（Method Call）中指针分析的规则？

### 怎样理解过程间的指针分析算法（Inter-procedural Pointer Analysis Algorithm）？

### 即时调用图构建（On-the-fly Call Graph Construction）的含义是什么？

### 上下文敏感（Context Sensitivity, C.S.）是什么？

### 上下文敏感堆（C.S. Heap）是什么？

### 为什么 C.S. 和 C.S. Heap 能够提高分析精度？

### 上下文敏感的指针分析有哪些规则？

### 如何理解上下文敏感的指针分析算法（Algorithm for Context-sensitive Pointer Analysis）？

### 常见的上下文敏感性变体（Context Sensitivity Variants）有哪些？

### 常见的几种上下文变体之间的差别和联系是什么？

## 静态分析与安全
### 信息流安全（Information Flow Security）的概念是什么？

### 如何理解机密性（Confidentiality）与完整性（Integrity）？

### 什么是显式流（Explicit）和隐蔽信道（Covert Channels）？

### 如何使用污点分析（Taint Analysis）来检测不想要的信息流？

## 基于 Datalog 的程序分析
### Datalog 语言的基本语法和语义是什么？

### 如何用 Datalog 来实现指针分析？

### 如何用 Datalog 来实现污点分析？

## CFL 可达与 IFDS

### 什么是 CFL 可达（CFL-Reachability）？

### IFDS（Interprocedural Finite Distributive Subset Problem）的基本想法是什么？

### 怎样的问题可以用 IFDS 来解决？

## 完全性与近似完全性

### 近似完全性（Soundiness）的动机和概念是什么？

### 为什么 Java 反射（Reflection）和原生代码是难分析的？
