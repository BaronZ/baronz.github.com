---
layout: post
title: "EditText使用笔记"

tags: [android, edittext]
categories: articles
comments: true
share: true
---
###android:imeOptions
该属性用于修改输入法键盘里的Enter的图标或者文字，比如值为“actionSearch”，图标为搜索的图片或者文字"Search"之类的，类似的还有"Send", "Go"等文字

###android:inputType
该属性用于帮助输入法决定使用什么键盘，比如如果值是"textCapCharacters"时，会第一个字母大写。类似的还有"textPassword", "textEmail", "textPhonetic"

代码输入可以用editText.setInputType(EditorInfo.inputType);

默认是数字，但是可以输入其他，注意xml中不要设置inputType

et.setRawInputType(InputType.TYPE_CLASS_NUMBER);

###光标显示最右边
editText.setSelection(text.length());

###EditText不可编辑(android:editable已经过期)
{% highlight xml %}
<EditText 
        android:clickable="false" 
        android:cursorVisible="false" 
        android:focusable="false" 
        android:focusableInTouchMode="false">
</EditText>
{% endhighlight %}
代码设置
{% highlight java %}
editText.setKeyListener(null);//设了就不能编辑
{% endhighlight %}
###setError()
看官方demo时，发现editText有个很好的方法，setError()。可以弹出错误信息，用法如下
{% highlight java %}
editText.setError(error);
{% endhighlight %}
###自动换行
设置inputType会导致editText不会自动换行

###获取焦点并弹出键盘
{% highlight java %}
et.requestFocus();
et.setSelection(et.getText().toString().length());
InputMethodManager inputManager = (InputMethodManager) getSystemService(Context.INPUT_METHOD_SERVICE);
inputManager.showSoftInput(et, 0);
{% endhighlight %}
###最大字数
android:maxLength

###字符串过滤
InputFilter