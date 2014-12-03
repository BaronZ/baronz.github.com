---
layout: post
title: 跟着Android官网学习Activity
tags: [android, activity]
categories: articles
comments: true
share: true
---


## 1.Activity介绍 ##

An Activity is an application component that provides a screen with which users can interact in order to do something, such as dial the phone, take a photo, send an email, or view a map. Each activity is given a window in which to draw its user interface. （说白了Activity就是一个和用户交互的界面） The window typically fills the screen, but may be smaller than the screen and float on top of other windows.(一般来说Activity会填满屏幕，也有可能比屏幕小，浮在其他窗口(Activity)的上面)

An application usually consists of multiple activities that are loosely bound to each other. Typically, one activity in an application is specified as the "main" activity,（有的Activity会被设为主Activity（<Activiy>元素里含有<intent-filter>）,当程序第一次启动的时候，会显示这个Activity） which is presented to the user when launching the application for the first time. Each activity can then start another activity in order to perform different actions.(一般来说，每个Activity都可以启动其他Activity) Each time a new activity starts, the previous activity is stopped, (每次一个新的Activity启动，前面启动的那个Activity就会停止)but the system preserves the activity in a stack (the "back stack")（但是系统会把这些Activity保存在栈里面）. When a new activity starts, it is pushed onto the back stack and takes user focus.(每次一个activit启动，就会被它放到back stack里同，且获得聚集) The back stack abides to the basic "last in, first out" stack mechanism（这个栈是后进先出的）, so, when the user is done with the current activity and presses the Back button, it is popped from the stack (and destroyed) and the previous activity resumes.（所以当对当前Activity_2操作完成，要返回Activity_2时，Activity_2会从back stack中弹出，并且被销毁，而Activity_1则会resume） (The back stack is discussed more in the Tasks and Back Stack document.)

When an activity is stopped because a new activity starts, it is notified of this change in state through the activity's lifecycle callback methods. There are several callback methods that an activity might receive,(当一个Activity因为一个新的Activity的启动而改变状态时，会调用相应的回调函数) due to a change in its state—whether the system is creating it, stopping it, resuming it, or destroying it—and each callback provides you the opportunity to perform specific work that's appropriate to that state change. For instance, when stopped, your activity should release any large objects, such as network or database connections（当Activity停止(stop)的时候，应该释放一些大的对象，比如网络连接，数据库连接）. When the activity resumes, you can reacquire the necessary resources and resume actions that were interrupted（但是Activity resume的时候，可以继续之前中断的一些操作）. These state transitions are all part of the activity lifecycle.


### 1.1  back stack 后退栈 (先进后出)([ ]表示栈)

| Step        | Stack           |                
|:------------- |:-------------|
| 应用启动显示Activity_1,栈会插入Activity_1      | [Activity_1] |
|----
| 跳到Activity_2,栈插入Activity_2      | [Activity_1,Activity_2]     |
|----
| 跳转到Activity_3,栈插入Activity_3 | [Activity_1,Activity_2,Activity_3]      |
|----
| 按返回键，Activity_3会从back stack中弹出，并且被销毁，Activity_2会继续(resume) | [Activity_1,Activity_2]      |
|----
{: rules="groups"}

### 总结 ###
Activity一显示，就会被压到栈中，一按返回按钮，就会被弹出栈，栈先进后出。应用只显示栈顶的Activity

## 2.创建Activity

写一个类继承Activity,实现一些生命周期的类，比如:created, stopped, resumed, or destroyed
其中下面两个方法是最重要的
 <font color='red'>onCreate()</font>
You must implement this method.<font color='red'>必须实现的方法</font> The system calls this when creating your activity(系统创建activity的时候就会调用该方法). Within your implementation, you should initialize the essential components of your activity. （在onCreate()实现里，要对activity的组件进行初始化）Most importantly, this is where you must call <font color='red'>setContentView()</font> to define the layout for the activity's user interface.（最重要的是还要高用setContentView(),这个方法是用来定义这个activiy的布局的）
<font color='red'>onPause()</font>
The system calls this method as the first indication that the user is leaving your activity (though it does not always mean the activity is being destroyed). This is usually where you should commit any changes that should be persisted beyond the current user session (because the user might not come back)(系统会调用这个方法做为第一迹象表明用户要离开这个Activity。这时候,这个方法需要对一些超过当前用户session的“更改”进行保存。虽然不一定意味着这个Activity会被destory,但是用户可能不会再回来)
