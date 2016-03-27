---
layout: post
title: WinDbg排查.Net问题系列-命令集合
date: 2016-03-27
author: luchigster
comments: false
categories: [Blog]
tags: [AspNet,WinDbg,SOS,SOSEX,kb,ln,threadpool,runaway,threads,clrstack,dso,syncblk,gcroot,mdt,dumpclass,dumpmt,dumpheap,dumpobj,do,da,dumparray,objsize]
excerpt: 整理windbg+sos+sosex的主要命令集合
---

### 一、生成内存镜像

#### Process Explorer 

* 微软自家的绿色软件
* 比较简单，输出完整的dump就可以了
* [下载地址](https://technet.microsoft.com/en-us/sysinternals/processexplorer)

#### 系统自带的任务管理器

* 需要操作系统Windows Vista以上的支持
* 注意默认的任务管理器是32位的，如果需要dump64的进程，请使用 C:\Windows\SysWOW64\Taskmgr.exe

### 二、准备工作

* Debugging Tools for Windows (WinDbg)
  - [下载地址](https://msdn.microsoft.com/en-us/windows/hardware/hh852365)
* 加载符号表 srv\*`<path to symbols>`\*http://msdl.microsoft.com/download/symbols
* 加载SOS .load `<path to sos>`\sos.dll
  - 目录：C:\Windows\Microsoft.NET\Framework`<64位?>`\\v`<CLR版本>`\sos.dll
* 加载SOSEX .load `<path to sos>`\sosex.dll
  - [下载地址](http://www.stevestechspot.com/)

### 三、获取帮助
```
!help
!sos.help #两者等价
```

### 四、系统命令

```
kb                                 #如果没有pdb符号文件，无法还原clrstack，那么只能够用kb了
ln mscorwks!GetCLRFunction+0x2371f #但是kb看到的是native代码，可以用ln(line near)来翻译为正常的托管代码栈
```

### 五、主要命令

```
.time          #捕获dump时系统时间
!threadpool    #CPU、线程使用情况
!runaway       #所有运行的线程cpu时间
!threads       #所有托管线程运行状况，ID：XXXX 说明已经结束并等待回收
~50s           #切换到线程50
!clrstack [-p] #显示当前线程栈，-p显示调用的参数
!dso           #打印栈引用的所有对象 dumpstackobjects
!da 27239b98   #打印数组 dumparray
```

### 六、务必记得三个大招

```
!dumpheap -stat   #打印内存堆简要，记得用-stat，否则数量会太多，配合使用参数 -type -min -short
!objsize 27239b98 #打印数组内容大小，而不是默认的指针大小
!do 0x27246e38    #打印对象 dumpobject
```

### 七、简单指令

#### .foreach

```
.foreach(myVariable {!dumpheap -type System.String -min 30000 -short}){!do myVariable;.echo *************}
```

指令说明：

* 创建变量 myVariable，其值为后面命令输出的每一行
* !dumpheap 输出大小超过30000字节的字符串的内存地址
* !do 打印每一个对象
* .echo 每个对象后面打印一行分隔符

### 八、检查CPU

第一步还是先检查所有栈的使用情况：

```
~* kb 2000      #显示所有线程栈顶最多2000个地址，包含非托管代码，托管代码不能正常显示
~* e !clrstack  #检查全部线程的托管代码栈，或者
~#s             #切换到线程仔细看
!clrstack       #检查当前线程的托管代码栈
!threadpool     #看cpu使用情况
.time           #看总用户态时间
!runaway        #看各个线程的用户态时间，但是单单看一个dump的数据是不准确的，需要连续捕捉多个dump做横向比较才能确定某时间段内cpu是否被某个线程占用多
```

然后检查有没有阻塞：

```
!syncblk #以下两行是输出信息
Index SyncBlock MonitorHeld Recursion Owning Thread Info  SyncBlock Owner
   20 001c6f74           85         1 0f4a0a70  1ea0  30   02f07964 System.Object
```

说明：MonitorHeld 是 85 说明有一个线程占用，还有 84/2=42 线程等待，如果运行以下命令，就可能看到42个线程等待的栈信息

```
 .shell -ci “~* e !clrstack” FIND /C Monitor.Enter
```

### 九、检查异常

如果刚好在出现异常的时候捕获了dump，那么可以查看异常的详细信息：

```
!pe <original exception address>    #打印当前栈的异常，无参数则显示最后一个异常
!u <IP shown in the exceptionstack> #反编译为汇编代码
```

### 十、检查内存

#### 性能计数器

优先在生产环境检查系统性能计数器，用来初步判断内存占用情况：

```
.NET CLR Memory/# Bytes in all Heaps    #所有堆的大小
.NET CLR Memory/Large Object Heap Size  #大对象堆大小
.NET CLR Memory/Gen 2 heap size         #Gen 2堆大小，年老了，准备销毁的对象
.NET CLR Memory/Gen 1 heap size         #Gen 1堆大小，长大了，过渡状态的对象
.NET CLR Memory/Gen 0 heap size         #Gen 0堆大小，刚出生，正在使用的对象
.NET CLR LoadingCurrent Assemblies      #当前加载的程序集大小
Process/Private Bytes                   #对象内部定义的所有字段空间
Process/Virtual Bytes                   #系统预留的空间
```

计数器排查技巧：

* virtual bytes 上升, private bytes 保持不变，说明 virtual byte 泄漏。
  - 原因：有些组件申请了内存，但是没有使用过。
  - 解决：检查内存比较大、引用数量比较多的对象。
* private bytes 上升但是 #Bytes in all heaps 保持不变
  - 原因：native 或者 loader heap 泄漏
  - 解决：检查程序集增加的数量，尤其是动态加载的程序集
* #Bytes in all heaps 和 private bytes 共同升降
  - 解决：检查 .net GC heaps

#### 内存占用

```
!address -summary  #打印内存占用情况
!eeheap -gc        #输出8个堆，因为cpu是8核
Number of GC Heaps: 8
------------------------------
Heap 0 (0000000001f7d0e0)
!finalizequeue     #查看销毁队列，如果很多在这个队列中，有可能休眠导致无法销毁而一直占用内存
```

#### 杀手锏

```
!gcroot <address of string> #检查内存对应的对象在哪里
!mdt 0000000485a95768 [-r]  #打印对象的字段内容，-r可以递归显示内部对象字段
```

#### 排查native内存泄漏

如果是native泄漏，则检查当前加载了多次的那些动态module，固定要加载的module，一般没有问题

```
!dumpdomain                 #加载的所有程序集，看那个module重复加载多次
!dumpmodule <moduleaddress> #输出模块基本信息，如下
MetaData start address: 114d09e4 (4184 bytes)
dc 114d09e4 114d09e4+0n4184 #查看dll元数据信息，0n表示10进制，看记载的是那个程序集
```

#### 排查gc heap内存泄漏 

重复多查几次，一般就可以查出来

```
!dumpheap -stat         #看那些底部占用内存最高的
!dumpheap -min 85000    #查大对象，看是不是同类型的都一样大小，有可疑
!gcroot <address>       #随意一个对象地址，看下这个对象长在那个对象上
DOMAIN(001CCE68):HANDLE(Strong):2c510a8:Root:  06eb6238(System.Threading._TimerCallback)->
0x8265810(MemoryIssues.StuffHappenedEventHandler)->
!dumpdomain 001CCE68    #查看程序集信息
```

输出说明

* DOMAIN(001CCE68) 所在的程序集
* HANDLE(Strong) 一般是静态变量

HANDLE(类型)说明

```
ESP       Extended Stack Pointer, 对象在栈
Strong    Strong reference, 强引用，如静态变量
WeakLn    Weak Long Handle, 弱引用，在销毁过程中，可复活
WeakSh    Weak Short Handle, 弱引用，在销毁过程中，不可复活
Pinned    Pinned object, 钉在特定地址，在gc时无法移动
RefCnt    Reference count, 引用计数器，只要>0就一直引用
```

如果gcroot输出的是委托，但是又不知道那个方法绑定了这个委托，怎么办？

```
!dumpobj 0x8265810  #输出方法，找到指针 _methodPtr
        MT      Field     Offset                 Type       Attr      Value Name
0x79b97094 0x400004c      0xc         System.Int32   instance 34845811 _methodPtr
0x79b97094 0x400004d      0x4                CLASS   instance 0x08264500 _target

?0n34845811         #转换为16进制
Evaluate expression: 34845811 = 0213b473

!ip2md 0213b473     #尝试获取方法的描述
```

如果幸运的话，这个ip2md(instruction pointer to method descriptor)，或许可以获取方法的描述，否则只能够从对象所在的类的元数据上去手动找方法

```
!dumpobj 0x08264500     #输出方法所在的类 _target
Name: ASP.WebForm1_aspx
MethodTable 0x0213b78c
EEClass 0x0215a214

!dumpclass 0x0215a214   #查看类元数据
Method Table : 0x0213b4cc

!dumpmt -md 0x0213b4cc  #查看方法表，从方法表里面对照找地址为0x0213b473的方法就可以了
0x0213b473 0x0213b478 None [DEFAULT] [hasThis] Void MemoryIssues.WebForm1.MyStaticObject_StuffHappened(Object,Class System.EventArgs)
```

### 参考
1. [.NET Debugging Demos – Information and setup instructions](https://blogs.msdn.microsoft.com/tess/2008/02/04/net-debugging-demos-information-and-setup-instructions/)
2. [.NET Memory Leak Case Study: The Event Handlers That Made The Memory Baloon](http://blogs.msdn.com/tess/archive/2006/01/23/net-memory-leak-case-study-the-event-handlers-that-made-the-memory-baloon.aspx)
3. [ASP.NET Quiz Answers: Does Page.Cache leak memory?](http://blogs.msdn.com/tess/archive/2006/08/11/695268.aspx)
