---
layout: post
title: "重写equals的问题"

tags: [java, equals]
categories: articles
comments: true
share: true
---
我们都知道，在比较两个对象是否相同时，会用到equals方法，比如List的contains方法，就会调用比较对象的equals方法。前段时间在公司写的一个文件查看小工具，就用到了equals这个方法。具体是遍历两个现个文件夹，如果名称相同的话，比较它们的文件大小，大小不同的话就列举出来。

由于不精通算法。就用了土方法。遍历两个指定的文件夹，把拿到的文件对象存到两条列表中。再对一个列表进行遍历，然后用List的contains判断是否是符合要求的文件，是的话就存到另外一个List，最后返回出去。

{% highlight java %}
List<FileEntity> a_files = getFiles("a_floder");//遍历两个指定的文件夹，把拿到的文件对象存到两条列表中
List<FileEntity> b_files = getFiles("b_floder");
List<FileEntity> files = new ArrayList<FileEnityt>();//存同名，不同大小文件的
for(FileEntity f:a_files){
    if(b_files.contains(f)){//这里的contains就是调用了FileEntity的equals方法，所以我重写了FileEntity的equals方法
      files.add(f);
    }
}
{% endhighlight %}
下面是我FileEntity类重写equals方法
{% highlight java %}
public class FileEntity{
   //省略属性的定义
  @Override
   public boolean equals(Object obj){
      if(!(obj instanceof FileEntity)){//    这里可能出现问题   
      	return false;
      }else{
        FileEntity f = (FileEntity)obj;
        if(this.name.equals(f.name)&&this.size!=f.size){//名字相同，大小不同
             return true;
        }else{
             return false;
        }
      }
   }
}
{% endhighlight %}
由于重写了FileEntity的equals方法，所以 b_files.contains(f)这句代码就会调用FileEntity的equals方法，按自己重写的方法进行比较。

但是这里想说的就是，用instanceof有可能出现问题。

因为 a instanceof b  (如果b是a的父类，也是返回true的)，比如下面的代码就会出现问题了。假设Employee继承自Person
{% highlight java %}
@Override       
public boolean equals(Object obj) {
    if(obj instanceof Employee){
      Employee e = (Employee) obj;
      return super.equals(obj)&& e.getId() == id;//id相同就是同一个人            
    }
    return false;       
}  
{% endhighlight %}
如果我们写
{% highlight java %}
Person p = new Person(15);//id为15的人
Employee ep1 = new Employee(15);//id为15的员工
Employee ep2 = new Employee(15);//id为15的员工
Employee ep3 = new Employee(16);//id为16的员工

ep1.equals(eq2);//结果为true，这个是可以接受的
p.equals(ep1);//这个结果也为true，我们期望的应该是false.因为id为15的人不一定是id为15的员工
{% endhighlight %}
上面的问题就是出在 p.equals(ep1)调用的是父类Person的equals方法。而父类的equals方法有   obj instanceof Person，员工也是人，所以返回的是true，又因为id相同，所就equals返回的就是true了。

因`父类使用了instanceof关键字来判断是否是一个类的实例对象的，这很容易让子类“钻空子”`，为了避免这个问题，应该使用`getClass`来代替`instanceof`进行类型判断，改变后的代码如下，
{% highlight java %}
if(obj instanceof Employee)
//用下面这各种方法避免上面提到的问题
if(this.getClass()==obj.getClass())
{% endhighlight %}