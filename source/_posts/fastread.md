---
title: 快读
mathjax: true
toc: false
date: 2021-09-30 14:28:36
categories: algorithm
---
最近时常和同学一起刷笔试题, 顺带着开始学点儿OJ上常用的算法. 以往写OJ少, 都是些笔试题, 没卡过输入输出, 一直在用cin/cout. 最近学树状数组的时候tle了, 自己没想到cin/cout被卡了, 看完讨论a掉了.

这样想来, 做题就没必要cin/cout了, 可以直接上scanf/printf, 也不是很麻烦, 而且我觉得格式化字符串比流清晰.

然而有的题还会卡读数字, 如果输入不是很乱可以用快读. 思路就是手动读字母拼数字. 下面这段代码没有负数的时候可以把f去掉.

```c++
/// 普通快读
inline void Read(int &x) {
    x = 0; char c = getchar(); int f = 1;
    while (c < '0' || c > '9') { if (c == '-') f = -1; c = getchar(); }
    while (c >= '0' && c <= '9') { x = x * 10 + c - '0'; c = getchar(); }
    x *= f;
}
```

有的文章用位操作替换了 `x * 10 + c - '0'` 这个部分, 换成 `(x << 1) + (x << 3) + (c ^ 48)` , 我觉得意义不大. 如果需要更快, 可以看下文操快读, 手写一个buffer减少取stdin的次数.

```c++
/// 文操快读
const int MAX_SIZE = 1 << 15;
char *l, *r, buf[MAX_SIZE];
inline char Getchar() {
    if (l == r) {
        l = buf;
        r = l + fread(buf, 1, MAX_SIZE, stdin);
        if (l == r)
            return -1;
    }
    return *l++;
}
inline void Read(int & x) {
    x = 0; char c = Getchar(); int f = 1;
    while (c < '0' || c > '9') { if (c == '-') f = -1; c = Getchar(); }
    while (c >= '0' && c <= '9') { x = (x << 1) + (x << 3) + (c ^ 48); c = Getchar(); }
    x *= f;
}
```

输出也可以加速, 通过手动把数字拼成字符串, 或者结合fwrite的方式输出, 之后如果用到了再看.
