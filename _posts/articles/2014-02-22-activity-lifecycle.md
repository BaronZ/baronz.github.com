---
layout: post
title: "Activity生命周期"

tags: [android, activity, activity-lifecycle]
categories: articles
comments: true
share: true
---

## 1.Android生命周期大致可分为三个状态：

* stop:

* pause:此时这个activity在前台仍然部分可见， 但是另一个应用在它的上面运行

* active:此时，activity在前台是完全可见的

## 2. 应用开始显示
* activity一开始会调用onCreate，当activity可见后，会调用onStart，onResume

* 当activity部分可见时，会调用onPause，当activity完全不可见时，会调用onStop。

* 如果用户退出这个应用，或者由于其他原因被杀了，onDestroy会被调用

## 3.应用不在前台
* 当应用不在前台，但是占用很多资源时，android可能会终止该应用。

* 这时候当用户再次回到应用时，会再次调用onCreate方法。

* 如果如没有被系统终止，当用户再次回到这个应用时，此时会调用onRestart，onRestart再调用onStart

## 4.应用打开至退到后台
* 一个应用打开，会调用onCreate,onStart,onResume

* 一个应用退到后台(没有退出应用)，会调用onPause,onStop

* 但是当应用占用资源多时，会调用onDestroy,所以一般会在onStop里存储应用的状态，因为应用被destroy的时机是不可控的。



## 5.entire lifetime(整个生命)
* 从调用onCreate开始到onDestroy结束，比如要下载东西时，在onCreate会创建一个线程，在onDestroy结束线程

## 6.foreground lifetime(前台生命)
* **onResume和onPause这两个方法会被经常调用**在这两个方法之间，activity会显示在其他activity之上，一个activity会常常在这两个状态之差转换，所以里面重写的**方法要轻量(lightweight)**一些。

## 7. visible lifetime (可视生命)
* onStart和onStop之间，这个期间activity可视，但是不一定是在最上面(foreground)，或者和用户交互中。

* 可以在这两个方法里面操作一些资源，比如在onStart注册BroadcastReceiver在onStop里面注销BroadcastReceiver。

* 当activity显示或者隐藏的时候这两个方法会被调用

## 8.onPause这个方法是用来持久化保存一些未保存的数据

<figure>
	<figcaption>Activity生命周期图</figcaption>
	<a href="{{ site.url }}/images/blog/activity_lifecycle.png"><img src="{{ site.url }}/images/blog/activity_lifecycle.png" alt=""></a>
	
</figure>
　　

