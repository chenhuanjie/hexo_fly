---
title: Dither Fade效果学习
date: 2021-01-11 10:27:45
categories: shader
---

前几天看了Dithering相关的一些内容, 没有深入了解, 但是感觉值得记录一下.

最初是因为在Unity的Amplify Shader Editor这个包 (很好的工具!) 的例子中看到了一个效果, 名字叫做Dithering Fade. 顺带一提这个效果以前被我误以为是screen-door透明, 其实不是, 透明的效果和这个还是差很多的. 抖出来的透明效果在性能上快得很, 而且随着像素密度的升高效果也很不错, 所以比较适合手机环境.

Amplify Shader Editor中提供了两版实现, 一种是使用Bayer抖动表, 另一种则是使用Blue Noise Sampling做的.

# 什么是Dithering?

> 就好像哪怕我只有0度的水和100度的水，只要调整好比例，理论上我也能混合出0度到100度之间任何温度的水。[3]

抖动(*dither*), 是指一种特意引入的噪声, 用于替代简单量化产生的误差. 不仅出现在图形处理中色彩表示精度不足的情况, 在音频处理中也常常用于构造一个平坦底噪来替代简单量化产生的谐波失真.

**但是!** 抖动并不能解决图像中的失真现象, 它通过引入随机变量的方式来使变化更加平滑, 这样的效果是因为人的眼睛倾向于忽略随机噪声. 这就好比使用黑白二值表示灰度图像, 只需在更黑的地方增加黑点的数量, 在较亮的部位减少黑点的数量就可以. 这并没有改变图像中只有黑白二值的规定, 但是肉眼却能看出不同的颜色, 这个方法也被称为*图案法*[1].

比方说现在有这样一张8位灰度图:

![0到255渐变](dither_example_0.png)

考虑一个极端情况, 将这张图的位深缩减为1位, 如果采用四舍五入的方式决定颜色就会发生失真, 变成下面这个样子:

![0到1, 或许也可称之为渐变](dither_example_1.png)

而实际上在左半侧图像内将1\~127的值处理为0, 或是在右半侧图像中将128\~254处理为255都是存在误差的, 因此要更好地表示出颜色连续的变化, 可以考虑将误差扩散到附近的点, 改变他们的值以使误差的分布更加均匀. 那么问题来了, 改变哪一个点的颜色呢? 如果随机地为点加一个颜色的偏差值, 看上去会是这个样子:

![随机抖动的结果, 一定程度上还原出了原始图像](dither_example_2.png)

ok, 这个图已经有点那个意思了, 通过加入一个白噪声把简单量化的误差遮过去了, 但是可能最终图片和原始图片并不相像, 这一点通过取一个高斯模糊就能看出.

![实际上与原始图像相比多了许多没有的特征, 比如偏白色的区域内有黑色聚集](dither_example_3.png)

# 有序抖动与Bayer抖动表

为了使变化看上去更加均匀, 我们可以使用一个固定的矩阵与图像中的内容做比较, 当图像亮度大于矩阵时画一个白点儿, 反之则画一个黑点儿, 这样便可以在考虑到误差的情况下较为均匀地表示一个范围内的变化与误差. 现在我们称这种方法为有序抖动(*ordered dithering*).

Limb于1969年提出了一种标准图案设计的算法, 如下定义[1]:

$$ M_1 = \begin{bmatrix} 0 & 2 \\ 3 & 1 \end{bmatrix} $$

$$ M_{n+1} = \begin{bmatrix} 4M_n & 4M_n + 2U \\ 4M_n + 3U & 4M_n + U \end{bmatrix} $$

特殊地, M3矩阵被称为Bayer有序抖动矩阵(*Bayer Ordered Dither Matrix*), 也成Bayer抖动表, 内容如下[2]:

$$ M_3 = \begin{bmatrix} 0 & 48 & 12 & 60 & 3 & 51 & 15 & 63 \\ 32 & 16 & 44 & 28 & 35 & 19 & 47 & 31 \\
8 & 56 & 4 & 52 & 11 & 59 & 7 & 55 \\ 40 & 24 & 36 & 20 & 43 & 27 & 39 & 23 \\ 2 & 50 & 14 & 62 & 1 & 49 & 13 & 61 \\
34 & 18 & 46 & 30 & 33 & 17 & 45 & 29 \\ 10 & 58 & 6 & 54 & 9 & 57 & 5 & 53 \\ 42 & 26 & 38 & 22 & 41 & 25 & 37 & 21 \end{bmatrix} $$

在上面灰度图像转换为黑白二值图像的过程中使用该方法进行抖动, 将会得到这样一个结果:

![Bayer抖动结果](dither_example_4.png)

这效果眼熟至极, 早年间的各路游戏机屏幕上常常会出现这种充斥着叉叉和对角线的显示效果, 就是为了提高显示的效果.

# Floyd-Steinberg算法

