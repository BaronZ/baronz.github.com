---
layout: post
title: "TextView使用笔记"

tags: [android, textview]
categories: articles
image:
  feature: abstract-8.jpg
  credit: dargadgetz
  creditlink: http://www.dargadgetz.com/ios-7-abstract-wallpaper-pack-for-iphone-5-and-ipod-touch-retina/
comments: true
share: true
---
###下划线
 {% highlight java %}
textView.getPaint().setFlags(Paint. UNDERLINE_TEXT_FLAG ); //下划线
 {% endhighlight %}
###抗锯齿
 {% highlight java %}
textView.getPaint().setFlags(Paint.UNDERLINE_TEXT_FLAG|Paint.ANTI_ALIAS_FLAG );
//或者
textView.getPaint().setAntiAlias(true);//抗锯齿
{% endhighlight %}
###中划线
 {% highlight java %}
textview.getPaint().setFlags(Paint. STRIKE_THRU_TEXT_FLAG); 
{% endhighlight %}
###android:onClick 设置无效
需要设置属性android:clickable="true"

###This tag and its children can be replaced by one TextView and a compound drawable
当我们用一个LinearLayout来实现一个ImageView和TextView在一起的时候，就会出现上面的提示。

根据提示来修改，可以使用TextView的drawableLeft等属性，代码如下
 {% highlight xml %}
<TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:drawableLeft="@drawable/ic_launcher"
        android:drawablePadding="4dp" 
        android:gravity="center"
        />
{% endhighlight %}
###在代码中改drawableLeft
 {% highlight java %}
Drawable drawable= getResources().getDrawable(R.drawable.drawable);
/// 这一步必须要做,否则不会显示.
drawable.setBounds(0, 0, drawable.getMinimumWidth(), drawable.getMinimumHeight());
myTextview.setCompoundDrawables(drawable,null,null,null);
//也或参考另一个函数
public void setCompoundDrawablesWithIntrinsicBounds (Drawable left, Drawable top, Drawable right, Drawable bottom)
{% endhighlight %}
###行距
 {% highlight xml %}
android:lineSpacingExtra="3dp"
{% endhighlight %}
###省略号
 {% highlight xml %}
<!-- start,end,middle,marquee-->
android:ellipsize="end"
android:singleLine="true"{% endhighlight %}
###HTML
 {% highlight java %}
//注:font的size属性不起作用，如果需要改变大小，使用h1等的标签
textView.setText(Html.fromHtml("<h1><font color='#FF783F'>text</font></h1>");
{% endhighlight %}
###获取行数
 {% highlight java %}
textview.post(new Runnable() {
    @Override
    public void run() {
        int lineCount = textview.getLineCount();          
    }
});    
{% endhighlight %}