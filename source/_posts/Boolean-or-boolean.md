---
title: Unity调用Android时出现的方法签名问题
date: 2019-01-21 18:09:32
tags:
- Unity
- Android
- java
description: 这篇文章主要描述了java方法签名相关知识以及Unity调用Android时出现的一个BUG
categories: 
- 问题分析
---

## 问题描述
不久前给公司的SDK进行升级，使用了如下代码进行Unity端调用Android端方法。

``` C#
// C#端代码
if (JavaClassMain != null)
{
 JavaClassMain.CallStatic("UploadGameServerLoginStatus", ipPort, isSuccess);
}
else
{
 Console.Error.WriteLine("JavaClassMain is null");
}

// java端代码
public static void UploadGameServerLoginStatus(String ipPort, Boolean connectSuccess) 
{
	p("upload!");
	if(null != mPlatform) 
	{
		mPlatform.UploadGameServerLoginStatus(ipPort, connectSuccess);
	}
}
```

其中，ipPort是string类型，isSuccess是bool类型。然后在调用的过程中报错。
	
	java.lang.NoSuchMethodError: no static method with name='UploadGameServerLoginStatus' signature='(Ljava/lang/String;Z)V'
	
## 排查过程

从报错来看，肯定是C#端在调用的时候没有找到对应的方法。

第一反应就是Android端代码在打包时没打进去，于是用AndroidHelper解包出来查了下，发现确实打进去了。然后就陷入迷思状态中，试了很多方法，包括Unity版本，android的编译版本之类的办法，都无效。

重看报错中关于方法签名的部分，方法名肯定是没有错的（专门复制粘贴替换了一下看了一眼），方法修饰符都是static也没错。所以错可能出在方法参数中，于是去详细的查了下java中关于方法签名的部分（具体部分见后一节）。

C#端试图调用一个参数为字符串和bool类型，返回值为void的方法，乍一看没什么错。幸亏早些年写了比较久的java，突然意识到Boolean和boolean可能是无法通过jni过程正常转换的。C#端试图调用的bool类型为Z，也就是boolean，基础类型bool，而非Boolean，引用类型bool，而我签名用的是Boolean而非boolean。改正后问题解决。

为了测试我的想法，我写了如下方法

``` java
public void Init(Boolean a, boolean b, boolean[] c) {
	
}
```

然后通过`javap -s`命令来查询对应的方法签名，得到的签名为：

	(Ljava/lang/Boolean;Z[Z)V

验证了我的想法（注意参数1/2的bool类型的不一致）。

## 附录：java的方法签名字段描述表

此部分引用[这篇文章](https://blog.csdn.net/z69183787/article/details/54140319)

基本类型终结符表

| 终结符 | 类型 |
| :--: | :--:|
| B | byte |
| C | char |
| D | double |
| F | float |
| I | int |
| J | long |
| S | short |
| Z | boolean |
| V | void |

数组类型通过`[`来表示，`L`用来表示对象类型终结符。

以下列出一些方法的描述符示例

| 描述符 | 方法声明 |
| :--: | :--:|
| ()I | int getSize() |
| ()Ljava/lang/String; | String toString() |
| ([Ljava/lang/String;)V |  void main(String[] args) |
| ()V |  void wait() |
| (JI)V | void wait(long timeout, int nanos) |
| (Z[Ljava/lang/String;II)Z | boolean regionMatches(boolean ignoreCase, int toOffset, String other, in oooffset, int len) |
| ([BII)I | int read(byte[] b, int off, int len) |