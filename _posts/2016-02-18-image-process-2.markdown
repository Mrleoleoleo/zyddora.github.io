---
layout:     post
title:      "图像旋转优化（二）——以Android中YUV422I旋转算法为例"
subtitle:   ""
date:       2016-02-18 18:43:00
author:     "Orchid"
header-img: "img/post-bg-img.jpg"
tags:
    - 图像处理
    - C/C++
---
<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>

> 在上一篇博文里介绍了[图像处理初探（一）——图像转换基础及OpenCV应用](http://zyddora.github.io/2016/02/05/image-process-1/)，本篇在此基础上重点谈图像旋转算法的原理与实现。本文以Android中常见的YUV422I（YUY2）格式为例。

### Catalog

1. [通用YUV422I旋转90/270算法](#yuv422i90270)
2. [通用YUV422I旋转0/180算法](#yuv422i0180)
3. [完善算法以支持不规则尺寸图像的旋转](#section-2)
4. [更加省时的图像旋转](#section-3)

## 通用YUV422I旋转90/270算法

### **解决思路**

**实现图像旋转算法的关键在于理解清楚：旋转前后图像格式排列规则之间的联系，这很重要。**初步考虑是，旋转90°或270°涉及相邻行列的像素点（最少2*2个像素点），将Y元素转置而UV元素进行适当地转换，UV的转换是重点。强烈建议画出前后的图像排列格式，再找转换关系。

### **YUV422I旋转90°示意**

![img](/img/in-post/rot90.jpg)

观察旋转前后可得以下几点信息：

- 旋转前的奇数行（以1为初始）中的UV信息都无用；
- 旋转后的偶数行（以1位初始）中的UV信息都是原图中没有的，需要推算；
  * 比如，图中的U10、V10需从Y10、U9、V9转换得到，通常思路为：(Y10, U9, V9) -> (R9, G9, B9) -> (U10, V10)
- 所有的Y信息都进行了转置且都有用（亮度信息必然有用）；
- `4*2`个YUV信息为旋转算法处理的最小单位；
- 旋转前图片格式为`(width * 2) * height`，旋转后为`(heigh * 2) * width`；

旋转90°的C程序框架：

```cpp
const unsigned char *src_y = src; /* dst_1_1指向旋转前1-1处的指针 */
unsigned char *dst_y = dst_1_1; /* dst_1_1指向旋转后1-1处的指针 */

for (int i = 0; i < height / 2; ++i) { /* 假设一次外循环处理2行YUYV信息 */
  src = src_y;
  dst = dst_y;
  for (int j = 0; j < doublewidth / 4) { /* 假设一次处理每行相邻4个YUYV信息 */
    ... // 移动Y信息
    ... // 计算新的UV信息

    src += 4; /* 更新内循环的指针 */
    dst += doubleheight * 2;
  }
  src_y += doublewidth * 2; /* 更新外循环的指针 */
  dst_y -= 4;
}
```

### **YUV422I旋转270°示意**

![img](/img/in-post/rot270.jpg)

与旋转90°情况的不同在于：

- 旋转前的偶数行（以1为初始）中的UV信息都无用；
- 旋转后的奇数行（以1位初始）中的UV信息都是原图中没有的，需要推算；
  * 比如，图中的U24、V24需从Y24、U23、V23转换得到，通常思路为：(Y24, U23, V23) -> (R24, G24, B24) -> (U24, V24)

旋转270°的C程序框架与90°类似，此处不赘述。

---

## 通用YUV422I旋转0/180算法

### **解决思路**

旋转0°、180°较90°、270°更简单，仅涉及行上的像素点的转换（最少2*1个像素点）。

### **YUV422I旋转180°示意**

![img](/img/in-post/rot180.jpg)

观察可发现：

- 旋转前的所有YUV信息都有用；
- 旋转后的每行Y信息逆置，UV信息则需要推算，推算思路与上述类似；
- 旋转前后的图像规格不变，仍为`(width * 2) * height`；

旋转0°的本质：

- 图像数据的拷贝；

旋转180°的C程序框架：

```cpp
const unsigned char *src_y = src; /* dst_1_1指向旋转前1-1处的指针 */
unsigned char *dst_y = dst_1_1; /* dst_1_1指向旋转后1-1处的指针 */

for (int i = 0; i < height; ++i) {
  src = src_y;
  dst = dst_y;
  for (int j = 0; j < doublewidth / 4) { /* 假设一次处理每行相邻4个YUYV信息 */
    // 移动Y信息
    // 计算新的UV信息

    src += 4; /* 更新内循环的指针 */
    dst -= 4;
  }
  src_y += doublewidth; /* 更新外循环的指针 */
  dst_y -= doublewidth;
}
```

---

## 完善算法以支持不规则尺寸图像的旋转

上述算法仅是一个基础版本，对于不规则的行列数的图像是不支持的，目前基本算法的局限：

1. 若上述算法一次处理每行多个YUYV信息块（即大于4，如8、16、32等），则存在每行的尾数；
2. 旋转90°/270°算法仅支持高度height为2的倍数；

处理问题1的思路：

可以确定的是YUV422I格式的图像的宽度一定是4的倍数（以YUYV为一组信息），因此不论一次处理每行YUYV块的个数是多少（4、8、16、32等），剩余的YUYV信息一定也是4的整数倍，故对尾数按照以4为行上一组的单位来处理。

处理问题2的思路：

YUV422I格式图像旋转90°/270°，若原图高度为奇数，则对旋转后的宽度造成影响。故此情况，需要在转换过程中进行补齐，可以直接存入255代替缺失的高度。

---

## 更加省时的图像旋转

> 一般地，图像呈现局部相关性，上下左右相邻的像素值相关度很高。

可以发现，旋转算法存在着不少用于计算新UV信息的计算量。但是对于性能优化来说，**有时节省资源与编码效果是互补的关系，以一方的牺牲来换取另一方的提升**，其实这个原则在很多地方都适用，重点在于程序要解决的主要矛盾是什么。那么，**如果对图片质量的需求并不高，就将邻近的UV信息直接复制，以节省掉计算UV的时间，将会带来很大的增益**。

### 推导证明“直接复制UV”的可行性

$$
256\cdot \begin{bmatrix} R\\ G\\ B\end{bmatrix} = 
\begin{bmatrix}298 & 0 & 409\\ 298 & -100 & -208\\ 298 & 516 & 0\end{bmatrix}
\begin{bmatrix}Y-16\\ U-128\\ V-128\end{bmatrix}\\ \\ \\

256^{2}\begin{bmatrix}Y\\ U\\ V\end{bmatrix} = 
256\begin{bmatrix}66 & 129 & 25\\ -38 & -74 & 112\\ 112 & -94 & -18\end{bmatrix}
\begin{bmatrix}298 & 0 & 409\\ 298 & -100 & -208\\ 298 & 516 & 0\end{bmatrix}
\begin{bmatrix}R\\ G\\ B\end{bmatrix} + 256^{2}
\begin{bmatrix}16\\ 128\\ 128\end{bmatrix}\\ \\ \\

256\begin{bmatrix}Y\\ U\\ V\end{bmatrix} = 
\begin{bmatrix}256.09 & 0 & 0.63\\ 0 & 254.66 & -0.59\\ 0 & 0.44 & 255.31\end{bmatrix}
\begin{bmatrix}Y-16\\ U-128\\ V-128\end{bmatrix} + 
256\begin{bmatrix}16\\ 128\\ 128\end{bmatrix}
$$

从推导的结果很容易发现，最终式子的系数矩阵主对角线绝对占优，因此YUV->RGB->YUV整个过程更像是YUV的直接“复制”。

下图是用“直接复制UV”的方法做的图像旋转的实验。

![img](/img/in-post/cont1.JPG)
![img](/img/in-post/cont2.JPG)

可以看到旋转90°后的图像除了高频部分有些杂乱模糊外（注意图中的几个圆圈字母边缘有些模糊，由于是笔者截图后的结果，所以计算机可能做了些处理，减小了前后的差异性），其他表现都不错。这证明直接复制UV的方法某种意义上也是可行的。

---