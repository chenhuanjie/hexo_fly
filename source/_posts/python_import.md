---
title: Python中的import
---

前段时间由于本蒟蒻才疏学浅, 写出来的服务器被ImportError填满, 于是腾了个时间出来整理下python中import相关的内容, 希望能够对看到这里的同学有些帮助. 其实多数内容都是抄自Python官方文档, 文中相关位置都给出了链接, 其中的内容真的非常详细, 给力! 这个问题本身非常简单, 现有如下所示两个文件:

```python
# a.py
import b
x = 1

# b.py
from a import x
```

然后在python交互环境中敲```import a```就会得到一个```ImportError: cannot import name x```. 表面上看上去由于a中引入了b, b又反过来引入a导致了循环引用, 下面就来考虑下怎么解决这个问题.

---

### 1. import相关概念

import操作大致可以分为两种, 其内部实现均直接或间接使用了__import__方法:

* import module
* from module import name

参考[这份文档](https://docs.python.org/zh-cn/3/reference/simple_stmts.html#the-import-statement), 其定义如下所示:

```
import_stmt     ::=  "import" module ["as" identifier] ("," module ["as" identifier])*
                     | "from" relative_module "import" identifier ["as" identifier]
                     ("," identifier ["as" identifier])*
                     | "from" relative_module "import" "(" identifier ["as" identifier]
                     ("," identifier ["as" identifier])* [","] ")"
                     | "from" module "import" "*"
module          ::=  (identifier ".")* identifier
relative_module ::=  "."* module | "."+
```

我不喜欢原文列出的繁琐步骤, 这里大概概括下:

对于```import module```的方式, 首先查找模块, 进行加载并初始化模块, 并在局部命名空间中定义相对应的名称; 而对于```from module import name```的方式, 在加载并初始化模块完成后, 会去检查每个需要引入的符号是否存在, 如果没有则去查找子模块, 如果存在则在局部命名空间中定义相应名称, 否则抛出异常.

也就是说后者相对前者多了一步对于目标模块是否包含要引入的标识符的判断, 也就是说如果引入的时候目标模块中没有包含对应符号就GG了.

聪明的小伙伴已经发现问题了, 在上面的例子中我们定义```x```的位置在```import b```后面, 所以执行到```from a import x```的时候a中还没有```x```这个变量. 但是为何第二次到达```from a import x```时没有将```a.py```执行一次呢? 以及这个变量是保存在哪里以供import进行检测的呢? 这就需要进一步深入了解.

---

### 2. import导入过程

上面的流程中, 有一步“加载并初始化模块”, 参考[这个文档](https://docs.python.org/zh-cn/3/reference/import.html#loading), 其过程应大致为:

1. 先通过查找器(*finder*)根据给定的路径查找文件位置, 在3.4之前的python中, 查找器将直接返回一个加载器(*loader*), 后来会返回一个模块规格说明(*module spec*)是一个包含有模块引入相关信息的封装
2. 如果有加载器的话使用该加载器创建一个模块对象, 否则直接实例化一个模块对象
3. 模块说明中如果不包含加载器, 但是提供了一个查找路径, 那么它是一个名字空间模块(*namespace module*), 否则不支持引入并raise一个异常
4. 无论名字空间模块还是普通模块, 先将模块对象加入sys.modules, 然后对于普通模块会去执行模块代码, 如果执行过程中出错再把模块对象从sys.modules中移除 (这里实际上有个特例, 就是对于模块说明中没有指明如何执行模块的情况, 但是原文的伪代码我没看懂)

注意如果模块已经存在于sys.modules中, 导入操作会直接将其返回, 此外导入过程会先将模块加入sys.modules以防止导入过程发生不停的循环, 这也就是为什么重复引入的时候代码没有被重复执行.

---

### 3. 可行的方法

通过上面的内容我们已经可以了解到, 一个模块只要被引用过, 无论import语句写在哪里其实都无需担心其代码会被重复执行, 因为会直接返回sys.modules中保存的对象, 并不会真的重新执行一遍, 除非用imp.reload这种方法强制重新载入.

#### 可以不用```from module import name```的方式引入, 使用```import module```

所有地方都用绝对引入的方式直接引入模块, 这样就不会在引入过程中检查标识符是否存在, 如果最终标识符不存在的话则会在运行中抛出异常, 而且可以避免下面会列出来的一个引入标识符的问题.

但是这样做的缺点在于, 如果工程目录结构较为复杂, 绝对引入会导致代码又臭又长, 想象一下所有的函数调用都变成```rootdir.module.sub_module.function_name()```, 那简直惨不忍睹.

#### 可以在函数内, 或者模块底部调用```from module import name```

这个方法我不太喜欢, 虽然不会导致模块重复载入, 但是可读性降低了, 在[pep8](https://www.python.org/dev/peps/pep-0008/#imports)和[flake8](https://www.flake8rules.com/rules/E402.html)中都有指出引入操作应当位于模块开始的地方, 并以标准库、相关三方库、本地应用程序或库的顺序进行引入.

#### 合并代码, 或者重构以合理拆分代码, 将公共部分写入独立的模块

这个方法可以解决多数问题, 但是要改的地方不少, 而且多数情况下需要具体问题具体分析, 并不能一劳永逸.

比方说我们游戏有这样一个逻辑, 登录时需要从登录模块调用任务模块以更新每日任务数据. 当时设计跨天重置逻辑时为了节约存储、保证数据一致性, 判断两次请求跨天只用了用户数据中保存的一个时间戳. 但是有时跨天用户并不会退出游戏, 而此时访问任务数据要求能够触发整个跨天重置的逻辑, 于是这俩互相调用导致引入不成功.

最后在这一点上的处理是考虑模块中会优先加载```__init__.py```的逻辑, 代码(大概)长这样:

```python
# game/__init__.py
import game.login
import game.quest

# game/login.py
import time
from game.config import BASE_TIME  # 某个周一的零点, 用来判断跨天、跨周
from game.quest import refresh_quest
def check_new_day(user):
    last = (user.lastactive - BASE_TIME) % 86400
    today = int(time.time() - BASE_TIME) % 86400
    if last != today:
        quest_refresh(user)  # 跨天时刷新任务

# game/quest
import game.login
def quest_data(user):  # 获取数据时检查是否需要跨天刷新
    game.login.check_new_day(user)

def quest_refresh(user):
    pass
```

#### (捂脸), 上面的方法都没能简单轻松地解决这个问题, 但是我已经是黔驴技穷了, 如果有什么好的方法还请务必告诉我, 感激不尽.

---

### 4. 其他收获

##### 1. 上面有提到import找到目标标识符之后会在局部命名空间中定义相应的名称, 这会导致一个问题, 比如现在有:

```python
# config.py
CONNECTION_POOL_SIZE = 25

# manager.py
from config import CONNECTION_POOL_SIZE
def build():
    pool = ConnectionPool(size=CONNECTION_POOL_SIZE)  # 假设有这样一个类

# main.py
import config
import manager
config.CONNECTION_POOL_SIZE = 3
manager.build()  # 最终得到的pool大小为25
```

这样在执行```main.py```时可以正常修改```config.py```中的连接池大小配置, 但是没能修改```manager.py```中的变量大小, 因为在其中使用```from```进行引入时等同于将变量拷贝到其本地空间一份, 而非常不巧的是这个变量使用了值传递的方式.

这同时意味着如果要引入的变量是通过引用传递的, 那么通过这两个方法得到的结果是一样的, 但我感觉避免误导, 尽量还是不要用这种方式引入配置.

##### 2. 在学习import相关内容的过程中了解到的关于是否使用pyc文件加速引入#的规则:

> 默认情况下, Python 通过在所写入缓存文件中保存源文件的最新修改时间戳和大小来实现这一点. 在运行时, 导入系统会通过比对缓存文件中保存的元数据和源文件的元数据确定该缓存的有效性.
> 在 3.7 版更改: 增加了基于哈希的 .pyc 文件. 在此之前, Python 只支持基于时间戳来确定字节码缓存的有效性.
