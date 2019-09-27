---
title: 常用的图像插值算法
date: 2019-09-27 14:18:30
tags: ['图像处理']
---

最近看了一篇随机纹理的算法，补充了下图像差值的基础知识。图像插值就是利用已知邻近像素点的灰度值（或rgb图像中的三色值）来产生未知像素点的灰度值，以便由原始图像再生出几何变换之后的图像。

# 常见算法

## 1. 最近邻插值法（Nearest Neighbour Interpolation）

这是最简单的一种插值方法，在待求象素的四邻象素中，将距离待求象素最近邻的像素灰度赋给待求象素。设为待求象素坐标(x+u,y+v) ，【注：x,y为整数， u，v为大于零小于1的小数】则待求象素灰度的值 f(x+u,y+v)为 ，选取距离插入的像素点（x+u, y+v)最近的一个像素点，用它的像素点的灰度值代替插入的像素点。

特点：最近邻插值法虽然计算量较小，但可能会造成插值生成的图像灰度上的不连续，在灰度变化的地方可能出现明显的锯齿状。

三种不同的空间距离：
a. 欧式距离：

![](/images/image-interpolation/1.webp)

b. D4距离（城市距离）

![](/images/image-interpolation/2.webp)

c. D8距离（棋盘距离）

![](/images/image-interpolation/3.webp)


一般采用欧式距离。

## 2. 双线性插值
双线性插值，顾名思义，在像素点矩阵上面，x和y两个方向的线性插值所得的结果。

对于一维线性插值：

![](/images/image-interpolation/4.png)

![](/images/image-interpolation/5.png)

对于二维图像：

![](/images/image-interpolation/6.png)


先在x方向上面线性插值，得到R2、R1像素值：

![](/images/image-interpolation/7.png)
                 

然后以R2,R1在y方向上面再次线性插值。


本质：根据4个近邻像素点的灰度值做2个方向共3次线性插值。

特点：双线性内插法的计算比最邻近点法复杂，计算量较大，但没有灰度不连续的缺点，结果基本令人满意。它具有低通滤波性质，使高频分量受损，图像轮廓可能会有一点模糊。

![](/images/image-interpolation/8.webp)


## 3. 双三次插值
在数值分析这个数学分支中，双三次插值是二维空间中最常用的插值方法。在这种方法中，函数f 在点 (x, y) 的值可以通过矩形网格中最近的十六个采样点的加权平均得到，在这里需要使用两个多项式插值三次函数，每个方向使用一个。两个方向加权相乘。

特点：三次运算可以得到更接近高分辨率图像的放大效果，但也导致了运算量急速增加。

![](/images/image-interpolation/9.png)


![](/images/image-interpolation/10.png)


# 附注：

1. 参考坐标系问题

matlab和opencv等以变换后的几何中心重合（并不是以左上角之类的其他点作为变换的原点）。此处应当注意。

2. BiCubic函数

![](/images/image-interpolation/11.png)


上文中a取-1

# 参考文献：

https://www.jianshu.com/p/8118e708b766

https://blog.csdn.net/tercel_zhang/article/details/70196057

https://blog.csdn.net/qq_34885184/article/details/79163991

https://www.jianshu.com/p/dc07ba10854c