但是上面给出的效果“并不好看”, 实际上在抖动的过程中由于矩阵内容的特殊性, 引入了许多原图像中并不存在的特征, 图案法产生的图案化非常明显, 而且并不能很好地显示颜色误差过低的情况. 另一个更好的方法是将误差传递到相邻的像素, 并且累积下来, Floyd-Steinberg算法采用的就是这个方法.

设原图像颜色显示误差为e, 则分别将(3/8)e加到右方和下方的像素颜色上, 并将(1/4)e加到右下方像素的颜色上. 使用这种抖动方法绘制的图片像这样:

![Floyd-Steinberg算法抖动结果](dither_example_5.png)

这张图与上面的结果相比是看不出什么图案的, 也就是说, 它更加接近于原图像了.

# 使用蓝噪声(*Blue Noise*)进行抖动

学习过OpenGL的同学可能有所了解, 自带的GL_DITHER是默认开启的, 但当我们使用8位色去渲染物体时仍然会出现轻微的colour banding artifacts, 这是因为多数OpenGL版本中对于GL_DITHER只做了个空实现, 没有加入抖动效果[4]. 有时需要我们手动加入抖动效果, 那么问题就出现了, Bayer抖动表会引入图案化, 而Floyd-Steinberg算法的代价高, 因此可以传统艺能再现, 使用一个噪声来控制抖动.

这个噪声需要一些良好的特性, 如果直接生成一个白噪声(*White Noise*), 你会发现抖动出来的图案将会具备一些“大的结构”, 因为白噪声在高频和低频的功率密度是个常数, 过强的低频噪声会导致误差分布得不均匀. 此时我们可以尝试弱化低频噪声, 使用一个蓝噪声来进行抖动, 蓝噪声的功率密度会随频率升高而升高[6], 因此使用一个蓝噪声控制抖动可以使误差分布得更加均匀.

