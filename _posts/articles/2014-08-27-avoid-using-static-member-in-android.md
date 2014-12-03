---
layout: post
title: "Android中少用静态变量(Android静态变量static生命周期)"

tags: [android, static-lifecycle, process, memory]
categories: articles
comments: true
share: true
---
在android中，要少用静态变量。

我现在做的一个应用中，之前的开发人员使用静态变量来存储cookie，这个全局的静态变量用来验证身份。

客户反应，应用长时间不使用，再次使用，会提示身份过期。

由于长时间使用出现这个bug，也是考虑到应用进程是不是被系统杀了。时间长的bug不好复现，想到进程被杀，就好模拟。
自己的应用打开退到后台，再不断打开其他应用，导致自己的应用被杀，再次打开自己的应用，发现果然身份过期了。
debug一下发现那个静态变量的值是空的。

问题基本确定在静态变量上。

上stackoverflow查了android中static变量的生命周期，有人这么说

> Lifetime of a static variable: A static variable comes into existence when a class is loaded by the JVM and dies when the class is unloaded，if you create an android application and initialize a static variable, it will remain in the JVM until one of the following happens:

> 1. the class is unloaded
2. the JVM shuts down
3. the process dies

我们应用出现的情况就是进程被系统杀掉导致的。


静态变量，要慎用！