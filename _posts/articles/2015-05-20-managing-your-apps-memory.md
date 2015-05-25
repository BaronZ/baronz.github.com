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

## 共享内存(Sharing Memory)

为了满足RAM的需求，Android尝试在进程之间分享内存页面(RAM pages)，下面这些方式可以做到：

* 每个应用的进程都是从Zygote进程fork出来的。Zygote进程在系统启动时启动，然后加载框架代码和资源(比如Activity的主题)。启动一个新的应用进程，系统会从Zygote进程中fork出一个进程，然后在这个新进程里加载和运行应用的代码。这可以让大部分被框架代码和资源申请的内存被所有应用的进程分享

* 大部分的静态数据(static data)被映射到进程中。这不仅可以让部分数据在进程之间分享，也可以在需要使用内存的时候被paged out。比如下面这些静态数据：Dalvik的代码，应用的资源，还有传统的项目元素，比如.so文件里的本地代码。




* 在很多地方，Android在进程之间使用显示地内存回收(ashmem或者gralloc)去分享动态内存。比如，window surfaces在应用和屏幕图像合成器之间使用共享内存，cursor buffers在content provider和client之前使用共享内存。

由于共享内存使用广泛，所以决定应用应用使用多少内存需要很谨慎。如果正常地决定应用使用多少内存，请读[Investigating Your RAM Usage](https://developer.android.com/tools/debugging/debugging-memory.html)

## 申请与回收内存(Allocating and Reclaiming App Memory)

下面这些内容是关于安卓是怎么申请与回收内存的：

* 每个进程的Dalvik堆是被约束在一个单独的虚拟内存范围里的。这定义了逻辑堆(logical heap)的大小，堆大小可以按需增长，但是不能超过系统分配给每个应用的大小


* 逻辑内存的堆大小和物理内存堆使用的大小是不一样的，当你查看你应用的堆，Android会算出一个值，叫Proportional Set Size(PSS)<font color="blue">[3]</font> (实际使用的物理内存,比例分配共享库占用的内存)，which accounts for both dirty and clean pages that are shared with other processes—but only in an amount that's proportional to how many apps share that RAM.
* Android系统会根据PSS的值来判定你的应用占用的物理内存大小。关于PSS的更多信息，请读[Investigating Your RAM Usage](https://developer.android.com/tools/debugging/debugging-memory.html)

* Dalvik堆不会压缩(compact)逻辑堆的大小，这意味着Android不会整理磁盘碎片来关闭空间。Android只会在当堆的尾部有没用过的空间的时候才会shrink(收缩)逻辑堆的大小。但这也不是说被堆使用的物理内存不可以shrink。垃圾回收后，Dalvik遍历堆去发现没用使用的页面(pages)，然后使用madvise返回这些找到的页面给kernel。所以成对地申请与释放大块的内存会导致全部(或者说几乎所有)的物理内存被回收。然而，申请的小内存被回收的效率会相当低，因为这个页面被小内存申请的话可能还被其他没被释放的进程共享着而不能释放。


## 限制应用的内存(Restricting App Memory)
为了维持一个功能性多任务的环境，Android对于应用的堆大小有严格的限制。而具体分配多少大小与设备剩余多少可用内存大小有关。如果你的应用占用的内存已经达到了分配堆的大小，如果再申请内存，就会出现<font color="red">OutOfMemoryError</font>。

有时候，你可能会查询系统去查看你的设备具体有多少内存空间是可以使用的，比如，缓存多少数据是安全的。你可以通过调用<font color="red">getMemoryClass()</font>去确定有多少内存可以使用。这个方法会返回一个整型,这个数值就是你设备可用堆内存的大小(单位是MB)。下面"Check how much memory you should use"章节会有更详细的分析。

## 切换应用(Switching Apps)

在用户切换app的时候，Android会使用LRU(least-recently used)算法去缓存一些后台应用的进程，而不是使用swap分区<font color="blue">[1]</font>。比如，<font color="red">当用户第一次打开一个应用，会创建一条进程，当用户离开应用时，这个进程不会退出。系统会缓存这个进程，当用户再次回来时，这个进程会被复用(切换应用会更加快速)</font>

如果你的应用有缓存的进程，它占有了一些当前不需要使用的内存，那么你的应用，即使用户没在使用，它也会限制系统的性能。所以，当系统内存不够的时候，它就会根据LRU算法去杀掉后台进程，但是，除了LRU算法，也会考虑哪个应用是最占内存的。所以，为了让你应用的进程能够尽量久地留存不被杀掉，你应该遵循下面章节关于什么时候释放引用的建议。

更多关于后台进程是怎么缓存及Android怎么决定哪个应用会被杀掉的信息，请阅读[Processes and Threads](https://developer.android.com/guide/components/processes-and-threads.html)

未完。。。

[1] Swap分区在系统的物理内存不够用的时候，把硬盘空间中的一部分空间释放出来，以供当前运行的程序使用。那些被释放的空间可能来自一些很长时间没有什么操作的程序，这些被释放的空间被临时保存到Swap分区中，等到那些程序要运行时，再从Swap分区中恢复保存的数据到内存中

[2] 内存映射文件，是由一个文件到一块内存的映射。Win32提供了允许应用程序把文件映射到一个进程的函数 (CreateFileMapping)。内存映射文件与虚拟内存有些类似，通过内存映射文件可以保留一个地址空间的区域，同时将物理存储器提交给此区域，内存文件映射的物理存储器来自一个已经存在于磁盘上的文件，而且在对该文件进行操作之前必须首先对文件进行映射。使用内存映射文件处理存储于磁盘上的文件时，将不必再对文件执行I/O操作，使得内存映射文件在处理大数据量的文件时能起到相当重要的作用。

[3] Proportional Set Size 是一个值<font color="red"> PSS = private memory + (shared memory/number of Processes Sharing it)</font> ,Android系统会根据这个值的大小来决定是杀了这个应用

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