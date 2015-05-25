---
layout: post
title: "[译]Managing Your App's Memory"

tags: [android, memory, process]
categories: articles
comments: true
share: true
---
本文为翻译Android官网文章[Managing Your App's Memory](https://developer.android.com/training/articles/memory.html)

随机存取内存 (RAM)在任何软件环境中都是很有价值的资源，然而，在手机物理内存有限的情况下，它在手机操作系统里面价值就更大了。虽然Android的虚拟机Dalvik VM会执行垃圾回收，那也不允许你忽略你的应用在何时何地申请和释放内存。

为了让你应用的内存能被虚拟机正常地垃圾回收，你要避免产生内存泄露(常常是因为一个全局成员变量(静态变量)持有一个对象的引用导致的)然后在正确的时候释放对任何对象的引用(下面会讨论到如何在应用的内存回调中释放引用)。对于大部分应用，Dalvik VM会处理垃圾回收：当对象离开了线程的范围，它就会被垃圾回收。

这篇文章解释了Android是如何管理应用的进程和内存回收，还有你可以在开发应用时如何主动地减少内存的使用。如果你想看关于如何分析你应用的内存，可以读[Investigating Your RAM Usage](https://developer.android.com/tools/debugging/debugging-memory.html)

## 安卓怎么管理内存(How Android Manages Memory)
Android没有提供Swap分区<font color="blue">[1]</font>，但是它用内存分页(paging)和内存映射<font color="blue">[2]</font> (memory-mapping,mmapping)来管理内存。这意味着任何你修改的内存，不管是为新对象申请内存，还是touching mmapped pages(不会翻译)，这些内存都会常驻不会被交换(page out)。所以，释放这些内存的唯一方式就是释放这些可能被持有的对象引用，让内存可以被垃圾回收。这里有一个例外:任何文件，如果映射到内存中没有修改，比如代码，在系统需要内存的时候，是可以被paged out的(any files mmapped in without modification, such as code, can be paged out of RAM if the system wants to use that memory elsewhere.）

### 共享内存(Sharing Memory)

为了满足RAM的需求，Android尝试在进程之间分享内存页面(RAM pages)，下面这些方式可以做到：

* 每个应用的进程都是从Zygote进程fork出来的。Zygote进程在系统启动时启动，然后加载框架代码和资源(比如Activity的主题)。启动一个新的应用进程，系统会从Zygote进程中fork出一个进程，然后在这个新进程里加载和运行应用的代码。这可以让大部分被框架代码和资源申请的内存被所有应用的进程分享

* 大部分的静态数据(static data)被映射到进程中。这不仅可以让部分数据在进程之间分享，也可以在需要使用内存的时候被paged out。比如下面这些静态数据：Dalvik的代码，应用的资源，还有传统的项目元素，比如.so文件里的本地代码。




* 在很多地方，Android在进程之间使用显示地内存回收(ashmem或者gralloc)去分享动态内存。比如，window surfaces在应用和屏幕图像合成器之间使用共享内存，cursor buffers在content provider和client之前使用共享内存。

由于共享内存使用广泛，所以决定应用应用使用多少内存需要很谨慎。如果正常地决定应用使用多少内存，请读[Investigating Your RAM Usage](https://developer.android.com/tools/debugging/debugging-memory.html)

### 申请与回收内存(Allocating and Reclaiming App Memory)

下面这些内容是关于安卓是怎么申请与回收内存的：

* 每个进程的Dalvik堆是被约束在一个单独的虚拟内存范围里的。这定义了逻辑堆(logical heap)的大小，堆大小可以按需增长，但是不能超过系统分配给每个应用的大小


* 逻辑内存的堆大小和物理内存堆使用的大小是不一样的，当你查看你应用的堆，Android会算出一个值，叫Proportional Set Size(PSS)<font color="blue">[3]</font> (实际使用的物理内存,比例分配共享库占用的内存)，which accounts for both dirty and clean pages that are shared with other processes—but only in an amount that's proportional to how many apps share that RAM.
* Android系统会根据PSS的值来判定你的应用占用的物理内存大小。关于PSS的更多信息，请读[Investigating Your RAM Usage](https://developer.android.com/tools/debugging/debugging-memory.html)

* Dalvik堆不会压缩(compact)逻辑堆的大小，这意味着Android不会整理磁盘碎片来关闭空间。Android只会在当堆的尾部有没用过的空间的时候才会shrink(收缩)逻辑堆的大小。但这也不是说被堆使用的物理内存不可以shrink。垃圾回收后，Dalvik遍历堆去发现没用使用的页面(pages)，然后使用madvise返回这些找到的页面给kernel。所以成对地申请与释放大块的内存会导致全部(或者说几乎所有)的物理内存被回收。然而，申请的小内存被回收的效率会相当低，因为这个页面被小内存申请的话可能还被其他没被释放的进程共享着而不能释放。


### 限制应用的内存(Restricting App Memory)
为了维持一个功能性多任务的环境，Android对于应用的堆大小有严格的限制。而具体分配多少大小与设备剩余多少可用内存大小有关。如果你的应用占用的内存已经达到了分配堆的大小，如果再申请内存，就会出现<font color="red">OutOfMemoryError</font>。

有时候，你可能会查询系统去查看你的设备具体有多少内存空间是可以使用的，比如，缓存多少数据是安全的。你可以通过调用<font color="red">getMemoryClass()</font>去确定有多少内存可以使用。这个方法会返回一个整型,这个数值就是你设备可用堆内存的大小(单位是MB)。下面"Check how much memory you should use"章节会有更详细的分析。

### 切换应用(Switching Apps)

在用户切换app的时候，Android会使用LRU(least-recently used)算法去缓存一些后台应用的进程，而不是使用swap分区<font color="blue">[1]</font>。比如，<font color="red">当用户第一次打开一个应用，会创建一条进程，当用户离开应用时，这个进程不会退出。系统会缓存这个进程，当用户再次回来时，这个进程会被复用(切换应用会更加快速)</font>

如果你的应用有缓存的进程，它占有了一些当前不需要使用的内存，那么你的应用，即使用户没在使用，它也会限制系统的性能。所以，当系统内存不够的时候，它就会根据LRU算法去杀掉后台进程，但是，除了LRU算法，也会考虑哪个应用是最占内存的。所以，为了让你应用的进程能够尽量久地留存不被杀掉，你应该遵循下面章节关于什么时候释放引用的建议。

更多关于后台进程是怎么缓存及Android怎么决定哪个应用会被杀掉的信息，请阅读[Processes and Threads](https://developer.android.com/guide/components/processes-and-threads.html)

## 你的应用该如何管理内存(How Your App Should Manage Memory)

你在开发的各个阶段，都应该时刻考虑内存的限制。包括应用的设计(开发之前)。你可以通过很多的方式去设计和写代码然后获取高效的结果

为了让你的应用使用内存更高效，你应该遵循下面的一些技术建议

###  (节约使用Service)se services sparingly

如果你的应用需要Service去执行后台任务，除非你需要执行任务，否则不要让Service一直运行(也即使用完了就要结束)。还有你也要小心点不要在任务完成完，因为没有成功关闭Service而导致内存泄露。

<font color="red">当你启动一个Service, 系统会更倾向于保持有Service在运行的进程。这会让这些进程很耗内存。因为被这个Service使用的内存，不能被用作他用，也不可以被page out。这会减少系统所能缓存的进程数目，导致应用切换相对缓慢。</font>它在内存低，不能够维持所有当前有Service在运行的进程时，甚至会导致系统抖动(thrashing)<font color="blue">[4]</font>

<font color="red">控制Service寿命最好的方法就是使用IntentService</font>。IntentService会在做完任务后自动结束。更多的信息，请读[Running in a Background Service](https://developer.android.com/training/run-background-service/index.html)

当Service完成任务后还让它保持运行，是一个应用在内存管理中能犯的最大的错误。所以，不要贪婪地让你的Service一直运行了。它不但会增加你应用因为内存不够而导致体验差，给用户发现了这个不当行为，可能会删除这个应用。

### 在用户界面不可见时，释放内存(Release memory when your user interface becomes hidden)
当用户切换到不同应用，使你的应用不可见时，你应该释放你UI使用的资源。这时候释放UI资源能有效地提高系统能缓存的进程的数目，这对提高用户体验有着直接的影响。


-----------------------------未完。。。

[1] Swap分区在系统的物理内存不够用的时候，把硬盘空间中的一部分空间释放出来，以供当前运行的程序使用。那些被释放的空间可能来自一些很长时间没有什么操作的程序，这些被释放的空间被临时保存到Swap分区中，等到那些程序要运行时，再从Swap分区中恢复保存的数据到内存中

[2] 内存映射文件，是由一个文件到一块内存的映射。Win32提供了允许应用程序把文件映射到一个进程的函数 (CreateFileMapping)。内存映射文件与虚拟内存有些类似，通过内存映射文件可以保留一个地址空间的区域，同时将物理存储器提交给此区域，内存文件映射的物理存储器来自一个已经存在于磁盘上的文件，而且在对该文件进行操作之前必须首先对文件进行映射。使用内存映射文件处理存储于磁盘上的文件时，将不必再对文件执行I/O操作，使得内存映射文件在处理大数据量的文件时能起到相当重要的作用。

[3] Proportional Set Size 是一个值<font color="red"> PSS = private memory + (shared memory/number of Processes Sharing it)</font> ,Android系统会根据这个值的大小来决定是杀了这个应用

[4]系统抖动:在请求分页存储管理中，从主存（DRAM）中刚刚换出（Swap Out）某一页面后（换出到Disk），根据请求马上又换入（Swap In）该页，这种反复换出换入的现象，称为系统颠簸，也叫系统抖动。

危害：系统时间消耗在低速的I/O上，大大降低系统效率。进程对当前换出页的每一次访问，与对RAM中页的访问相比，要慢几个数量级。 

## 英文原文

Random-access memory (RAM) is a valuable resource in any software development environment, but it's even more valuable on a mobile operating system where physical memory is often constrained. Although Android's Dalvik virtual machine performs routine garbage collection, this doesn't allow you to ignore when and where your app allocates and releases memory.


In order for the garbage collector to reclaim memory from your app, you need to avoid introducing memory leaks (usually caused by holding onto object references in global members) and release any Reference objects at the appropriate time (as defined by lifecycle callbacks discussed further below). For most apps, the Dalvik garbage collector takes care of the rest: the system reclaims your memory allocations when the corresponding objects leave the scope of your app's active threads.

This document explains how Android manages app processes and memory allocation, and how you can proactively reduce memory usage while developing for Android. For more information about general practices to clean up your resources when programming in Java, refer to other books or online documentation about managing resource references. If you’re looking for information about how to analyze your app’s memory once you’ve already built it, read Investigating Your RAM Usage.

How Android Manages Memory

Android does not offer swap space for memory, but it does use paging and memory-mapping (mmapping) to manage memory. This means that any memory you modify—whether by allocating new objects or touching mmapped pages—remains resident in RAM and cannot be paged out. So the only way to completely release memory from your app is to release object references you may be holding, making the memory available to the garbage collector. That is with one exception: any files mmapped in without modification, such as code, can be paged out of RAM if the system wants to use that memory elsewhere.


* Each app process is forked from an existing process called Zygote. The Zygote process starts when the system boots and loads common framework code and resources (such as activity themes). To start a new app process, the system forks the Zygote process then loads and runs the app's code in the new process. This allows most of the RAM pages allocated for framework code and resources to be shared across all app processes.

* Most static data is mmapped into a process. This not only allows that same data to be shared between processes but also allows it to be paged out when needed. Example static data include: Dalvik code (by placing it in a pre-linked .odex file for direct mmapping), app resources (by designing the resource table to be a structure that can be mmapped and by aligning the zip entries of the APK), and traditional project elements like native code in .so files.

* In many places, Android shares the same dynamic RAM across processes using explicitly allocated shared memory regions (either with ashmem or gralloc). For example, window surfaces use shared memory between the app and screen compositor, and cursor buffers use shared memory between the content provider and client.

Due to the extensive use of shared memory, determining how much memory your app is using requires care. Techniques to properly determine your app's memory use are discussed in Investigating Your RAM Usage.

Allocating and Reclaiming App Memory

Here are some facts about how Android allocates then reclaims memory from your app:

* The Dalvik heap for each process is constrained to a single virtual memory range. This defines the logical heap size, which can grow as it needs to (but only up to a limit that the system defines for each app).

* The logical size of the heap is not the same as the amount of physical memory used by the heap. When inspecting your app's heap, Android computes a value called the Proportional Set Size (PSS), which accounts for both dirty and clean pages that are shared with other processes—but only in an amount that's proportional to how many apps share that RAM. This (PSS) total is what the system considers to be your physical memory footprint. For more information about PSS, see the Investigating Your RAM Usage guide.

* The Dalvik heap does not compact the logical size of the heap, meaning that Android does not defragment the heap to close up space. Android can only shrink the logical heap size when there is unused space at the end of the heap. But this doesn't mean the physical memory used by the heap can't shrink. After garbage collection, Dalvik walks the heap and finds unused pages, then returns those pages to the kernel using madvise. So, paired allocations and deallocations of large chunks should result in reclaiming all (or nearly all) the physical memory used. However, reclaiming memory from small allocations can be much less efficient because the page used for a small allocation may still be shared with something else that has not yet been freed.


Restricting App Memory

To maintain a functional multi-tasking environment, Android sets a hard limit on the heap size for each app. The exact heap size limit varies between devices based on how much RAM the device has available overall. If your app has reached the heap capacity and tries to allocate more memory, it will receive an OutOfMemoryError.

In some cases, you might want to query the system to determine exactly how much heap space you have available on the current device—for example, to determine how much data is safe to keep in a cache. You can query the system for this figure by calling getMemoryClass(). This returns an integer indicating the number of megabytes available for your app's heap. This is discussed further below, under Check how much memory you should use.

Switching Apps

Instead of using swap space when the user switches between apps, Android keeps processes that are not hosting a foreground ("user visible") app component in a least-recently used (LRU) cache. For example, when the user first launches an app, a process is created for it, but when the user leaves the app, that process does not quit. The system keeps the process cached, so if the user later returns to the app, the process is reused for faster app switching.

If your app has a cached process and it retains memory that it currently does not need, then your app—even while the user is not using it—is constraining the system's overall performance. So, as the system runs low on memory, it may kill processes in the LRU cache beginning with the process least recently used, but also giving some consideration toward which processes are most memory intensive. To keep your process cached as long as possible, follow the advice in the following sections about when to release your references.

More information about how processes are cached while not running in the foreground and how Android decides which ones can be killed is available in the Processes and Threads guide.

How Your App Should Manage Memory

You should consider RAM constraints throughout all phases of development, including during app design (before you begin development). There are many ways you can design and write code that lead to more efficient results, through aggregation of the same techniques applied over and over.

You should apply the following techniques while designing and implementing your app to make it more memory efficient.

Use services sparingly

If your app needs a service to perform work in the background, do not keep it running unless it's actively performing a job. Also be careful to never leak your service by failing to stop it when its work is done.

When you start a service, the system prefers to always keep the process for that service running. This makes the process very expensive because the RAM used by the service can’t be used by anything else or paged out. This reduces the number of cached processes that the system can keep in the LRU cache, making app switching less efficient. It can even lead to thrashing in the system when memory is tight and the system can’t maintain enough processes to host all the services currently running.

The best way to limit the lifespan of your service is to use an IntentService, which finishes itself as soon as it's done handling the intent that started it. For more information, read Running in a Background Service .

Leaving a service running when it’s not needed is one of the worst memory-management mistakes an Android app can make. So don’t be greedy by keeping a service for your app running. Not only will it increase the risk of your app performing poorly due to RAM constraints, but users will discover such misbehaving apps and uninstall them.

Release memory when your user interface becomes hidden

When the user navigates to a different app and your UI is no longer visible, you should release any resources that are used by only your UI. Releasing UI resources at this time can significantly increase the system's capacity for cached processes, which has a direct impact on the quality of the user experience.

To be notified when the user exits your UI, implement the onTrimMemory() callback in your Activity classes. You should use this method to listen for the TRIM_MEMORY_UI_HIDDEN level, which indicates your UI is now hidden from view and you should free resources that only your UI uses.

Notice that your app receives the onTrimMemory() callback with TRIM_MEMORY_UI_HIDDEN only when all the UI components of your app process become hidden from the user. This is distinct from the onStop() callback, which is called when an Activity instance becomes hidden, which occurs even when the user moves to another activity in your app. So although you should implement onStop() to release activity resources such as a network connection or to unregister broadcast receivers, you usually should not release your UI resources until you receive onTrimMemory(TRIM_MEMORY_UI_HIDDEN). This ensures that if the user navigates back from another activity in your app, your UI resources are still available to resume the activity quickly.

Release memory as memory becomes tight

During any stage of your app's lifecycle, the onTrimMemory() callback also tells you when the overall device memory is getting low. You should respond by further releasing resources based on the following memory levels delivered by onTrimMemory():

TRIM_MEMORY_RUNNING_MODERATE
Your app is running and not considered killable, but the device is running low on memory and the system is actively killing processes in the LRU cache.
TRIM_MEMORY_RUNNING_LOW
Your app is running and not considered killable, but the device is running much lower on memory so you should release unused resources to improve system performance (which directly impacts your app's performance).
TRIM_MEMORY_RUNNING_CRITICAL
Your app is still running, but the system has already killed most of the processes in the LRU cache, so you should release all non-critical resources now. If the system cannot reclaim sufficient amounts of RAM, it will clear all of the LRU cache and begin killing processes that the system prefers to keep alive, such as those hosting a running service.
Also, when your app process is currently cached, you may receive one of the following levels from onTrimMemory():

TRIM_MEMORY_BACKGROUND
The system is running low on memory and your process is near the beginning of the LRU list. Although your app process is not at a high risk of being killed, the system may already be killing processes in the LRU cache. You should release resources that are easy to recover so your process will remain in the list and resume quickly when the user returns to your app.
TRIM_MEMORY_MODERATE
The system is running low on memory and your process is near the middle of the LRU list. If the system becomes further constrained for memory, there's a chance your process will be killed.
TRIM_MEMORY_COMPLETE
The system is running low on memory and your process is one of the first to be killed if the system does not recover memory now. You should release everything that's not critical to resuming your app state.
Because the onTrimMemory() callback was added in API level 14, you can use the onLowMemory() callback as a fallback for older versions, which is roughly equivalent to the TRIM_MEMORY_COMPLETE event.

Note: When the system begins killing processes in the LRU cache, although it primarily works bottom-up, it does give some consideration to which processes are consuming more memory and will thus provide the system more memory gain if killed. So the less memory you consume while in the LRU list overall, the better your chances are to remain in the list and be able to quickly resume.

Check how much memory you should use

As mentioned earlier, each Android-powered device has a different amount of RAM available to the system and thus provides a different heap limit for each app. You can call getMemoryClass() to get an estimate of your app's available heap in megabytes. If your app tries to allocate more memory than is available here, it will receive an OutOfMemoryError.

In very special situations, you can request a larger heap size by setting the largeHeap attribute to "true" in the manifest <application> tag. If you do so, you can call getLargeMemoryClass() to get an estimate of the large heap size.

However, the ability to request a large heap is intended only for a small set of apps that can justify the need to consume more RAM (such as a large photo editing app). Never request a large heap simply because you've run out of memory and you need a quick fix—you should use it only when you know exactly where all your memory is being allocated and why it must be retained. Yet, even when you're confident your app can justify the large heap, you should avoid requesting it to whatever extent possible. Using the extra memory will increasingly be to the detriment of the overall user experience because garbage collection will take longer and system performance may be slower when task switching or performing other common operations.

Additionally, the large heap size is not the same on all devices and, when running on devices that have limited RAM, the large heap size may be exactly the same as the regular heap size. So even if you do request the large heap size, you should call getMemoryClass() to check the regular heap size and strive to always stay below that limit.

Avoid wasting memory with bitmaps

When you load a bitmap, keep it in RAM only at the resolution you need for the current device's screen, scaling it down if the original bitmap is a higher resolution. Keep in mind that an increase in bitmap resolution results in a corresponding (increase2) in memory needed, because both the X and Y dimensions increase.

Note: On Android 2.3.x (API level 10) and below, bitmap objects always appear as the same size in your app heap regardless of the image resolution (the actual pixel data is stored separately in native memory). This makes it more difficult to debug the bitmap memory allocation because most heap analysis tools do not see the native allocation. However, beginning in Android 3.0 (API level 11), the bitmap pixel data is allocated in your app's Dalvik heap, improving garbage collection and debuggability. So if your app uses bitmaps and you're having trouble discovering why your app is using some memory on an older device, switch to a device running Android 3.0 or higher to debug it.

For more tips about working with bitmaps, read Managing Bitmap Memory.

Use optimized data containers

Take advantage of optimized containers in the Android framework, such as SparseArray, SparseBooleanArray, and LongSparseArray. The generic HashMap implementation can be quite memory inefficient because it needs a separate entry object for every mapping. Additionally, the SparseArray classes are more efficient because they avoid the system's need to autobox the key and sometimes value (which creates yet another object or two per entry). And don't be afraid of dropping down to raw arrays when that makes sense.

Be aware of memory overhead

Be knowledgeable about the cost and overhead of the language and libraries you are using, and keep this information in mind when you design your app, from start to finish. Often, things on the surface that look innocuous may in fact have a large amount of overhead. Examples include:

Enums often require more than twice as much memory as static constants. You should strictly avoid using enums on Android.
Every class in Java (including anonymous inner classes) uses about 500 bytes of code.
Every class instance has 12-16 bytes of RAM overhead.
Putting a single entry into a HashMap requires the allocation of an additional entry object that takes 32 bytes (see the previous section about optimized data containers).
A few bytes here and there quickly add up—app designs that are class- or object-heavy will suffer from this overhead. That can leave you in the difficult position of looking at a heap analysis and realizing your problem is a lot of small objects using up your RAM.

Be careful with code abstractions

Often, developers use abstractions simply as a "good programming practice," because abstractions can improve code flexibility and maintenance. However, abstractions come at a significant cost: generally they require a fair amount more code that needs to be executed, requiring more time and more RAM for that code to be mapped into memory. So if your abstractions aren't supplying a significant benefit, you should avoid them.

Use nano protobufs for serialized data

Protocol buffers are a language-neutral, platform-neutral, extensible mechanism designed by Google for serializing structured data—think XML, but smaller, faster, and simpler. If you decide to use protobufs for your data, you should always use nano protobufs in your client-side code. Regular protobufs generate extremely verbose code, which will cause many kinds of problems in your app: increased RAM use, significant APK size increase, slower execution, and quickly hitting the DEX symbol limit.

For more information, see the "Nano version" section in the protobuf readme.

Avoid dependency injection frameworks

Using a dependency injection framework such as Guice or RoboGuice may be attractive because they can simplify the code you write and provide an adaptive environment that's useful for testing and other configuration changes. However, these frameworks tend to perform a lot of process initialization by scanning your code for annotations, which can require significant amounts of your code to be mapped into RAM even though you don't need it. These mapped pages are allocated into clean memory so Android can drop them, but that won't happen until the pages have been left in memory for a long period of time.

Be careful about using external libraries

External library code is often not written for mobile environments and can be inefficient when used for work on a mobile client. At the very least, when you decide to use an external library, you should assume you are taking on a significant porting and maintenance burden to optimize the library for mobile. Plan for that work up-front and analyze the library in terms of code size and RAM footprint before deciding to use it at all.

Even libraries supposedly designed for use on Android are potentially dangerous because each library may do things differently. For example, one library may use nano protobufs while another uses micro protobufs. Now you have two different protobuf implementations in your app. This can and will also happen with different implementations of logging, analytics, image loading frameworks, caching, and all kinds of other things you don't expect. ProGuard won't save you here because these will all be lower-level dependencies that are required by the features for which you want the library. This becomes especially problematic when you use an Activity subclass from a library (which will tend to have wide swaths of dependencies), when libraries use reflection (which is common and means you need to spend a lot of time manually tweaking ProGuard to get it to work), and so on.

Also be careful not to fall into the trap of using a shared library for one or two features out of dozens of other things it does; you don't want to pull in a large amount of code and overhead that you don't even use. At the end of the day, if there isn't an existing implementation that is a strong match for what you need to do, it may be best if you create your own implementation.

Optimize overall performance

A variety of information about optimizing your app's overall performance is available in other documents listed in Best Practices for Performance. Many of these documents include optimizations tips for CPU performance, but many of these tips also help optimize your app's memory use, such as by reducing the number of layout objects required by your UI.

You should also read about optimizing your UI with the layout debugging tools and take advantage of the optimization suggestions provided by the lint tool.

Use ProGuard to strip out any unneeded code

The ProGuard tool shrinks, optimizes, and obfuscates your code by removing unused code and renaming classes, fields, and methods with semantically obscure names. Using ProGuard can make your code more compact, requiring fewer RAM pages to be mapped.

Use zipalign on your final APK

If you do any post-processing of an APK generated by a build system (including signing it with your final production certificate), then you must run zipalign on it to have it re-aligned. Failing to do so can cause your app to require significantly more RAM, because things like resources can no longer be mmapped from the APK.

Note: Google Play Store does not accept APK files that are not zipaligned.

Analyze your RAM usage

Once you achieve a relatively stable build, begin analyzing how much RAM your app is using throughout all stages of its lifecycle. For information about how to analyze your app, read Investigating Your RAM Usage.

Use multiple processes

If it's appropriate for your app, an advanced technique that may help you manage your app's memory is dividing components of your app into multiple processes. This technique must always be used carefully and most apps should not run multiple processes, as it can easily increase—rather than decrease—your RAM footprint if done incorrectly. It is primarily useful to apps that may run significant work in the background as well as the foreground and can manage those operations separately.

An example of when multiple processes may be appropriate is when building a music player that plays music from a service for long period of time. If the entire app runs in one process, then many of the allocations performed for its activity UI must be kept around as long as it is playing music, even if the user is currently in another app and the service is controlling the playback. An app like this may be split into two process: one for its UI, and the other for the work that continues running in the background service.

You can specify a separate process for each app component by declaring the android:process attribute for each component in the manifest file. For example, you can specify that your service should run in a process separate from your app's main process by declaring a new process named "background" (but you can name the process anything you like):

<service android:name=".PlaybackService"
         android:process=":background" />
Your process name should begin with a colon (':') to ensure that the process remains private to your app.

Before you decide to create a new process, you need to understand the memory implications. To illustrate the consequences of each process, consider that an empty process doing basically nothing has an extra memory footprint of about 1.4MB, as shown by the memory information dump below.

adb shell dumpsys meminfo com.example.android.apis:empty

** MEMINFO in pid 10172 [com.example.android.apis:empty] **
                Pss     Pss  Shared Private  Shared Private    Heap    Heap    Heap
              Total   Clean   Dirty   Dirty   Clean   Clean    Size   Alloc    Free
             ------  ------  ------  ------  ------  ------  ------  ------  ------
  Native Heap     0       0       0       0       0       0    1864    1800      63
  Dalvik Heap   764       0    5228     316       0       0    5584    5499      85
 Dalvik Other   619       0    3784     448       0       0
        Stack    28       0       8      28       0       0
    Other dev     4       0      12       0       0       4
     .so mmap   287       0    2840     212     972       0
    .apk mmap    54       0       0       0     136       0
    .dex mmap   250     148       0       0    3704     148
   Other mmap     8       0       8       8      20       0
      Unknown   403       0     600     380       0       0
        TOTAL  2417     148   12480    1392    4832     152    7448    7299     148
Note: More information about how to read this output is provided in Investigating Your RAM Usage. The key data here is the Private Dirty and Private Clean memory, which shows that this process is using almost 1.4MB of non-pageable RAM (distributed across the Dalvik heap, native allocations, book-keeping, and library-loading), and another 150K of RAM for code that has been mapped in to execute.

This memory footprint for an empty process is fairly significant and it can quickly grow as you start doing work in that process. For example, here is the memory use of a process that is created only to show an activity with some text in it:

** MEMINFO in pid 10226 [com.example.android.helloactivity] **
                Pss     Pss  Shared Private  Shared Private    Heap    Heap    Heap
              Total   Clean   Dirty   Dirty   Clean   Clean    Size   Alloc    Free
             ------  ------  ------  ------  ------  ------  ------  ------  ------
  Native Heap     0       0       0       0       0       0    3000    2951      48
  Dalvik Heap  1074       0    4928     776       0       0    5744    5658      86
 Dalvik Other   802       0    3612     664       0       0
        Stack    28       0       8      28       0       0
       Ashmem     6       0      16       0       0       0
    Other dev   108       0      24     104       0       4
     .so mmap  2166       0    2824    1828    3756       0
    .apk mmap    48       0       0       0     632       0
    .ttf mmap     3       0       0       0      24       0
    .dex mmap   292       4       0       0    5672       4
   Other mmap    10       0       8       8      68       0
      Unknown   632       0     412     624       0       0
        TOTAL  5169       4   11832    4032   10152       8    8744    8609     134
The process has now almost tripled in size, to 4MB, simply by showing some text in the UI. This leads to an important conclusion: If you are going to split your app into multiple processes, only one process should be responsible for UI. Other processes should avoid any UI, as this will quickly increase the RAM required by the process (especially once you start loading bitmap assets and other resources). It may then be hard or impossible to reduce the memory usage once the UI is drawn.

Additionally, when running more than one process, it's more important than ever that you keep your code as lean as possible, because any unnecessary RAM overhead for common implementations are now replicated in each process. For example, if you are using enums (though you should not use enums), all of the RAM needed to create and initialize those constants is duplicated in each process, and any abstractions you have with adapters and temporaries or other overhead will likewise be replicated.

Another concern with multiple processes is the dependencies that exist between them. For example, if your app has a content provider that you have running in the default process which also hosts your UI, then code in a background process that uses that content provider will also require that your UI process remain in RAM. If your goal is to have a background process that can run independently of a heavy-weight UI process, it can't have dependencies on content providers or services that execute in the UI process.