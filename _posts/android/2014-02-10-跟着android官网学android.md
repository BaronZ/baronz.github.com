---
layout: post
---
{% include JB/setup %}
```html
由于新项目服务端方面没这么忙，老大也让我学起客户端来。这也就上了andorid官网学习起andoird来了
```

## 1.AndroidManifest.xml

跟着官网写的第一个app，其中提到了AndroidManifest.xml这个文件是描述这个app的特点还有定义各类组件。
该文件中其中一个最重要的元素是<uses-sdk android:minSdkVersion="9" android:targetSdkVersion="17" />.
其中要把targetSdkVersion设得越高越好

## 2.数据库存储sqlite

使用sqlite下面两个方法的时候，要在一个异步执行

Note: Because they can be long-running, be sure that you call getWritableDatabase() or getReadableDatabase() in a background thread, such as with AsyncTask or IntentService

```java
/**BaseColumns 接口
一般可以写个内部类实现这个接口，里面定义一些数据库的表名，列名。方便使用，防止在使用过程中拼写错误*/
public static abstract class FeedEntry implements BaseColumns{
        public static final String TABLE_NAME = "entry";
        public static final String COL_NAME_ENTRY_ID = "entry_id";
        public static final String COL_NAME_TITLE = "title";
        public static final String COL_NAME_SUBTITLE = "subtitle";
    }
```

## 3.android.content.Context 

Context是android.app.Activity的父类
