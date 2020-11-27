---
title: Cocos2dx 4.x 项目工程构建
---

Cocos2dx从4.0版本开始改用cmake, 删掉了原有的各个平台的工程文件, 但是没关系, 为了愉快地使用IDE进行编译, 可以使用cmake来构建工程文件供IDE使用.

首先这篇文章中的项目文件是cocos new生成的, 前面的过程全部略过了, 从new得到的目录开始进行项目构建. 其次这里用的项目是个lua项目, 但是实际上都一样, 没有区别的.

----

### <font color=red>20201127更新:</font>

实际上[官方教程](https://docs.cocos.com/cocos2d-x/manual/zh/installation/)有留方法, 就是用创建工程的那个脚本, 使用方法:

```shell
mkdir build
cd build
cocos run --proj-dir .. -p [mac|win32|android|linux|ios]
```

其实就是直接跑cocos run, 指定下项目路径, 指定下platform. 注意-p的值不可以是windows.

此外不论如何挣扎, 添加新的代码文件或者资源文件之后都需要重新运行一下, 这很蛋疼.

----

# Windows上构建Visual Studio项目

先创建个目录用来装项目工程(powershell)

```shell
mkdir win32-build && cd win32-build;
```

然后用cmake进行构建

```shell
cmake .. -G "Visual Studio 16 2019" -Tv141 -A win32
```

上面这个命令的-G参数要看咱用的是什么IDE, 比如: (8之前的官方文档上写着removed, 可能是不支持了)

```shell
-G "Visual Studio 9 2008"
-G "Visual Studio 10 2010"
-G "Visual Studio 11 2012"
-G "Visual Studio 12 2013"
-G "Visual Studio 14 2015"
-G "Visual Studio 15 2017"
-G "Visual Studio 16 2019"
```

然后-A参数指定编译的目标平台, 也就是要看咱运行的环境(原文叫target platform, architecture), 比如

```shell
-A Win32
-A x64
-A ARM
-A ARM64
```

运行成功之后在win32-build路径下会生成对应的解决方案项目文件, 直接用VS打开即可. 接下来就可以尝试编译运行了, 我这个版本会报一个链接错误, 找日志看到:

```log
14>已完成生成项目“COPY_LUA-xxx.vcxproj”的操作 - 失败.
```

往上看找到具体报错, 发现是个python脚本运行失败了.

```log
14>sync_folder.py: error: argument -l: expected one argument
```

这个sync_folder是4.0新增的一个用来同步res和src的脚本, 参考[这篇帖子](https://forum.cocos.org/t/cocos2dx-4-0-cmake/86952), 其中指出:

> 这个bug在于当前的写法并非支持无参选项。会让vs报错就不能直接启动项目。

原本不打算修改sync_folder.py中的内容, 但当我尝试使用python2.7和python3.8去运行这个脚本, 全都报出了相同的错误, 我才确定不是python版本的问题. 因此直接选择修改调用处的参数, 同文章中所讲的, 找到

```python
parser.add_argument("-l", dest="luajit", default=None)
parser.add_argument("-m", dest="mode", default=None)
```

将其修改为:

```python
parser.add_argument("-l", dest="luajit", nargs="?", default=None)
parser.add_argument("-m", dest="mode", nargs="?", default=None)
```

再编译即可, 注意选择启动项目, 我当时选错了半天没启动起来, 会提示找不到目标(捂脸).

----

# Mac上构建Xcode项目

同样的, 首先找个地方创建项目工程

```shell
mkdir ios_mac-build && cd ios_mac-build;
cmake .. -G "Xcode"
```

不出预料报错了, 提示“No CMAKE_C_COMPILER could be found.”, 不知道是不是最近换了Xcode12导致的. 具体报错内容大概长这样:

```log
CMake Error at CMakeLists.txt:28 (project):
    No CMAKE_C_COMPILER could be found.
CMake Error at CMakeLists.txt:28 (project):
    No CMAKE_CXX_COMPILER could be found.
-- Configuring incomplete, errors occurred!
```

首先我尝试在google上搜索, 得出两个解决方案, 有遇到这个问题的小伙伴可以先尝试下:

```shell
sudo xcode-select --reset
sudo xcode-select --switch /Applications/Xcode.app
```

但是在我的问题中并没能正常工作, 于是我想到可能是cmake版本过旧, 或者cmake的配置在更新Xcode之后没有自动调整, 于是我尝试更新cmake:

```shell
brew upgrade cmake
```

完事儿以后就能够正常构建项目工程了. 跑完之后用xcode直接打开生成的xcodeproj即可编译运行. 和Windows环境一样, 记得选编译目标哦, 就在运行按钮的右侧.

### 参考

[关于cocos2dx 4.0 cmake编译的一些坑](https://forum.cocos.org/t/cocos2dx-4-0-cmake/86952)

[Cocos2dx-v4.0学习-使用CMake编译Cocos2d-4.0 (For Visual Studio)](https://blog.csdn.net/hunter_wyh/article/details/104377872)

