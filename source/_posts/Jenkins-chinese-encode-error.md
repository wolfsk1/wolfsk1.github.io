---
title: Jenkins控制台中文乱码问题
date: 2018-11-19 18:26:32
tags:
- jenkins
- 持续集成
description: 这篇文章主要描述了Jenkins中控制台输出也就是构建日志中出现中文乱码的解决方法与相关分析。
categories: 
- 问题分析
---

## 问题描述
jenkins使用war包的形式在windows上通过批处理脚本启动。构建时**控制台输出**（也就是日志部分）的中文显示为乱码。

## 问题涉及的一些知识点
- jenkins的日志系统将控制台输出的日志重定向到某个日志文件中，因此文件的编码格式决定了日志的编码格式。构建相关日志文件在**对应job目录下的build文件夹**中，打开后可以查看编码格式来确定日志编码格式。
- 在jenkins系统管理-系统信息中，可以查找到file.encoiing一栏，查看到当前的**文件处理时所使用的**编码格式。
- 由于jenkins本身是使用java运行的，因此java使用何种方式处理文件便是关键。

## 解决方案
1. 添加系统变量**JAVA_TOOL_OPTIONS**，值为**-Dfile.encoding=UTF-8**。此方法带来的问题是，war包运行jenkins时，启动jenkins的那个cmd窗口中的输出如果涉及中文，将会乱码。推测这与系统环境编码为GBK有关。同时不确定对外部的影响，没有进行测试。
2. 直接在启动jenkins对应的bat或者命令行窗口中输入
	
	set JAVA_TOOL_OPTIONS=-Dfile.encoding=UTF-8 
	CHCP 65001
	
其中第一行是设置临时系统变量，只在当前命令行窗口中生效。第二行官方的解释是设置活动代码页的编号，个人理解就是设置字符集，65001是UTF-8的字符集。这就解决了命令行窗口使用GBK编码显示的乱码问题。同样的，第二行指令也是在当前命令行窗口中生效，不会影响外部。
3. 由于问题发生的时候jenkins的运行方式为war包方式，此方法尚未尝试。而直接通过.msi文件安装的（也就是有安装目录）jenkins可以试一下此方法。在Jenkins安装目录下找到**jenkins.xml**文件，在&lt;arguments&gt; ……&lt;/arguments&gt;中添加**-Difile.encoding=utf-8**。

## 问题总结
本质上这个问题是JVM对FileEncoding的方式问题。