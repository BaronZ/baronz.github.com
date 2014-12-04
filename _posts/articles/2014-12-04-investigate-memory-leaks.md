---
layout: post
title: "This Handler class should be static or leaks might occur"

tags: [android, activity, handler, looper, message-queue, memory, memory-leaks]
categories: articles
comments: true
share: true
---
大家应该试过把一个Handler写成一个内部类。如果这个内部类不是静态的话，Eclipse会有一个警告`This Handler class should be static or leaks might occur`。就是说如果这个内部类不声明成静态的，有可能导致内存泄露，代码如下。

{% highlight java %}
public class TestActivity extends Activity{
	@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
	}
	private class LeakHandler extends Handler{
	}
}
{% endhighlight %}


Eclipse也对这个警告做了解释

>Since this handler is declared as an inner class, it may prevent the outer class from being garbage collected. If the Handler is using a Looper or MessageQueue for a thread other than the main thread, then there is no issue. If the Handler is using the Looper or MessageQueue of the main thread, you need to fix your Handler declaration, as follows: Declare the Handler as a static class; In the outer class, instaniate a WeakReference to Handler; Make all references to members of the outer class using the WeakReference object.

上面的解释大概意思就是如果Handler使用的不是主线程(MainThread)的Looper 或者 MessageQueue，那么不声明成static的内部类没问题。但是如果使用的是主线程的Looper或者MessageQueue，那么，你必须把你的内部类声明成静态的，并且在外部类，把这个类声明成弱引用。Handler里面的成员变量，如果有引用到外部类的成员变量，全部声明成弱引用

那么为什么Handler使用主线程的Looper就会导致内存泄露呢？
为什么要声明成静态内部类呢？

1. 当一个安卓应用启动的时候，会为这个应用的主线程创建一个Looper对象。这个Looper对象里面有一个MessageQueue对象。当我们每次触发一个事件(比如点击事件，activity生命周期的回调)，就会往主线程的Looper的MessageQueue里添加message,然后Looper会一个接一个处理queue里面的message。这个主线程的Looper的生命周期与应用的生命周期一样长。
2. 当你的Handler是在主线程里初始化的，并且没有指定Looper,那么这个Handler使用的就是主线程的Looper。当你的handler发送消息(eg:sendEmptyMessage)给这个Looper的MessageQueue的时候，Looper会持用这个Handler的引用，这样Looper处理这个handler发来的message的时候，android这个框架才能调用Handler.handleMessage(Message)
3. 在java中，非静态内部类对外部类会有一个隐式的引用，而静态内部类则没有

假如，我们这么使用handler
{% highlight java %}
@Override
protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
	mLeakyHandler.postDelayed(new Runnable() {
    	@Override
      	public void run() { }
    }, 60 * 10 * 1000);
	finish();
}
{% endhighlight %}
即这个handler是10分钟后才执行的。那么这10分钟内会导致Activity不会被回收。
因为这个delay的message持有Activity里面非静态Handler的一个引用，这个Handler因为是非静态的，所以持有对它的外部类Activity的一个引用。Handler对Activity的引用，要等到10分钟后，那个消息被处理了之后才能解除引用，这样就会导致Activty所引用的资源无法被回收，从而导致内存泄露。
如果你在这10分钟之内, n次进来这个Activity，会导致内存泄露n次，当内存不够用时，就OOM了

所以，这也说明了一个情况，如果Activity的内部类的生命周期可能比Activity的生命周期长，那就要声明成静态内部类



下面，正确的写法就是
{% highlight java %}
public class TestActivity extends Activity{
	private Context mContext;
	//这里的handler要声明成弱引用
	private WeakReference<StaticHandler> mHandlerRef;

	@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
	
		mHandlerRef = new WeakReference(new StaticHandler(mContext));
	｝
	private static class StaticHandler extends Handler{
		//这里的Context是引用外部类的context,所以要声明成弱引用
		private WeakReference<Context> mContextRef; 
		public StaticHandler(Context ctx){
			mContextRef = new WeakReference(ctx);
		}
	}
}
{% endhighlight %}