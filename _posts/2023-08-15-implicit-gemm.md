---
layout: post
title:  卷积转化为矩阵乘法
date:   2023-08-15 09:00:00 +0800
categories: cuda algorithm paper
---

## 卷积

| 参数  | 含义          |
| ----- | ------------- |
| $N$     | batch大小     |
| $C$     | 输入通道数量  |
| $H$     | 输入的高      |
| $W$     | 输入的宽      |
| $K$     | 输出通道数量  |
| $R$     | 卷积核高度    |
| $S$     | 卷积核宽度    |
| $u$     | 垂直步幅      |
| $v$     | 水平步幅      |
| $pad_h$ | padding的高度 |
| $pad_w$ | padding的宽度 |

输入矩阵$D \in \mathbb{R}^{NCHW}$, 卷积核$F \in \mathbb{R}^{KCRS}$. 输出矩阵$O \in \mathbb{R}^{NKPQ}$. 
其中 $P = \lceil \frac{H-R+1+2pad_h}{u} \rceil$, $Q = \lceil \frac{W-S+1+2pad_w}{v} \rceil$.

把边缘零扩展的 $D$ 记作 $D_0$.

$$O[n, k, p, q] = \sum_{c=0}^{C-1} \sum_{r=0}^{R-1} \sum_{s=0}^{S-1} F[k, c, r, s] \cdot D_0[n, c, pu + R - r - 1 - pad_h, qv + S - s - 1 - pad_w]$$

## 矩阵乘法

$$C = AB$$

其中 $A \in \mathbb{R}^{mk}$, $B \in \mathbb{R}^{kn}$, $C \in \mathbb{R}^{mn}$.

$$C[i, j] = \sum_{g=0}^{k-1} A[i, g] \cdot B[g, j]$$

## 卷积到矩阵乘法

观察矩阵乘法定义中 $C[i, j]$ 的表达式, $C$ 和 $A$ 下标的公用部分是 $i$, $C$ 和 $B$ 下标的公用部分是 $j$, $A$ 和 $B$ 下标的公用部分是 $g$. 这三个变量互不相关. 我们可以自然地想到, 卷积的表达式中是否存在类似的公用部分呢? 

- $O$和$F$的公用部分是$k$,
- $O$和$D$的公用部分是$n$, $p$, $q$,
- $F$和$D$的公用部分是$c$, $r$, $s$.

其余都是常量. 这样, 我们找到了沟通卷积与矩阵乘法的桥梁. $k$ 相当于矩阵乘法中的 $i$, 三元组 $<n, p, q>$ 相当于矩阵乘法中的$j$, 三元组 $<c, r, s>$ 相当于矩阵乘法中的 $g$. 对$D$, $F$, $O$的元素进行重新排列: 

$$O[k][n, p, q] = \sum_{<c, r, s>} F[k][c, r, s] \cdot D_0[c, pu + R - r - 1 - pad_h, qv + S - s - 1 - pad_w][n]$$

$O$和$F$ 已经变成了我们想要的形式, 而 $D$ 矩阵还需要进一步的处理. 
观察 $O$ 和 $F$, 
- $O$的第一维是 $k$, 第二维是$<n, p, q>$, 相当于把每一个输出通道对应的 $N$ 个 batch 拍扁成一维数组, 共 $K$ 个一维数组
- $F$的第一维也是 $k$, 第二位是 $<c, r, s>$, 如图所示, 相当于把每一个输出通道对应的 $C$ 个 $R \times S$ 的卷积核拍扁成一维数组 (图中的一行), 共 $K$ 个一维数组

$D$ 的第一维应当是 $<c, r, s>$, 第二维应当是 $<n, p, q>$, 大小应当是 $CRS \times NPQ$. 现在 $D$ 的大小是 $NCHW$, 因此需要对 $D$ 中的元素进行复制，重新排列. 

由于 $F$ 的一行对应一个输出通道的 $C$ 个卷积核, 因此 $D$ 的一列对应的是一个感受野. 
感受野有$C$个通道, 每个通道是 $R \times S$ 的矩阵, 这个感受野会被拍扁成 $D$ 的一列, 长度为 $C \times R \times S$. 
$O$ 的第二维是 $<n, p, q>$, 一行对应了一个输出通道, 这个通道里的每个元素都是左侧的一组卷积核 (一行) 与上方的一组感受野 (一列) 作点积运算得到的. 

![test](https://github.com/yu-yake2002/yu-yake2002.github.io/raw/main/pictures/implicit-gemm.png) 

## 总结

把卷积核 $F$ 重构成一个 $K \times CRS$ 的矩阵 $F_m$. 
把输入矩阵 $D$ 复制成一个 $CRS \times NPQ$ 的矩阵 $D_m$. 
二者相乘, 得到 $K \times NPQ$ 的矩阵 $O_m$. 

