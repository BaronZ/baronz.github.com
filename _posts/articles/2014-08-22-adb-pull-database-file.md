---
layout: post
title: "adb pull命令复制android数据库文件.db到电脑"

tags: [android, adb-shell, sqlite]
categories: articles
image:
  feature: abstract-6.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---

本文介绍如何在把android系统上的数据文件复制到电脑上。

当然，前提是你的手机要`root`了

步骤如下:

###1. win+r cmd进入命令行

###2. cd 进入[sdk]/platform-tools目录下

###3. 执行下面命令行，复制xxx.db到F:/dest

{% highlight ruby %}
adb pull /data/data/[package name]/databases/xxx.db F:\dest
{% endhighlight %}

###4. 是的，这一步正常来说就要报错了，原因是权限不够
> failed to copy xxx.db to F:\dest\dest.db : Permission denied

###5. 只能用shell命令进行系统里给xxx.db增加读权限了，执行下面命令

{% highlight ruby %}
adb shell
su
cd data/data/[package name]/databases
chmod 664 xxx.db
ctrl+c
{% endhighlight %}

###6. 再次执行步骤3即可