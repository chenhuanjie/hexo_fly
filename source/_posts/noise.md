---
title: 噪声
mathjax: true
toc: false
date: 2021-09-28 10:13:31
categories: shader
---
之前写博客喜欢排版, 浪费很多时间, 而且会导致热情消退之后懒得花时间写博客. 恰逢生活比较忙, 一段时间没写点什么了.

最近整理了博客页面, 用GLSL es3.0加了[这个页面](https://chenhuanjie.github.io/frag), 打算拿着色器写噪声, 用来展示效果, 很方便. 写的编辑框好像有bug, 建议还是编辑器写, 网页验证结果. 此外这玩意儿在手机上行不通, 能画, 但是结果不对, 不知道是不是精度问题.

### 1. 随机数, 白噪声

这篇文章我计划一点点更新, 把常见的几个噪声加进去. GLSL里需要手搓随机数, 常用的有这么几个hash方法.

```glsl
float hash11(float n){ return fract(sin(n) * 43758.5453123); }
float hash12(vec2 p) { return fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453); }
vec3 hash33(vec3 p) {
    p = mat3(127.1, 269.5, 113.5, 311.7, 183.3, 271.9, 74.7, 246.1, 124.6) * p;
    return fract(sin(p) * 43758.5453123) * 2. - 1.;
}
```

虽然静止的噪声看上去没啥问题, 但是当我在参数中掺上时间, 再画出来不难发现, 这两个函数都有比较明显的瑕疵, 肉眼看上去感觉图片总在往一个方向移动. 我找到了另一个使用位操作的噪声, [来源网址](http://amindforeverprogramming.blogspot.com/2013/07/random-floats-in-glsl-330.html).

```glsl
uint hash(uint x) {
    x += (x << 10u);
    x ^= (x >> 6u);
    x += (x << 3u);
    x ^= (x >> 11u);
    x += (x << 15u);
    return x;
}
float uintToFloat(uint x) {
    const uint mantissaMask = 0x007FFFFFu;
    const uint floatOne = 0x3F800000u;
    x &= mantissaMask;
    x |= floatOne;
    return uintBitsToFloat(x) - 1.0f;
}
uint hash(uvec2 v) { return hash(v.x ^ hash(v.y)); }
uint hash(uvec3 v) { return hash(v.x ^ hash(v.y) ^ hash(v.z)); }
uint hash(uvec4 v) { return hash(v.x ^ hash(v.y) ^ hash(v.z) ^ hash(v.w)); }
float random(float x) { return uintToFloat(hash(floatBitsToUint(x))); }
float random(vec2 v) { return uintToFloat(hash(floatBitsToUint(v))); }
float random(vec3 v) { return uintToFloat(hash(floatBitsToUint(v))); }
float random(vec4 v) { return uintToFloat(hash(floatBitsToUint(v))); }
```

[最终结果](https://chenhuanjie.github.io/frag#precision%20mediump%20float;%0Auniform%20float%20time;%0Auniform%20vec2%20resolution;%0Aout%20vec4%20fragColor;%0Auint%20hash(uint%20x)%20%7B%0A%20%20%20%20x%20+=%20(x%20%3C%3C%2010u);%0A%20%20%20%20x%20%5E=%20(x%20%3E%3E%206u);%0A%20%20%20%20x%20+=%20(x%20%3C%3C%203u);%0A%20%20%20%20x%20%5E=%20(x%20%3E%3E%2011u);%0A%20%20%20%20x%20+=%20(x%20%3C%3C%2015u);%0A%20%20%20%20return%20x;%0A%7D%0Afloat%20uintToFloat(uint%20x)%20%7B%0A%20%20%20%20const%20uint%20mantissaMask%20=%200x007FFFFFu;%0A%20%20%20%20const%20uint%20floatOne%20=%200x3F800000u;%0A%20%20%20%20x%20&=%20mantissaMask;%0A%20%20%20%20x%20%7C=%20floatOne;%0A%20%20%20%20return%20uintBitsToFloat(x)%20-%201.0f;%0A%7D%0Auint%20hash(uvec3%20v)%20%7B%20return%20hash(v.x%20%5E%20hash(v.y)%20%5E%20hash(v.z));%20%7D%0Afloat%20random(vec3%20v)%20%7B%20return%20uintToFloat(hash(floatBitsToUint(v)));%20%7D%0Avoid%20main()%20%7B%0A%20%20%20%20vec2%20position%20=%20gl_FragCoord.xy%20/%20resolution;%0A%20%20%20%20fragColor%20=%20vec4(vec3(random(vec3(position.x,%20position.y,%20time))),%201.0f);%0A%7D). 原文提到这个噪声均值在0.5附近, 而且无论我怎么变换time值, 结果上都没看到明显的特征. 感觉不错, 但到白噪声才刚刚开始, 就用这个随机数继续写其他噪声好了.

### 2. Perlin噪声

原始的Perlin噪声为了满足伪随机多次调用给出相同结果的特性, 用了一个shuffle数组. 在GLSL里写这玩意儿不方便, 而且会限制住坐标的范围, 我就直接用上一步写的hash了.

在空间中定义一个间距均等的网格, 为每个格点分配一个随机向量作为它的“梯度”, 对于方格内的点, 分别计算四个角点到该点的向量点乘其随机向量的积, 把四个角点上的噪声值按照点积的结果进行平滑插值, 得到每个点上的噪声值.

假设对于点 $ p(x,y) $ 有:
$$ x_0 = \lfloor x \rfloor, x_1 = \lceil x \rceil, \\ y_0 = \lfloor y \rfloor, y_1 = \lceil y \rceil ; $$
则与其相邻的四个角点可定义为:
$$ p_{00}(x_0, y_0), p_{01}(x_0, y_1), p_{10}(x_1, y_0), p_{11}(x_1, y_1) $$
设四个角点的随机向量为 $d_{00}, d_{01}, d_{10}, d_{11}$ . 则这四个角点对于目标点的噪声值的贡献度分别为:
$$ \begin{align}
c_{00} & = \overrightarrow{p_{00}p} \cdot \vec{d_{00}} \\
c_{01} & = \overrightarrow{p_{01}p} \cdot \vec{d_{01}} \\
c_{10} & = \overrightarrow{p_{10}p} \cdot \vec{d_{10}} \\
c_{11} & = \overrightarrow{p_{11}p} \cdot \vec{d_{11}}
\end{align} $$

对于平滑插值的函数, Perlin最早使用的是 $3t^2 + 2t^3$, 即smoothstep方法的曲线, 但是其导数有线性分量导致结果不够自然, 后来建议使用更加平滑的 $6t^5 - 15t^4 + 10t^3$ 进行插值.
$$ fade(t) = 6t^5 - 15t^4 + 10t^3 = ((6t - 15) t + 10) t^3 $$
则点 $p$ 的噪声值 $r$ 为:
$$ \begin{align}
r_{x0} = & fade(c_{00}, c_{10}, x) \\
r_{x1} = & fade(c_{10}, c_{11}, x) \\
r = & fade(r_{x0}, r_{x1}, y)
\end{align} $$

上面举的例子是二维的, 但这个方法不限制维数, 只是插值次数会随维数增大形成指数增长. 这里贴一个三维实现, 第三维用的时间.

```glsl
#define OCTAVES 4
precision mediump float;
uniform float time;
uniform vec2 resolution;
out vec4 fragColor;
uint hash(uint x) { x += (x << 10); x ^= (x >> 6); x += (x << 3); x ^= (x >> 11); x += (x << 15); return x; }
uint hash(uvec3 v) { return hash(v.x ^ hash(v.y ^ hash(v.z))); }
float parseFloat(uint x) { return uintBitsToFloat((x & 0x007FFFFFu) | 0x3F800000u) - 1.0f; }
float random(float x) { return parseFloat(hash(floatBitsToUint(x))); }
float random(vec3 v) { return parseFloat(hash(floatBitsToUint(v))); }
float fade(float t) { return ((6.0f * t - 15.0f) * t + 10.0f) * t * t * t; }
vec3 getVec(vec3 pos) {
    pos = mat3(127.1f, 269.5f, 113.5f, 311.7f, 183.3f, 271.9f, 74.7f, 246.1f, 124.6f) * pos;
    return normalize(vec3(random(pos.xyz), random(pos.yzx), random(pos.zxy)) * 2.0f - 1.0f);
}
float perlin(vec3 pos) {
    vec3 p0 = floor(pos), d = fract(pos), e = vec3(1.0f, 0.0f, 0.0f);
    float fx = fade(d.x), fy = fade(d.y), fz = fade(d.z);
    float g0 = dot(d - e.yyy, getVec(p0 + e.yyy)), g1 = dot(d - e.xyy, getVec(p0 + e.xyy)),
          g2 = dot(d - e.yxy, getVec(p0 + e.yxy)), g3 = dot(d - e.xxy, getVec(p0 + e.xxy)),
          g4 = dot(d - e.yyx, getVec(p0 + e.yyx)), g5 = dot(d - e.xyx, getVec(p0 + e.xyx)),
          g6 = dot(d - e.yxx, getVec(p0 + e.yxx)), g7 = dot(d - e.xxx, getVec(p0 + e.xxx));
    float result = mix(mix(mix(g0, g1, fx), mix(g2, g3, fx), fy),
                       mix(mix(g4, g5, fx), mix(g6, g7, fx), fy), fz);
    return result + 0.5f;
}
void main() {
    vec3 pos = vec3(gl_FragCoord.xy / resolution.y, time * 0.2f) * 3.0f;
    float noise = 0.0f, multiply = 1.0f, scale = 0.0f;
    for (int i = 0; i < OCTAVES; i++) {
        scale += multiply;
        noise += multiply * perlin(pos);
        pos *= 2.0f;
        multiply *= 0.5f;
    }
    fragColor = vec4(vec3(noise / scale), 1.0f);
}
```

[效果预览在这里](https://chenhuanjie.github.io/frag##define%20OCTAVES%204%0Aprecision%20mediump%20float;%0Auniform%20float%20time;%0Auniform%20vec2%20resolution;%0Aout%20vec4%20fragColor;%0Auint%20hash(uint%20x)%20%7B%20x%20+=%20(x%20%3C%3C%2010);%20x%20%5E=%20(x%20%3E%3E%206);%20x%20+=%20(x%20%3C%3C%203);%20x%20%5E=%20(x%20%3E%3E%2011);%20x%20+=%20(x%20%3C%3C%2015);%20return%20x;%20%7D%0Auint%20hash(uvec3%20v)%20%7B%20return%20hash(v.x%20%5E%20hash(v.y%20%5E%20hash(v.z)));%20%7D%0Afloat%20parseFloat(uint%20x)%20%7B%20return%20uintBitsToFloat((x%20&%200x007FFFFFu)%20%7C%200x3F800000u)%20-%201.0f;%20%7D%0Afloat%20random(float%20x)%20%7B%20return%20parseFloat(hash(floatBitsToUint(x)));%20%7D%0Afloat%20random(vec3%20v)%20%7B%20return%20parseFloat(hash(floatBitsToUint(v)));%20%7D%0Afloat%20fade(float%20t)%20%7B%20return%20((6.0f%20*%20t%20-%2015.0f)%20*%20t%20+%2010.0f)%20*%20t%20*%20t%20*%20t;%20%7D%0Avec3%20getVec(vec3%20pos)%20%7B%0A%20%20%20%20pos%20=%20mat3(127.1f,%20269.5f,%20113.5f,%20311.7f,%20183.3f,%20271.9f,%2074.7f,%20246.1f,%20124.6f)%20*%20pos;%0A%20%20%20%20return%20normalize(vec3(random(pos.xyz),%20random(pos.yzx),%20random(pos.zxy))%20*%202.0f%20-%201.0f);%0A%7D%0Afloat%20perlin(vec3%20pos)%20%7B%0A%20%20%20%20vec3%20p0%20=%20floor(pos),%20d%20=%20fract(pos),%20e%20=%20vec3(1.0f,%200.0f,%200.0f);%0A%20%20%20%20float%20fx%20=%20fade(d.x),%20fy%20=%20fade(d.y),%20fz%20=%20fade(d.z);%0A%20%20%20%20float%20g0%20=%20dot(d%20-%20e.yyy,%20getVec(p0%20+%20e.yyy)),%20g1%20=%20dot(d%20-%20e.xyy,%20getVec(p0%20+%20e.xyy)),%0A%20%20%20%20%20%20%20%20%20%20g2%20=%20dot(d%20-%20e.yxy,%20getVec(p0%20+%20e.yxy)),%20g3%20=%20dot(d%20-%20e.xxy,%20getVec(p0%20+%20e.xxy)),%0A%20%20%20%20%20%20%20%20%20%20g4%20=%20dot(d%20-%20e.yyx,%20getVec(p0%20+%20e.yyx)),%20g5%20=%20dot(d%20-%20e.xyx,%20getVec(p0%20+%20e.xyx)),%0A%20%20%20%20%20%20%20%20%20%20g6%20=%20dot(d%20-%20e.yxx,%20getVec(p0%20+%20e.yxx)),%20g7%20=%20dot(d%20-%20e.xxx,%20getVec(p0%20+%20e.xxx));%0A%20%20%20%20float%20result%20=%20mix(mix(mix(g0,%20g1,%20fx),%20mix(g2,%20g3,%20fx),%20fy),%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20mix(mix(g4,%20g5,%20fx),%20mix(g6,%20g7,%20fx),%20fy),%20fz);%0A%20%20%20%20return%20result%20+%200.5f;%0A%7D%0Avoid%20main()%20%7B%0A%20%20%20%20vec3%20pos%20=%20vec3(gl_FragCoord.xy%20/%20resolution.y,%20time%20*%200.2f)%20*%203.0f;%0A%20%20%20%20float%20noise%20=%200.0f,%20multiply%20=%201.0f,%20scale%20=%200.0f;%0A%20%20%20%20for%20(int%20i%20=%200;%20i%20%3C%20OCTAVES;%20i++)%20%7B%0A%20%20%20%20%20%20%20%20scale%20+=%20multiply;%0A%20%20%20%20%20%20%20%20noise%20+=%20multiply%20*%20perlin(pos);%0A%20%20%20%20%20%20%20%20pos%20*=%202.0f;%0A%20%20%20%20%20%20%20%20multiply%20*=%200.5f;%0A%20%20%20%20%7D%0A%20%20%20%20fragColor%20=%20vec4(vec3(noise%20/%20scale),%201.0f);%0A%7D).
关于噪声函数结果值的归一化, 在一维情况下考虑输入在 $[0, 1]$ 上的极端情况可以发现, 结果的噪声值在范围 $[-\frac{1}{2}, \frac{1}{2}]$ 之间, 因此直接 `+0.5f` 使结果落在范围 $[0, 1]$ 上. 这里顺便加了分形, 每次叠加一个频率翻倍强度减半的噪声, 以达到添加细节模拟自相似的效果.