用shader写出一个蓝噪声非常复杂, 而且没啥意义, 因为可以使用一个纹理代替这一计算过程, 只需要使用取得的纹理值代替Bayer表中取出的内容即可. 关于蓝噪声纹理的计算过程, 感兴趣的同学可以读下[Free blue noise textures](http://momentsingraphics.de/BlueNoise.html)[5], 其中介绍得比较详细, 包括在什么情况下会遇到色带(*Color Banding*), 以及如何使用抖动修复色带问题.

# 实战一个DitherFade

用的Unity, 没怎么写过这个东西(菜得很), 稍微记一下遇到了哪些问题:

1. 需要注意View Space下视线方向是-z;
2. _ProjectionParams可以获得投影相关的信息, 比如这里用到的_ProjectionParams.y就是近平面距离;
3. 屏幕坐标用ComputeScreenPos算, 参数是裁剪空间下的顶点坐标, 需要注意这个函数的输出会有w分量, 需要除以自身的w分量;
4. 不要忘记写 ```UNITY_INITIALIZE_OUTPUT``` ;
5. 最终报出下面这个错来, 没弄明白怎么解决这个问题, 索性把 ```#pragma target 3.0``` 换成 ```#pragma target 4.0``` 了.

```log
Shader error in 'Test/DitherFade': Too many texture interpolators would be used for ForwardBase pass (11 out of max 10) at line 16.
```

实现的效果看上去像这个样子, 代码见文章末尾:

![DitherFade实现效果, 能看出来距离屏幕近的部分透明程度较高, 而胶囊体底端由于距离屏幕较远没有透明](dither_fade_result.png)

关于这里我有一个想法, 在玩游戏的时候发现游戏内角色透明时不会因物体不同位置的深度产生差异, 也就是说整个物体的透明度感觉是一致的. 因此猜测为了节约对屏幕空间坐标和片段的深度值的差值计算, 可以采用cpu计算距离, 并使用uniform的方式传入距离用于显示, 兴许对于需要节约资源的移动端有不错的效果吧.

# 参考 & 感谢

[1] [抖动算法小议1](https://blog.csdn.net/coolbacon/article/details/4041988)

[2] [Ordered dithering - Wikipedia](https://en.wikipedia.org/wiki/Ordered_dithering)

[3] [什么是抖色Dithering?——节选自《高兴说显示进阶篇之三》](https://zhuanlan.zhihu.com/p/33637225)

[4] [OpenGL gradient “banding” artifacts](https://stackoverflow.com/questions/16005952/opengl-gradient-banding-artifacts)

[5] [Free blue noise textures](http://momentsingraphics.de/BlueNoise.html)

[6] [有色噪声_百度百科](https://baike.baidu.com/item/%E6%9C%89%E8%89%B2%E5%99%AA%E5%A3%B0)

[7] [AD/DA 破解数字信号的玄学 Digital Show and Tell](https://www.bilibili.com/video/BV1k4411J7Fi)

# 附录: 代码

这部分内容是拆开来的, 需要把相关的部分拼起来运行. 顺带一提图片是用Image.save保存的.

1. 图是用pillow画的

```python
from PIL import Image
from PIL.ImageFilter import GaussianBlur
import random
```

2. Bayer矩阵定义

```python
BAYER_DITHER_MATRIX = [
     0, 48, 12, 60,  3, 51, 15, 63,
    32, 16, 44, 28, 35, 19, 47, 31,
     8, 56,  4, 52, 11, 59,  7, 55,
    40, 24, 36, 20, 43, 27, 39, 23,
     2, 50, 14, 62,  1, 49, 13, 61,
    34, 18, 46, 30, 33, 17, 45, 29,
    10, 58,  6, 54,  9, 57,  5, 53,
    42, 26, 38, 22, 41, 25, 37, 21,
]

def get_dither(x, y):
    return BAYER_DITHER_MATRIX[(x % 8) * 8 + (y % 8)]
```

3. 白噪声抖动(原图/高斯模糊)与Bayer有序抖动附图

```python
img_white = Image.new('RGB', (512, 64))
img_bayer = Image.new('RGB', (512, 64))
for i in range(512):
    for j in range(64):
        c0 = i * 256 / 512  # expected color
        mask = int(random.random() * 255)
        c = 0 if c0 <= mask else 255
        img_white.putpixel((i, j), (c, c, c))
        mask = get_dither(i, j) * 4
        c = 0 if c0 <= mask else 255
        img_bayer.putpixel((i, j), (c, c, c))

img_blurred = img_white.filter(GaussianBlur(radius=3))
img_white.show()
img_blurred.show()
img_bayer.show()
```

4. Floyd-Steinberg抖动算法附图

```python
img = Image.new('RGB', (512, 256))
src = [[(i + j) / 2 for j in range(256)] for i in range(256)]
for i in range(256):  # column
    for j in range(256):  # row
        # left side of original image
        c0 = (i + j) / 2
        img.putpixel((i, j), (c0, c0, c0))
        # right side of dithered image
        c = 0 if src[i][j] < 127.5 else 255
        e = src[i][j] - c1
        if i + 1 < len(src):
            src[i+1][j] += e * 0.375
        if j + 1 < len(src):
            src[i][j+1] += e * 0.375
        if i + 1 < len(src) and j + 1 < len(src):
            src[i+1][j+1] += e * 0.25
        img.putpixel((i + 256, j), (c, c, c))

img.show()
```

5. DitherFade Shader附图(代码类型随便选了个glsl, 为了语法高亮)

```glsl
Shader "Test/DitherFade"
{
    Properties
    {
        _Color ("Color", Color) = (1,1,1,1)
        _MainTex ("Albedo (RGB)", 2D) = "white" {}
        _Glossiness ("Smoothness", Range(0,1)) = 0.5
        _Metallic ("Metallic", Range(0,1)) = 0.0
        _BeginFade ("Begin Fade Distance", Float) = 0.0
        _EndFade ("End Fade Distance", Float) = 1.0
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        LOD 200
        CGPROGRAM
        #pragma surface surf Standard fullforwardshadows vertex:vert
        #pragma target 4.0
        struct Input
        {
            fixed2 uv_MainTex;
            fixed distance;
            fixed4 screenPosition;
        };
        uniform sampler2D _MainTex;
        uniform half _Glossiness;
        uniform half _Metallic;
        uniform fixed4 _Color;
        uniform float _BeginFade;
        uniform float _EndFade;
        inline float DitherMatrix(int x, int y)
        {
            const float dm[ 64 ] = {
                 1, 49, 13, 61,  4, 52, 16, 64,
                33, 17, 45, 29, 36, 20, 48, 32,
                 9, 57,  5, 53, 12, 60,  8, 56,
                41, 25, 37, 21, 44, 28, 40, 24,
                 3, 51, 15, 63,  2, 50, 14, 62,
                35, 19, 47, 31, 34, 18, 46, 30,
                11, 59,  7, 55, 10, 58,  6, 54,
                43, 27, 39, 23, 42, 26, 38, 22};
            return dm[y * 8 + x] / 64;
        }
        void vert(inout appdata_full v, out Input o)
        {
            UNITY_INITIALIZE_OUTPUT(Input, o);
            o.distance = -UnityObjectToViewPos(v.vertex).z;
            o.screenPosition = ComputeScreenPos(UnityObjectToClipPos(v.vertex));
        }
        void surf(Input i, inout SurfaceOutputStandard o)
        {
            fixed4 c = tex2D(_MainTex, i.uv_MainTex) * _Color;
            o.Albedo = c.rgb;
            o.Metallic = _Metallic;
            o.Smoothness = _Glossiness;
            o.Alpha = c.a;
            float4 sp = i.screenPosition / i.screenPosition.w;
            sp.xy = sp.xy * _ScreenParams.xy;
            float msk = DitherMatrix(fmod(sp.x, 8), fmod(sp.y, 8));
            clip((i.distance - _ProjectionParams.y - _BeginFade) / (_ProjectionParams.y + _EndFade) - msk);
        }
        ENDCG
    }
    FallBack "Diffuse"
}
```

