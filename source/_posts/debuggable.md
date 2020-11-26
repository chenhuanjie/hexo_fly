---
title: 尝试改AndroidManifest并添加Debuggable
---

# 尝试改AndroidManifest并添加Debuggable

最近想玩下RenderDoc, 用Mac版的软件自动patch debuggable时不知为何失败了, 于是想问强哥要个debug包, 未遂. 强哥表示我可以自力更生, 倒也没错就是了(悲). 思路大致有这么三个:

* 可以通过反编译再回编译的方式修改AndroidManifest.xml

* 或者用AXMLEditor这个工具直接修改apk中的二进制AndroidManifest.xml

* 再或者直接上Magisk, 开启全局可调式.

第三种方法由于咱不太懂搞机, 搞不定手头这个红米Note7Pro, 所以单留一个[参考链接](https://blog.csdn.net/yhsnihao/article/details/106760666), 暂且搁置. 这里在Mac上尝试下前两个方法.

<font color="#EEE">本文中涉及的内容比较简单, 没有拆dex, 没有看java源码, 我也就是拆开包来加了两行垃圾, 又给他打包起来了. 水得很.</font>

----

# 反编译+回编译

**注意**: 这个方法前提是程序没有加固, 加固了的先去脱个壳先.

### 1. 装个apktool先

[下载地址](https://ibotpeaches.github.io/Apktool/install/)内包含了Mac的安装步骤, 也就是拷贝到bin下改成可执行, 这里就不提了.

### 2. 拆开

```shell
apktool d com.hello.apk
```

成功后会在当前目录下生成一个和包同名的目录.

### 3. 修改AndroidManifest.xml

找到applicaion, 在属性中修改android:debuggable="true", 如果没有这个属性手动添加就可以了.

```xml
<applicaion android:debuggable="true">
```

### 4. 打包

```shell
apktool b com.hello.apk/
```

成功后会在目标目录下生成dist目录, 其中就有我们需要的apk.

### 5. 生成一个keystore

如果有现成的那这一步可以跳过, 注意-keystore的参数和-alias的参数一定要一致, 不然会找不到有效密钥(其实就是自签名证书啦, 搞一个正经的证书太麻烦).

```shell
keytool -genkey -keystore test.keystore -keyalg RSA -validity 10000 -alias test.keystore
```

过程中会需要输入各种东西, 记住最开始自己输入的那个密码就可以了.

### 6. 重新签名

```shell
jarsigner -verbose -keystore test.keystore -signedjar result.apk com.hello.apk test.keystore
```

过程中会要上一步生成keystore时的密码, 大致当看到这样的输出时就成功了:

```log
>>> 签名者
    X.509, CN=..., OU=..., O=..., L=..., ST=..., C=...
    [可信证书]

jar 已签名。

警告:
签名者证书为自签名证书。
```

至此, 这个包已经可以拿来用了, 只是签名发生了变化, 和签名挂钩的各种验证就过不了了, 但是截个帧绰绰有余.

----

# 使用AXMLEditor修改AndroidManifest

但是某些apk反编译破解后无法成功回编译, 于是有了这样一个工具, 可以直接修改二进制文件, 无需繁琐的反编译、回编译过程, 厉害得很.

### 1. 装个AXMLEditor先

```shell
git clone git@github.com:fourbrother/AXMLEditor.git
```

啊, 实际上这个git仓库的readme上已经写了如何操作了, 下面只需要踩着脚印走一遍就可以了.

### 2. 解压 & 修改

首先把apk的内容unzip出来, 我这里直接用了unzip, 导致文件全都出现在当前目录了

```shell
unzip -n -d com.hello com.hello.apk
```

可以看到AndroidManifest.xml是个二进制文件, 内容无法理解. 尝试使用AXMLEditor对其进行修改.

```shell
java -jar AXMLEditor/AXMLEditor.jar -attr -m application 标签唯一标识 debuggable true com.hello/AndroidManifest.xml com.hello/AndroidManifest_out.xml
```

这里顺带提一下, 看工具源码中[这个位置](https://github.com/fourbrother/AXMLEditor/blob/master/src/cn/wjdiankong/main/XmlEditor.java#L151)对manifest和application标签跳过了标签唯一标识的判断, 所以上面那个位置写啥都行, 甚至不用改.

最后用输出的文件覆盖原本的输入文件:

```shell
mv com.hello/AndroidManifest_out.xml com.hello/AndroidManifest.xml
```

### 3. 压缩 & 二次签名

这样一来就成功在application上添加了debuggable=true属性了, 需要重新把文件打包起来.

**重要**: 因为需要二次签名, 所以要先删掉META-INF, 否则后面jarsigner会出问题.

这里给它原汁原味地还原一下, 注意不要把文件夹给打包进去了:

```shell
cd com.hello/
rm -rf META-INF
zip -r ../com.hello_new.apk ./
cd ..
```

然后是同样的签名过程, 创建keystore请去上面的内容找:

```shell
jarsigner -verbose -keystore test.keystore -signedjar result.apk com.hello_new.apk test.keystore
```

到这里就算搞定了, 这个result.apk就已经是debuggable的了. 可以尝试下adb install, 如果签名有问题的话会安装失败的.

<font color="#EEE">撒, 开始快乐的截帧吧. </font>

---

### 参考&感谢

[【Android测试工具】03. ApkTool在Mac上的安装和使用](https://blog.csdn.net/wirelessqa/article/details/8997168)

[关于keystore的简单介绍](https://blog.csdn.net/zlh313_01/article/details/82424664)

[Android 8.0 以上开启全局可调式](https://blog.csdn.net/yhsnihao/article/details/106760666)

[Android中利用AXMLEditor工具不进行反编译就篡改apk文件](http://www.wjdiankong.cn/archives/1036)
