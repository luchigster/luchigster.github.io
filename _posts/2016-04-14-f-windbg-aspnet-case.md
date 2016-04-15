---
layout: post
title: WinDbg排查.Net问题系列案例2-CPU分析
date: 2016-04-14
author: luchigster
comments: false
categories: [Blog]
tags: [AspNet,WinDbg,do,threadpool,kb,ip2md,savemodule,SiteServer,CMS,DES]
excerpt: 分析了SiteServerCMS持续高CPU使用率后，得出结论是系统后台在读取用户名时，从Cookies中提取后再进行DES解密，频繁的解密操作导致CPU居高不下。
---

有同事反应CMS响应缓慢，看CPU曲线抖动的厉害，并且高峰期CPU占用率较高。
于是要求用procexp输出dump，并且提供64bit的sos。

拿到dump文件后，第一时间先查看cpu是否跟同事说的一样。

```
0:001> !threadpool
CPU utilization: 46%
Worker Thread: Total: 23 Running: 1 Idle: 7 MaxLimit: 32767 MinLimit: 8
Work Request in Queue: 0
--------------------------------------
Number of Timers: 3
--------------------------------------
Completion Port Thread:Total: 2 Free: 2 MaxFree: 16 CurrentLimit: 2 MaxLimit: 1000 MinLimit: 8
```

是的，占用cpu比较高，达到46%，当然这未必是CMS造成的，操作系统也可能进行其他工作。在看当前线程数23个，而正在运行的只有1个。

检查下全部clr栈，看运行了什么东西。

```
0:001> ~* e !clrstack
省略几百行...
OS Thread Id: 0x1e6bc (67)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x7f4 (68)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2f3d4 (69)
        Child SP               IP Call Site
0000000016b0ace8 0000000077a8db2a [HelperMethodFrame_2OBJ: 0000000016b0ace8] System.Security.Cryptography.Utils._AcquireCSP(System.Security.Cryptography.CspParameters, System.Security.Cryptography.SafeProvHandle ByRef)
0000000016b0ade0 000007fedcfac684 System.Security.Cryptography.CryptoAPITransform..ctor(Int32, Int32, Int32[], System.Object[], Byte[], System.Security.Cryptography.PaddingMode, System.Security.Cryptography.CipherMode, Int32, Int32, Boolean, System.Security.Cryptography.CryptoAPITransformMode)
0000000016b0aea0 000007fedd9ab85c System.Security.Cryptography.DESCryptoServiceProvider._NewEncryptor(Byte[], System.Security.Cryptography.CipherMode, Byte[], Int32, System.Security.Cryptography.CryptoAPITransformMode)
0000000016b0af70 000007fedd9ab3d4 System.Security.Cryptography.DESCryptoServiceProvider.CreateDecryptor(Byte[], Byte[])
0000000016b0afc0 000007fe9a927aae BaiRong.Core.Cryptography.DESEncryptor.DesDecrypt()
0000000016b0b030 000007fe9a925a66 BaiRong.Core.RuntimeUtils.DecryptStringByTranslate(System.String)
0000000016b0b070 000007fe9a92b815 BaiRong.Provider.Data.SqlServer.AdministratorDAO.get_UserName()
0000000016b0b0c0 000007fe9a935bcf BaiRong.Core.AdminManager.get_Current()
0000000016b0b110 000007fe9a942c55 SiteServer.CMS.Core.Security.AdminUtility.HasChannelPermissions(Int32, Int32, System.String[])
0000000016b0b170 000007fe9b0c3b8e SiteServer.CMS.Core.ContentUtility.GetCommandItemRowsHtml(BaiRong.Model.ETableStyle, SiteServer.CMS.Model.PublishmentSystemInfo, SiteServer.CMS.Model.NodeInfo, BaiRong.Model.ContentInfo, System.String)
0000000016b0b230 000007fe9b0c1270 SiteServer.CMS.BackgroundPages.BackgroundContent.rptContents_ItemDataBound(System.Object, System.Web.UI.WebControls.RepeaterItemEventArgs)
0000000016b0b300 000007fedab97fc7 System.Web.UI.WebControls.Repeater.CreateItem(Int32, System.Web.UI.WebControls.ListItemType, Boolean, System.Object)
0000000016b0b350 000007feda1ec763 System.Web.UI.WebControls.Repeater.CreateControlHierarchy(Boolean)
0000000016b0b3f0 000007feda1eca94 System.Web.UI.WebControls.Repeater.OnDataBinding(System.EventArgs)
0000000016b0b430 000007fe9ad3cb06 BaiRong.Controls.SqlPager.DataBind()
0000000016b0b490 000007fe9ad2eba8 SiteServer.CMS.BackgroundPages.BackgroundContent.Page_Load(System.Object, System.EventArgs)
0000000016b0b5b0 000007feda19d507 System.Web.UI.Control.LoadRecursive()
0000000016b0b600 000007feda1be1ea System.Web.UI.Page.ProcessRequestMain(Boolean, Boolean)
0000000016b0b6c0 000007feda1bd469 System.Web.UI.Page.ProcessRequest(Boolean, Boolean)
0000000016b0b730 000007feda1bd2c7 System.Web.UI.Page.ProcessRequest()
0000000016b0b7d0 000007feda1bb9f3 System.Web.UI.Page.ProcessRequest(System.Web.HttpContext)
0000000016b0b820 000007feda1c62a1 System.Web.HttpApplication+CallHandlerExecutionStep.System.Web.HttpApplication.IExecutionStep.Execute()
0000000016b0b900 000007feda18d495 System.Web.HttpApplication.ExecuteStep(IExecutionStep, Boolean ByRef)
0000000016b0b9a0 000007feda1aabda System.Web.HttpApplication+PipelineStepManager.ResumeSteps(System.Exception)
0000000016b0baf0 000007feda18d6a3 System.Web.HttpApplication.BeginProcessRequestNotification(System.Web.HttpContext, System.AsyncCallback)
0000000016b0bb40 000007feda1875de System.Web.HttpRuntime.ProcessRequestNotificationPrivate(System.Web.Hosting.IIS7WorkerRequest, System.Web.HttpContext)
0000000016b0bbe0 000007feda190561 System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
0000000016b0bdf0 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
0000000016b0be40 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
0000000016b0c628 000007fef9eeb42e [InlinedCallFrame: 0000000016b0c628] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016b0c628 000007feda2395eb [InlinedCallFrame: 0000000016b0c628] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016b0c600 000007feda2395eb DomainNeutralILStubClass.IL_STUB_PInvoke(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016b0c6d0 000007feda19074f System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
0000000016b0c8e0 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
0000000016b0c930 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
0000000016b0e288 000007fef9eeb42e [InlinedCallFrame: 0000000016b0e288] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016b0e288 000007feda2395eb [InlinedCallFrame: 0000000016b0e288] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016b0e260 000007feda2395eb DomainNeutralILStubClass.IL_STUB_PInvoke(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016b0e330 000007feda19074f System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
0000000016b0e540 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
0000000016b0e590 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
0000000016b0e788 000007fef9eeb683 [ContextTransitionFrame: 0000000016b0e788] 
省略几百行...
```

跟!threadpool的结果一致，当前只有一个托管线程，切换到 69 号线程

```
~69s
```

重新打印栈

```
0:069> !clrstack
OS Thread Id: 0x2f3d4 (69)
        Child SP               IP Call Site
0000000016b0ace8 0000000077a8db2a [HelperMethodFrame_2OBJ: 0000000016b0ace8] System.Security.Cryptography.Utils._AcquireCSP(System.Security.Cryptography.CspParameters, System.Security.Cryptography.SafeProvHandle ByRef)
0000000016b0ade0 000007fedcfac684 System.Security.Cryptography.CryptoAPITransform..ctor(Int32, Int32, Int32[], System.Object[], Byte[], System.Security.Cryptography.PaddingMode, System.Security.Cryptography.CipherMode, Int32, Int32, Boolean, System.Security.Cryptography.CryptoAPITransformMode)
0000000016b0aea0 000007fedd9ab85c System.Security.Cryptography.DESCryptoServiceProvider._NewEncryptor(Byte[], System.Security.Cryptography.CipherMode, Byte[], Int32, System.Security.Cryptography.CryptoAPITransformMode)
0000000016b0af70 000007fedd9ab3d4 System.Security.Cryptography.DESCryptoServiceProvider.CreateDecryptor(Byte[], Byte[])
0000000016b0afc0 000007fe9a927aae BaiRong.Core.Cryptography.DESEncryptor.DesDecrypt()
0000000016b0b030 000007fe9a925a66 BaiRong.Core.RuntimeUtils.DecryptStringByTranslate(System.String)
0000000016b0b070 000007fe9a92b815 BaiRong.Provider.Data.SqlServer.AdministratorDAO.get_UserName()
0000000016b0b0c0 000007fe9a935bcf BaiRong.Core.AdminManager.get_Current()
0000000016b0b110 000007fe9a942c55 SiteServer.CMS.Core.Security.AdminUtility.HasChannelPermissions(Int32, Int32, System.String[])
0000000016b0b170 000007fe9b0c3b8e SiteServer.CMS.Core.ContentUtility.GetCommandItemRowsHtml(BaiRong.Model.ETableStyle, SiteServer.CMS.Model.PublishmentSystemInfo, SiteServer.CMS.Model.NodeInfo, BaiRong.Model.ContentInfo, System.String)
0000000016b0b230 000007fe9b0c1270 SiteServer.CMS.BackgroundPages.BackgroundContent.rptContents_ItemDataBound(System.Object, System.Web.UI.WebControls.RepeaterItemEventArgs)
0000000016b0b300 000007fedab97fc7 System.Web.UI.WebControls.Repeater.CreateItem(Int32, System.Web.UI.WebControls.ListItemType, Boolean, System.Object)
0000000016b0b350 000007feda1ec763 System.Web.UI.WebControls.Repeater.CreateControlHierarchy(Boolean)
0000000016b0b3f0 000007feda1eca94 System.Web.UI.WebControls.Repeater.OnDataBinding(System.EventArgs)
0000000016b0b430 000007fe9ad3cb06 BaiRong.Controls.SqlPager.DataBind()
0000000016b0b490 000007fe9ad2eba8 SiteServer.CMS.BackgroundPages.BackgroundContent.Page_Load(System.Object, System.EventArgs)
```

看到这几行，知道是在进行`DES`的解密。关于DES的问题，网上已经有说法：

```
关于Des的算法，最高速度可以达到10M/s，但是，CPU的占用也相当可观，要想使程序不被当调，就是在你进行加解密算法的的线程中，每达到一定的速度就sleep(1)，这样，不至于它老占用CPU的时间，影响其他线程的执行。至于这个一定的速度，可以自己尝试查找。或者你可以调整一下Des的内部算法机制，因为他的内部算法中有个循环处理语句，Cpu的执行效率就是影响在这里了。 
```

所以基本上可以认为是`DES`导致的cpu阻塞，再看看是否有锁：

```
0:069> !syncblk
Index SyncBlock MonitorHeld Recursion Owning Thread Info  SyncBlock Owner
-----------------------------
Total           1385
CCW             12
RCW             3
ComClassFactory 0
Free            284
```

貌似不管用，换个sosex的命令，查是否有死锁

```
0:069> !dlk
Examining SyncBlocks...
Scanning for ReaderWriterLock(Slim) instances...
Scanning for holders of ReaderWriterLock locks...
Scanning for holders of ReaderWriterLockSlim locks...
Examining CriticalSections...
Scanning for threads waiting on SyncBlocks...
Scanning for threads waiting on ReaderWriterLock locks...
Scanning for threads waiting on ReaderWriterLocksSlim locks...
Scanning for threads waiting on CriticalSections...
No deadlocks detected.
```

也不管用，再换一个，发现一个锁在临界区`CriticalSections`。
深入了解临界区请[点击](http://www.cnblogs.com/dirichlet/archive/2011/03/16/1986251.html)。

_临界区是一种轻量级机制，在某一时间内只允许一个线程执行某个给定代码段。_
_通常在修改全局数据（如集合类）时会使用临界区。_
_事件、多用户终端执行程序和信号量也用于多线程同步，但临界区与它们不同，它并不总是执行向内核模式的控制转换，这一转换成本昂贵。_
_要获得一个未占用临界区，事实上只需要对内存做出很少的修改，其速度非常快。_
_只有在尝试获得已占用临界区时，它才会跳至内核模式。这一轻量级特性的缺点在于临界区只能用于对同一进程内的线程进行同步。_
_几乎所有的多线程程序均使用临界区。您迟早都会遇到一个使代码死锁的临界区，并且会难以确定是如何进入当前状态的。_

```
0:069> !mlocks
Examining SyncBlocks...
Scanning for ReaderWriterLock(Slim) instances...
Scanning for holders of ReaderWriterLock locks...
Scanning for holders of ReaderWriterLockSlim locks...
Examining CriticalSections...

ClrThread  DbgThread  OsThread    LockType    Lock              LockLevel
------------------------------------------------------------------------------
0x3b       69         0x2f3d4     thinlock    000000048063ffa8  (recursion:0)
```

是一个轻量锁`thinlock`，看来应该是`DES`造成的。

手头没有源代码，所以需要根据从!clrstack输出的`IP`指针来输出对应的程序集。

回头看栈

```
        Child SP               IP Call Site
0000000016b0b490 000007fe9ad2eba8 SiteServer.CMS.BackgroundPages.BackgroundContent.Page_Load(System.Object, System.EventArgs)
```

显示`IP`信息，看到这个`DLL`是没有预编译的`IsJitted=yes`

```
0:069> !ip2md 000007fe9ad2eba8 
MethodDesc:   000007fe9acdb888
Method Name:  SiteServer.CMS.BackgroundPages.BackgroundContent.Page_Load(System.Object, System.EventArgs)
Class:        000007fe9aced0b8
MethodTable:  000007fe9acdb948
mdToken:      00000000060003dd
Module:       000007fe9aa4aad8
IsJitted:     yes
CodeAddr:     000007fe9ad2e560
Transparency: Critical
```

看在那个程序集

```
0:069> !DumpModule /d 000007fe9aa4aad8
Name:       C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Temporary ASP.NET Files\root\ddad0b42\624b781\assembly\dl3\f91ca3aa\00d00f64_445ed101\SiteServer.CMS.dll
Attributes: PEFile 
Assembly:   0000000010e58e10
LoaderHeap:              0000000000000000
TypeDefToMethodTableMap: 000007fe9aa60020
TypeRefToMethodTableMap: 000007fe9aa61ca8
MethodDefToDescMap:      000007fe9aa62c48
FieldDefToDescMap:       000007fe9aa74260
MemberRefToDescMap:      0000000000000000
FileReferencesMap:       000007fe9aa7ef60
AssemblyReferencesMap:   000007fe9aa7ef68
MetaData start address:  0000000053347688 (1517568 bytes)
```

是这个程序集`SiteServer.CMS.dll`，找到程序集的内存起始地址

```
0:069> lmvm SiteServer.CMS
Browse full module list
start             end                 module name
```

这时候，并没有显示全部，点击`Browse full module list`，或者输入`lmDv`才会查看全部模块

```
0:069> lmDv
```

由于输出的内容很多，就不贴出来了，所以可以拷贝到记事本，搜索 `\SiteServer.CMS.dll`（加个反斜杠可以过滤那些不属于地址的内容），然后会发现

```
00000000`72720000 00000000`72958000   SiteServer_CMS_72720000   (deferred)             
    Image path: C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Temporary ASP.NET Files\api\18830d0c\1a46d8d6\assembly\dl3\a5a5ccd7\924a1d58_1c96d001\SiteServer.CMS.dll
    Image name: SiteServer.CMS.dll
    Browse all global symbols  functions  data
    Has CLR image header, track-debug-data flag not set
    Timestamp:        Thu May 14 22:41:06 2015 (5554B402)
    CheckSum:         00000000
    ImageSize:        00238000
    File version:     0.0.0.0
    Product version:  0.0.0.0
    File flags:       0 (Mask 3F)
    File OS:          4 Unknown Win32
    File type:        2.0 Dll
    File date:        00000000.00000000
    Translations:     0000.04b0
    InternalName:     SiteServer.CMS.dll
    OriginalFilename: SiteServer.CMS.dll
    ProductVersion:   0.0.0.0
    FileVersion:      0.0.0.0
    FileDescription:   
LegalCopyright:  
```

第一行的这个`72720000` 就是起始地址，导出程序集

```
0:069> !savemodule 72720000 F:\tmp\SiteServer.CMS.dll
4 sections in file
section 0 - VA=2000, VASize=22e8d4, FileAddr=400, FileSize=22ea00
section 1 - VA=232000, VASize=a5, FileAddr=22ee00, FileSize=200
section 2 - VA=234000, VASize=2bc, FileAddr=22f000, FileSize=400
section 3 - VA=236000, VASize=c, FileAddr=22f400, FileSize=200
```

拿到程序集后，用`ILSpy`分析，但到很多像混淆过的代码，不过也可能是使用了动态类或者lambda表达式造成的。

根据之前的`!clrstack`栈信息

```
0000000016b0b490 000007fe9ad2eba8 SiteServer.CMS.BackgroundPages.BackgroundContent.Page_Load(System.Object, System.EventArgs)
```

直接找到这个页面的源代码的绑定Repeater的代码

```
public void Page_Load(object sender, EventArgs E)
{
    ...
    this.rptContents.ItemDataBound += new RepeaterItemEventHandler(this.d);
    ...
}
```

进去`this.d`，对照栈

```
0000000016b0b170 000007fe9b0c3b8e SiteServer.CMS.Core.ContentUtility.GetCommandItemRowsHtml(BaiRong.Model.ETableStyle, SiteServer.CMS.Model.PublishmentSystemInfo, SiteServer.CMS.Model.NodeInfo, BaiRong.Model.ContentInfo, System.String)
0000000016b0b230 000007fe9b0c1270 SiteServer.CMS.BackgroundPages.BackgroundContent.rptContents_ItemDataBound(System.Object, System.Web.UI.WebControls.RepeaterItemEventArgs)
0000000016b0b300 000007fedab97fc7 System.Web.UI.WebControls.Repeater.CreateItem(Int32, System.Web.UI.WebControls.ListItemType, Boolean, System.Object)
0000000016b0b350 000007feda1ec763 System.Web.UI.WebControls.Repeater.CreateControlHierarchy(Boolean)
0000000016b0b3f0 000007feda1eca94 System.Web.UI.WebControls.Repeater.OnDataBinding(System.EventArgs)
0000000016b0b430 000007fe9ad3cb06 BaiRong.Controls.SqlPager.DataBind()
```

看对应的源代码

```
private void d(object obj, RepeaterItemEventArgs repeaterItemEventArgs)
{
    ...
    literal5.Text = ContentUtility.GetCommandItemRowsHtml(this.E, base.PublishmentSystemInfo, this.n, contentInfo, this.y());
    ...
}

```

进去GetCommandItemRowsHtml

```
0000000016b0b110 000007fe9a942c55 SiteServer.CMS.Core.Security.AdminUtility.HasChannelPermissions(Int32, Int32, System.String[])
```

对应的源代码

```
public static string GetCommandItemRowsHtml(ETableStyle tableStyle, PublishmentSystemInfo publishmentSystemInfo, NodeInfo nodeInfo, ContentInfo contentInfo, string pageUrl)
{
    ...
    if (publishmentSystemInfo.Additional.IsContentCommentable && enumType != EContentModelType.Job && AdminUtility.HasChannelPermissions(publishmentSystemInfo.PublishmentSystemID, contentInfo.NodeID, new string[]
    {
        S.y(401990),
        S.y(402026)
    }))
    ...
}
```

继续深入

```
public static bool HasChannelPermissions(int publishmentSystemID, int nodeID, params string[] channelPermissionArray)
{
    if (nodeID == 0)
    {
        return false;
    }
    if (PermissionsManager.Current.IsSystemAdministrator)
    {
        return true;
    }
```

继续进入Current

```
0000000016b0b0c0 000007fe9a935bcf BaiRong.Core.AdminManager.get_Current()
```

看到对应的源代码

```
public static AdministratorWithPermissions Current
{
    get
    {
        PermissionsManager permissionsManager = new PermissionsManager(AdminManager.Current.UserName);
        return permissionsManager.Permissions;
    }
}

public static AdministratorInfo Current
{
    get
    {
        AdministratorInfo administratorInfo = null;
        try
        {
            administratorInfo = AdminManager.GetAdminInfo(BaiRongDataProvider.AdministratorDAO.UserName);
        }
```

到了这里，就有些麻烦了，因为`AdministratorDAO`在其他程序集

```
public static IAdministratorDAO AdministratorDAO
{
    get
    {
        if (BaiRongDataProvider.U == null)
        {
            string typeName = BaiRongDataProvider.e + OA.x(114234);
            BaiRongDataProvider.U = (IAdministratorDAO)Assembly.Load(OA.x(114272)).CreateInstance(typeName);
        }
        return BaiRongDataProvider.U;
    }
}
```

所以，按照刚才到出程序集的方法，导出这个IP所在的程序集

```
0000000016b0b070 000007fe9a92b815 BaiRong.Provider.Data.SqlServer.AdministratorDAO.get_UserName()
```

同样的方法，通过`IP`找到模块

```
0:069> !ip2md 000007fe9a92b815 
MethodDesc:   000007fe9aad7a08
Method Name:  BaiRong.Provider.Data.SqlServer.AdministratorDAO.get_UserName()
Class:        000007fe9ab122d8
MethodTable:  000007fe9aad7a58
mdToken:      000000000600005f
Module:       000007fe9a8f0f60
IsJitted:     yes
CodeAddr:     000007fe9a92b800
Transparency: Critical
0:069> !DumpModule /d 000007fe9a8f0f60
Name:       C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Temporary ASP.NET Files\root\ddad0b42\624b781\assembly\dl3\537f3f91\0076ad61_445ed101\BaiRong.Provider.dll
Attributes: PEFile 
Assembly:   0000000010e3c540
LoaderHeap:              0000000000000000
TypeDefToMethodTableMap: 000007fe9a9021c0
TypeRefToMethodTableMap: 000007fe9a902470
MethodDefToDescMap:      000007fe9a902f50
FieldDefToDescMap:       000007fe9a904b70
MemberRefToDescMap:      0000000000000000
FileReferencesMap:       000007fe9a905bc0
AssemblyReferencesMap:   000007fe9a905bc8
MetaData start address:  0000000063380a3c (347808 bytes)
```

看到是这个程序集`BaiRong.Provider.dll`，从刚才保存出来到记事本的内容中搜索`\BaiRong.Provider.dll`检索，拿到内存地址

```
00000000`73740000 00000000`737cc000   BaiRong_Provider_73740000   (deferred)             
    Image path: C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Temporary ASP.NET Files\api\18830d0c\1a46d8d6\assembly\dl3\efc26c16\7d50cf57_1c96d001\BaiRong.Provider.dll
    Image name: BaiRong.Provider.dll
    Browse all global symbols  functions  data
    Has CLR image header, track-debug-data flag not set
    Timestamp:        Thu May 14 22:41:38 2015 (5554B422)
    CheckSum:         00000000
    ImageSize:        0008C000
    File version:     0.0.0.0
    Product version:  0.0.0.0
    File flags:       0 (Mask 3F)
    File OS:          4 Unknown Win32
    File type:        2.0 Dll
    File date:        00000000.00000000
    Translations:     0000.04b0
    InternalName:     BaiRong.Provider.dll
    OriginalFilename: BaiRong.Provider.dll
    ProductVersion:   0.0.0.0
    FileVersion:      0.0.0.0
    FileDescription:   
LegalCopyright:    
```

导出到文件

```
0:069> !savemodule 73740000 F:\tmp\BaiRong.Provider.dll
4 sections in file
section 0 - VA=2000, VASize=83734, FileAddr=400, FileSize=83800
section 1 - VA=86000, VASize=89, FileAddr=83c00, FileSize=200
section 2 - VA=88000, VASize=2c4, FileAddr=83e00, FileSize=400
section 3 - VA=8a000, VASize=c, FileAddr=84200, FileSize=200
```

打开`ILSpy`，直接定位到`BaiRong.Provider.Data.SqlServer.AdministratorDAO.UserName`的属性

```
0000000016b0b070 000007fe9a92b815 BaiRong.Provider.Data.SqlServer.AdministratorDAO.get_UserName()
```

对应的源代码

```
// BaiRong.Provider.Data.SqlServer.AdministratorDAO
public string UserName
{
    get
    {
        string text = PageUtils.UrlDecode(CookieUtils.GetCookie(AdminAuthConfig.UserNameCookieName));
        if (!string.IsNullOrEmpty(text))
        {
            text = text.Replace(k.w(16730), string.Empty);
        }
        else
        {
            text = string.Empty;
        }
        return text;
    }
}
```

进去`GetCookie`

```
0000000016b0b030 000007fe9a925a66 BaiRong.Core.RuntimeUtils.DecryptStringByTranslate(System.String)
```

发现了对应的源代码

```
public static string GetCookie(string name)
{
    if (CookieUtils.IsExists(name))
    {
        string value = HttpContext.Current.Request.Cookies[name].Value;
        return RuntimeUtils.DecryptStringByTranslate(value);
    }
    return string.Empty;
}
```

以及这个

```
0000000016b0afc0 000007fe9a927aae BaiRong.Core.Cryptography.DESEncryptor.DesDecrypt()
```

对应的源代码

```
// BaiRong.Core.RuntimeUtils
public static string DecryptStringByTranslate(string inputString)
{
    if (string.IsNullOrEmpty(inputString))
    {
        return string.Empty;
    }
    inputString = inputString.Replace(OA.x(259360), OA.x(259374)).Replace(OA.x(259380), OA.x(259400)).Replace(OA.x(259406), OA.x(259420)).Replace(OA.x(259426), OA.x(259450)).Replace(OA.x(259456), OA.x(259474)).Replace(OA.x(259480), OA.x(259498));
    DESEncryptor dESEncryptor = new DESEncryptor();
    dESEncryptor.InputString = inputString;
    dESEncryptor.DecryptKey = ConfigManager.Cipherkey;
    dESEncryptor.DesDecrypt();
    return dESEncryptor.OutString;
}
```

到了这里，找到是`DESEncryptor`这个类了。但是这个`DESEncryptor`并不是微软自带的类，所以有必要进去看看。

继续导出程序集，先拿到`IP`

```
0000000016b0afc0 000007fe9a927aae BaiRong.Core.Cryptography.DESEncryptor.DesDecrypt()
```

同样方法

```
0:069> !ip2md 000007fe9a927aae 
MethodDesc:   000007fe9aad1628
Method Name:  BaiRong.Core.Cryptography.DESEncryptor.DesDecrypt()
Class:        000007fe9aaccec8
MethodTable:  000007fe9aad1680
mdToken:      0000000006000af1
Module:       000007fe9a8f61a8
IsJitted:     yes
CodeAddr:     000007fe9a927980
Transparency: Critical
0:069> !DumpModule /d 000007fe9a8f61a8
Name:       C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Temporary ASP.NET Files\root\ddad0b42\624b781\assembly\dl3\bcc1e7ad\0003165b_325ed101\BaiRong.Core.dll
Attributes: PEFile 
Assembly:   0000000010e3be80
LoaderHeap:              0000000000000000
TypeDefToMethodTableMap: 000007fe9a980020
TypeRefToMethodTableMap: 000007fe9a9827b0
MethodDefToDescMap:      000007fe9a9837e0
FieldDefToDescMap:       000007fe9a9989c0
MemberRefToDescMap:      0000000000000000
FileReferencesMap:       000007fe9a9a8e40
AssemblyReferencesMap:   000007fe9a9a8e48
MetaData start address:  000000005a8b566c (1035128 bytes)
```

检索`\BaiRong.Core.dll`拿到

```
00000000`72960000 00000000`72b94000   BaiRong_Core_72960000   (deferred)             
    Image path: C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Temporary ASP.NET Files\api\18830d0c\1a46d8d6\assembly\dl3\37494ca0\9b28c657_1c96d001\BaiRong.Core.dll
    Image name: BaiRong.Core.dll
    Browse all global symbols  functions  data
    Has CLR image header, track-debug-data flag not set
    Timestamp:        Thu May 14 22:41:24 2015 (5554B414)
    CheckSum:         00000000
    ImageSize:        00234000
    File version:     1.0.5616.18283
    Product version:  1.0.5616.18283
    File flags:       0 (Mask 3F)
    File OS:          4 Unknown Win32
    File type:        2.0 Dll
    File date:        00000000.00000000
    Translations:     0000.04b0
    InternalName:     BaiRong.Core.dll
    OriginalFilename: BaiRong.Core.dll
    ProductVersion:   1.0.5616.18283
    FileVersion:      1.0.5616.18283
    FileDescription:   
LegalCopyright:    
```

导出程序集

```
0:069> !savemodule 72960000  F:\tmp\BaiRong.Core.dll
4 sections in file
section 0 - VA=2000, VASize=20fe64, FileAddr=400, FileSize=210000
section 1 - VA=212000, VASize=1dc14, FileAddr=210400, FileSize=1de00
section 2 - VA=230000, VASize=2e4, FileAddr=22e200, FileSize=400
section 3 - VA=232000, VASize=c, FileAddr=22e600, FileSize=200
```

打开`ILSpy`

```
0000000016b0af70 000007fedd9ab3d4 System.Security.Cryptography.DESCryptoServiceProvider.CreateDecryptor(Byte[], Byte[])
```

到此看到微软的代码了

```
// BaiRong.Core.Cryptography.DESEncryptor
public void DesDecrypt()
{
    if (!OA.c(1096))
    {
        byte[] rgbIV = new byte[]
        {
            18,
            52,
            86,
            120,
            144,
            171,
            205,
            239
        };
        byte[] array = new byte[this.c.Length];
        try
        {
            byte[] bytes = Encoding.UTF8.GetBytes(this.j.Substring(0, 8));
            DESCryptoServiceProvider dESCryptoServiceProvider = new DESCryptoServiceProvider();
            array = Convert.FromBase64String(this.c);
            MemoryStream memoryStream = new MemoryStream();
            CryptoStream cryptoStream = new CryptoStream(memoryStream, dESCryptoServiceProvider.CreateDecryptor(bytes, rgbIV), CryptoStreamMode.Write);
            cryptoStream.Write(array, 0, array.Length);
            cryptoStream.FlushFinalBlock();
            Encoding encoding = new UTF8Encoding();
            this.w = encoding.GetString(memoryStream.ToArray());
        }
        catch (Exception ex)
        {
            this.i = ex.Message;
        }
    }
}
```

进去`DESCryptoServiceProvider`

```
0000000016b0aea0 000007fedd9ab85c System.Security.Cryptography.DESCryptoServiceProvider._NewEncryptor(Byte[], System.Security.Cryptography.CipherMode, Byte[], Int32, System.Security.Cryptography.CryptoAPITransformMode)
```

看微软如何实现

```
// System.Security.Cryptography.DESCryptoServiceProvider
[SecuritySafeCritical]
public override ICryptoTransform CreateEncryptor(byte[] rgbKey, byte[] rgbIV)
{
    if (DES.IsWeakKey(rgbKey))
    {
        throw new CryptographicException(Environment.GetResourceString("Cryptography_InvalidKey_Weak"), "DES");
    }
    if (DES.IsSemiWeakKey(rgbKey))
    {
        throw new CryptographicException(Environment.GetResourceString("Cryptography_InvalidKey_SemiWeak"), "DES");
    }
    return this._NewEncryptor(rgbKey, this.ModeValue, rgbIV, this.FeedbackSizeValue, CryptoAPITransformMode.Encrypt);
}
```

再进去

```
0000000016b0ade0 000007fedcfac684 System.Security.Cryptography.CryptoAPITransform..ctor(Int32, Int32, Int32[], System.Object[], Byte[], System.Security.Cryptography.PaddingMode, System.Security.Cryptography.CipherMode, Int32, Int32, Boolean, System.Security.Cryptography.CryptoAPITransformMode)
```

看到

```
// System.Security.Cryptography.DESCryptoServiceProvider
[SecurityCritical]
private ICryptoTransform _NewEncryptor(byte[] rgbKey, CipherMode mode, byte[] rgbIV, int feedbackSize, CryptoAPITransformMode encryptMode)
{
    return new CryptoAPITransform(26113, num, array, array2, rgbKey, this.PaddingValue, mode, this.BlockSizeValue, feedbackSize, false, encryptMode);
}

// System.Security.Cryptography.CryptoAPITransform
[SecurityCritical]
internal CryptoAPITransform(int algid, int cArgs, int[] rgArgIds, object[] rgArgValues, byte[] rgbKey, PaddingMode padding, CipherMode cipherChainingMode, int blockSize, int feedbackSize, bool useSalt, CryptoAPITransformMode encDecMode)
{
    this._safeProvHandle = Utils.AcquireProvHandle(new CspParameters(24));
    SafeKeyHandle invalidHandle = SafeKeyHandle.InvalidHandle;
    Utils._ImportBulkKey(this._safeProvHandle, algid, useSalt, this._rgbKey, ref invalidHandle);
    this._safeKeyHandle = invalidHandle;
}
```

再进去`AcquireProvHandle`，刚好对应到

```
0000000016b0ace8 0000000077a8db2a [HelperMethodFrame_2OBJ: 0000000016b0ace8] System.Security.Cryptography.Utils._AcquireCSP(System.Security.Cryptography.CspParameters, System.Security.Cryptography.SafeProvHandle ByRef)
```

是这段代码

```
// System.Security.Cryptography.Utils
[SecurityCritical]
internal static SafeProvHandle AcquireProvHandle(CspParameters parameters)
{
    if (parameters == null)
    {
        parameters = new CspParameters(24);
    }
    SafeProvHandle invalidHandle = SafeProvHandle.InvalidHandle;
    Utils._AcquireCSP(parameters, ref invalidHandle);
    return invalidHandle;
}
```

然后`!clrstack`的栈就没有信息了，到了操作系统执行，下面这些就看不懂了

```
0:069> kb
 # RetAddr           : Args to Child                                                           : Call Site
00 00000000`77933e6c : 00000000`000e0298 00000000`77a8f9b8 00000000`0000000a 00000000`00000016 : ntdll!ZwQueryValueKey+0xa
01 00000000`77934538 : 00000000`16b0a838 00000000`16b0a8f0 00000000`16b0a901 00000000`00000014 : kernel32!LocalBaseRegQueryValue+0x17c
02 000007fe`fcff4340 : 00000000`00001fa0 00000000`00000000 00000000`00000021 00000000`16b0a918 : kernel32!RegQueryValueExA+0x138
03 000007fe`fcff3c66 : 00000000`16b0aa70 00000000`022812b0 00000000`16b0ab01 000007fe`00000035 : cryptsp!CryptAcquireContextA+0x698
04 000007fe`fdcf049d : 00000000`f0000000 00000000`00000018 00000000`00000000 000007fe`dd1994f8 : cryptsp!CryptAcquireContextW+0xce
05 000007fe`f9fa5815 : 00000000`00000018 00000000`022812b0 00000945`45c167b8 000007fe`fa55e7e8 : advapi32!CryptAcquireContextWStub+0x11
06 000007fe`f9fa577f : 00000000`00000000 00000000`16b0abd9 00000000`00000000 00000000`1a4951c0 : clr!CryptoHelper::WszCryptAcquireContext_SO_TOLERANT+0x6d
07 000007fe`f9fa5bab : 00000004`00b89690 00000000`00000001 00000000`1a4951c0 00000000`00000000 : clr!COMCryptography::OpenCSP+0x192
08 000007fe`dcfac684 : 00000004`00b892a0 00000004`00b89648 00000004`00b89558 00000004`00b894a0 : clr!COMCryptography::_AcquireCSP+0x133
```

虽然看不懂，但是可以得出结论，进行`DES`加密、解密是需要消耗cpu资源并加`thinlock`锁，如果有大量频繁的`DES`处理，那么cpu会受不了。

不过，由于这次dump时只有一个正在运行的线程，所以不排除有其他问题没有反映出来，可能需要同事再提供几个dump，继续分析其他问题了。

* * * * * * * * * * * * * * * * * * * * * * * * * * * * *

今天再分析了另一个`dump`，发现了新线索

```
0:000> !threadpool
CPU utilization: 88%
Worker Thread: Total: 18 Running: 5 Idle: 3 MaxLimit: 32767 MinLimit: 8
Work Request in Queue: 0
--------------------------------------
Number of Timers: 3
--------------------------------------
Completion Port Thread:Total: 3 Free: 3 MaxFree: 16 CurrentLimit: 3 MaxLimit: 1000 MinLimit: 8
```

cpu占用率更高了，达到88%，而且正在运行的线程郑家到5个。不过老实话，5个线程真的不多。看下这几个线程正在做什么

```
0:000> ~* e !clrstack
省略几百行... 
OS Thread Id: 0x2c604 (60)
        Child SP               IP Call Site
000000001ae5afd8 0000000077a8d9fa [InlinedCallFrame: 000000001ae5afd8] .SNIReadSyncOverAsync(SNI_ConnWrapper*, SNI_Packet**, Int32)
000000001ae5afd8 000007fed6c83ab1 [InlinedCallFrame: 000000001ae5afd8] .SNIReadSyncOverAsync(SNI_ConnWrapper*, SNI_Packet**, Int32)
000000001ae5afb0 000007fed6c83ab1 *** WARNING: Unable to verify checksum for System.Data.ni.dll
DomainNeutralILStubClass.IL_STUB_PInvoke(SNI_ConnWrapper*, SNI_Packet**, Int32)
000000001ae5b080 000007fed6c6a35f SNINativeMethodWrapper.SNIReadSyncOverAsync(System.Runtime.InteropServices.SafeHandle, IntPtr ByRef, Int32)
000000001ae5b0f0 000007fed6c6a07e System.Data.SqlClient.TdsParserStateObject.ReadSniSyncOverAsync()
000000001ae5b1a0 000007fed6c69f90 System.Data.SqlClient.TdsParserStateObject.TryReadNetworkPacket()
000000001ae5b1e0 000007fed6c6a760 System.Data.SqlClient.TdsParserStateObject.TryPrepareBuffer()
000000001ae5b210 000007fed6c6d6a2 System.Data.SqlClient.TdsParserStateObject.TryReadByte(Byte ByRef)
000000001ae5b250 000007fed6c6c6d6 System.Data.SqlClient.TdsParser.TryRun(System.Data.SqlClient.RunBehavior, System.Data.SqlClient.SqlCommand, System.Data.SqlClient.SqlDataReader, System.Data.SqlClient.BulkCopySimpleResultSet, System.Data.SqlClient.TdsParserStateObject, Boolean ByRef)
000000001ae5b3f0 000007fed6c792ed System.Data.SqlClient.SqlDataReader.TryConsumeMetaData()
000000001ae5b460 000007fed6c79276 System.Data.SqlClient.SqlDataReader.get_MetaData()
000000001ae5b4b0 000007fed6c739ec System.Data.SqlClient.SqlCommand.FinishExecuteReader(System.Data.SqlClient.SqlDataReader, System.Data.SqlClient.RunBehavior, System.String)
000000001ae5b530 000007fed6c738b7 System.Data.SqlClient.SqlCommand.RunExecuteReaderTds(System.Data.CommandBehavior, System.Data.SqlClient.RunBehavior, Boolean, Boolean, Int32, System.Threading.Tasks.Task ByRef, Boolean, System.Data.SqlClient.SqlDataReader)
000000001ae5b630 000007fed6c72d2a System.Data.SqlClient.SqlCommand.RunExecuteReader(System.Data.CommandBehavior, System.Data.SqlClient.RunBehavior, Boolean, System.String, System.Threading.Tasks.TaskCompletionSource`1, Int32, System.Threading.Tasks.Task ByRef, Boolean)
000000001ae5b700 000007fed6c72ac8 System.Data.SqlClient.SqlCommand.RunExecuteReader(System.Data.CommandBehavior, System.Data.SqlClient.RunBehavior, Boolean, System.String)
000000001ae5b790 000007fed6c71930 System.Data.SqlClient.SqlCommand.ExecuteReader(System.Data.CommandBehavior, System.String)
000000001ae5b850 000007fed6c7157b System.Data.SqlClient.SqlCommand.ExecuteDbDataReader(System.Data.CommandBehavior)
000000001ae5b8b0 000007fed70c4f11 System.Data.Common.DbCommand.System.Data.IDbCommand.ExecuteReader()
000000001ae5b8e0 000007fe9a926e4f *** WARNING: Unable to verify checksum for BaiRong.Core.dll
*** ERROR: Module load completed but symbols could not be loaded for BaiRong.Core.dll
BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbCommand, AdoConnectionOwnership)
000000001ae5b940 000007fe9a926aa0 BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbConnection, System.Data.IDbTransaction, System.Data.CommandType, System.String, System.Data.IDataParameter[], AdoConnectionOwnership)
000000001ae5b9e0 000007fe9a940026 BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbConnection, System.Data.CommandType, System.String, System.Data.IDataParameter[])
000000001ae5ba30 000007fe9a93ffd8 BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbConnection, System.Data.CommandType, System.String)
000000001ae5ba70 000007fe9a93fcda *** WARNING: Unable to verify checksum for BaiRong.Provider.dll
*** ERROR: Module load completed but symbols could not be loaded for BaiRong.Provider.dll
BaiRong.Provider.Data.SqlServer.DatabaseDAO.GetIntResult(System.String)
000000001ae5bae0 000007fe9b233e5b *** WARNING: Unable to verify checksum for SiteServer.CMS.dll
*** ERROR: Module load completed but symbols could not be loaded for SiteServer.CMS.dll
SiteServer.CMS.Provider.Data.SqlServer.BackgroundContentDAO.GetCountArrayListUnChecked(Boolean, System.Collections.Generic.List`1, System.Collections.ArrayList, System.String)
000000001ae5bbc0 000007fe9b233360 SiteServer.CMS.Core.CheckManager.GetUserCountArrayListUnChecked(System.String)
000000001ae5bc30 000007fe9b23305e SiteServer.CMS.Core.CheckManager.GetUserCountArrayListUnChecked()
000000001ae5bc90 000007fe9b23287d SiteServer.CMS.BackgroundPages.BackgroundRight.LoadCheckRepeater()
000000001ae5bd80 000007fe9b231a87 SiteServer.CMS.BackgroundPages.BackgroundRight.Page_Load(System.Object, System.EventArgs)
000000001ae5bdf0 000007feda19d507 *** WARNING: Unable to verify checksum for System.Web.ni.dll
System.Web.UI.Control.LoadRecursive()
000000001ae5be40 000007feda1be1ea System.Web.UI.Page.ProcessRequestMain(Boolean, Boolean)
000000001ae5bf00 000007feda1bd469 System.Web.UI.Page.ProcessRequest(Boolean, Boolean)
000000001ae5bf70 000007feda1bd2c7 System.Web.UI.Page.ProcessRequest()
000000001ae5c010 000007feda1bb9f3 System.Web.UI.Page.ProcessRequest(System.Web.HttpContext)
000000001ae5c060 000007feda1c62a1 System.Web.HttpApplication+CallHandlerExecutionStep.System.Web.HttpApplication.IExecutionStep.Execute()
000000001ae5c140 000007feda18d495 System.Web.HttpApplication.ExecuteStep(IExecutionStep, Boolean ByRef)
000000001ae5c1e0 000007feda1aabda System.Web.HttpApplication+PipelineStepManager.ResumeSteps(System.Exception)
000000001ae5c330 000007feda18d6a3 System.Web.HttpApplication.BeginProcessRequestNotification(System.Web.HttpContext, System.AsyncCallback)
000000001ae5c380 000007feda1875de System.Web.HttpRuntime.ProcessRequestNotificationPrivate(System.Web.Hosting.IIS7WorkerRequest, System.Web.HttpContext)
000000001ae5c420 000007feda190561 System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
000000001ae5c630 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
000000001ae5c680 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
000000001ae5ce68 000007fef9eeb42e [InlinedCallFrame: 000000001ae5ce68] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
000000001ae5ce68 000007feda2395eb [InlinedCallFrame: 000000001ae5ce68] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
000000001ae5ce40 000007feda2395eb DomainNeutralILStubClass.IL_STUB_PInvoke(IntPtr, System.Web.RequestNotificationStatus ByRef)
000000001ae5cf10 000007feda19074f System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
000000001ae5d120 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
000000001ae5d170 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
000000001ae5eac8 000007fef9eeb42e [InlinedCallFrame: 000000001ae5eac8] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
000000001ae5eac8 000007feda2395eb [InlinedCallFrame: 000000001ae5eac8] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
000000001ae5eaa0 000007feda2395eb DomainNeutralILStubClass.IL_STUB_PInvoke(IntPtr, System.Web.RequestNotificationStatus ByRef)
000000001ae5eb70 000007feda19074f System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
000000001ae5ed80 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
000000001ae5edd0 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
000000001ae5efc8 000007fef9eeb683 [ContextTransitionFrame: 000000001ae5efc8] 
OS Thread Id: 0x2ddb8 (61)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2d93c (62)
        Child SP               IP Call Site
0000000011d0ac18 0000000077a8d9fa [InlinedCallFrame: 0000000011d0ac18] .SNIReadSyncOverAsync(SNI_ConnWrapper*, SNI_Packet**, Int32)
0000000011d0ac18 000007fed6c83ab1 [InlinedCallFrame: 0000000011d0ac18] .SNIReadSyncOverAsync(SNI_ConnWrapper*, SNI_Packet**, Int32)
0000000011d0abf0 000007fed6c83ab1 DomainNeutralILStubClass.IL_STUB_PInvoke(SNI_ConnWrapper*, SNI_Packet**, Int32)
0000000011d0acc0 000007fed6c6a35f SNINativeMethodWrapper.SNIReadSyncOverAsync(System.Runtime.InteropServices.SafeHandle, IntPtr ByRef, Int32)
0000000011d0ad30 000007fed6c6a07e System.Data.SqlClient.TdsParserStateObject.ReadSniSyncOverAsync()
0000000011d0ade0 000007fed6c69f90 System.Data.SqlClient.TdsParserStateObject.TryReadNetworkPacket()
0000000011d0ae20 000007fed6c6a760 System.Data.SqlClient.TdsParserStateObject.TryPrepareBuffer()
0000000011d0ae50 000007fed6c6d6a2 System.Data.SqlClient.TdsParserStateObject.TryReadByte(Byte ByRef)
0000000011d0ae90 000007fed6c6c6d6 System.Data.SqlClient.TdsParser.TryRun(System.Data.SqlClient.RunBehavior, System.Data.SqlClient.SqlCommand, System.Data.SqlClient.SqlDataReader, System.Data.SqlClient.BulkCopySimpleResultSet, System.Data.SqlClient.TdsParserStateObject, Boolean ByRef)
0000000011d0b030 000007fed6c792ed System.Data.SqlClient.SqlDataReader.TryConsumeMetaData()
0000000011d0b0a0 000007fed6c79276 System.Data.SqlClient.SqlDataReader.get_MetaData()
0000000011d0b0f0 000007fed6c739ec System.Data.SqlClient.SqlCommand.FinishExecuteReader(System.Data.SqlClient.SqlDataReader, System.Data.SqlClient.RunBehavior, System.String)
0000000011d0b170 000007fed6c738b7 System.Data.SqlClient.SqlCommand.RunExecuteReaderTds(System.Data.CommandBehavior, System.Data.SqlClient.RunBehavior, Boolean, Boolean, Int32, System.Threading.Tasks.Task ByRef, Boolean, System.Data.SqlClient.SqlDataReader)
0000000011d0b270 000007fed6c72d2a System.Data.SqlClient.SqlCommand.RunExecuteReader(System.Data.CommandBehavior, System.Data.SqlClient.RunBehavior, Boolean, System.String, System.Threading.Tasks.TaskCompletionSource`1, Int32, System.Threading.Tasks.Task ByRef, Boolean)
0000000011d0b340 000007fed6c72ac8 System.Data.SqlClient.SqlCommand.RunExecuteReader(System.Data.CommandBehavior, System.Data.SqlClient.RunBehavior, Boolean, System.String)
0000000011d0b3d0 000007fed6c71930 System.Data.SqlClient.SqlCommand.ExecuteReader(System.Data.CommandBehavior, System.String)
0000000011d0b490 000007fed6c7157b System.Data.SqlClient.SqlCommand.ExecuteDbDataReader(System.Data.CommandBehavior)
0000000011d0b4f0 000007fed70c4f11 System.Data.Common.DbCommand.System.Data.IDbCommand.ExecuteReader()
0000000011d0b520 000007fe9a926e4f BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbCommand, AdoConnectionOwnership)
0000000011d0b580 000007fe9a926aa0 BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbConnection, System.Data.IDbTransaction, System.Data.CommandType, System.String, System.Data.IDataParameter[], AdoConnectionOwnership)
0000000011d0b620 000007fe9a940026 BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbConnection, System.Data.CommandType, System.String, System.Data.IDataParameter[])
0000000011d0b670 000007fe9a93ffd8 BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbConnection, System.Data.CommandType, System.String)
0000000011d0b6b0 000007fe9a93fcda BaiRong.Provider.Data.SqlServer.DatabaseDAO.GetIntResult(System.String)
0000000011d0b720 000007fe9b233e5b SiteServer.CMS.Provider.Data.SqlServer.BackgroundContentDAO.GetCountArrayListUnChecked(Boolean, System.Collections.Generic.List`1, System.Collections.ArrayList, System.String)
0000000011d0b800 000007fe9b233360 SiteServer.CMS.Core.CheckManager.GetUserCountArrayListUnChecked(System.String)
0000000011d0b870 000007fe9b23305e SiteServer.CMS.Core.CheckManager.GetUserCountArrayListUnChecked()
0000000011d0b8d0 000007fe9b23287d SiteServer.CMS.BackgroundPages.BackgroundRight.LoadCheckRepeater()
0000000011d0b9c0 000007fe9b231a87 SiteServer.CMS.BackgroundPages.BackgroundRight.Page_Load(System.Object, System.EventArgs)
0000000011d0ba30 000007feda19d507 System.Web.UI.Control.LoadRecursive()
0000000011d0ba80 000007feda1be1ea System.Web.UI.Page.ProcessRequestMain(Boolean, Boolean)
0000000011d0bb40 000007feda1bd469 System.Web.UI.Page.ProcessRequest(Boolean, Boolean)
0000000011d0bbb0 000007feda1bd2c7 System.Web.UI.Page.ProcessRequest()
0000000011d0bc50 000007feda1bb9f3 System.Web.UI.Page.ProcessRequest(System.Web.HttpContext)
0000000011d0bca0 000007feda1c62a1 System.Web.HttpApplication+CallHandlerExecutionStep.System.Web.HttpApplication.IExecutionStep.Execute()
0000000011d0bd80 000007feda18d495 System.Web.HttpApplication.ExecuteStep(IExecutionStep, Boolean ByRef)
0000000011d0be20 000007feda1aabda System.Web.HttpApplication+PipelineStepManager.ResumeSteps(System.Exception)
0000000011d0bf70 000007feda18d6a3 System.Web.HttpApplication.BeginProcessRequestNotification(System.Web.HttpContext, System.AsyncCallback)
0000000011d0bfc0 000007feda1875de System.Web.HttpRuntime.ProcessRequestNotificationPrivate(System.Web.Hosting.IIS7WorkerRequest, System.Web.HttpContext)
0000000011d0c060 000007feda190561 System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
0000000011d0c270 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
0000000011d0c2c0 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
0000000011d0caa8 000007fef9eeb42e [InlinedCallFrame: 0000000011d0caa8] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000011d0caa8 000007feda2395eb [InlinedCallFrame: 0000000011d0caa8] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000011d0ca80 000007feda2395eb DomainNeutralILStubClass.IL_STUB_PInvoke(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000011d0cb50 000007feda19074f System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
0000000011d0cd60 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
0000000011d0cdb0 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
0000000011d0e708 000007fef9eeb42e [InlinedCallFrame: 0000000011d0e708] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000011d0e708 000007feda2395eb [InlinedCallFrame: 0000000011d0e708] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000011d0e6e0 000007feda2395eb DomainNeutralILStubClass.IL_STUB_PInvoke(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000011d0e7b0 000007feda19074f System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
0000000011d0e9c0 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
0000000011d0ea10 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
0000000011d0ec08 000007fef9eeb683 [ContextTransitionFrame: 0000000011d0ec08] 
OS Thread Id: 0x2f3d4 (63)
        Child SP               IP Call Site
0000000016b0a798 0000000077a8d9fa [InlinedCallFrame: 0000000016b0a798] .SNIReadSyncOverAsync(SNI_ConnWrapper*, SNI_Packet**, Int32)
0000000016b0a798 000007fed6c83ab1 [InlinedCallFrame: 0000000016b0a798] .SNIReadSyncOverAsync(SNI_ConnWrapper*, SNI_Packet**, Int32)
0000000016b0a770 000007fed6c83ab1 DomainNeutralILStubClass.IL_STUB_PInvoke(SNI_ConnWrapper*, SNI_Packet**, Int32)
0000000016b0a840 000007fed6c6a35f SNINativeMethodWrapper.SNIReadSyncOverAsync(System.Runtime.InteropServices.SafeHandle, IntPtr ByRef, Int32)
0000000016b0a8b0 000007fed6c6a07e System.Data.SqlClient.TdsParserStateObject.ReadSniSyncOverAsync()
0000000016b0a960 000007fed6c69f90 System.Data.SqlClient.TdsParserStateObject.TryReadNetworkPacket()
0000000016b0a9a0 000007fed6c6a760 System.Data.SqlClient.TdsParserStateObject.TryPrepareBuffer()
0000000016b0a9d0 000007fed6c6d6a2 System.Data.SqlClient.TdsParserStateObject.TryReadByte(Byte ByRef)
0000000016b0aa10 000007fed6c6c6d6 System.Data.SqlClient.TdsParser.TryRun(System.Data.SqlClient.RunBehavior, System.Data.SqlClient.SqlCommand, System.Data.SqlClient.SqlDataReader, System.Data.SqlClient.BulkCopySimpleResultSet, System.Data.SqlClient.TdsParserStateObject, Boolean ByRef)
0000000016b0abb0 000007fed6c792ed System.Data.SqlClient.SqlDataReader.TryConsumeMetaData()
0000000016b0ac20 000007fed6c79276 System.Data.SqlClient.SqlDataReader.get_MetaData()
0000000016b0ac70 000007fed6c739ec System.Data.SqlClient.SqlCommand.FinishExecuteReader(System.Data.SqlClient.SqlDataReader, System.Data.SqlClient.RunBehavior, System.String)
0000000016b0acf0 000007fed6c738b7 System.Data.SqlClient.SqlCommand.RunExecuteReaderTds(System.Data.CommandBehavior, System.Data.SqlClient.RunBehavior, Boolean, Boolean, Int32, System.Threading.Tasks.Task ByRef, Boolean, System.Data.SqlClient.SqlDataReader)
0000000016b0adf0 000007fed6c72d2a System.Data.SqlClient.SqlCommand.RunExecuteReader(System.Data.CommandBehavior, System.Data.SqlClient.RunBehavior, Boolean, System.String, System.Threading.Tasks.TaskCompletionSource`1, Int32, System.Threading.Tasks.Task ByRef, Boolean)
0000000016b0aec0 000007fed6c72ac8 System.Data.SqlClient.SqlCommand.RunExecuteReader(System.Data.CommandBehavior, System.Data.SqlClient.RunBehavior, Boolean, System.String)
0000000016b0af50 000007fed6c71930 System.Data.SqlClient.SqlCommand.ExecuteReader(System.Data.CommandBehavior, System.String)
0000000016b0b010 000007fed6c7157b System.Data.SqlClient.SqlCommand.ExecuteDbDataReader(System.Data.CommandBehavior)
0000000016b0b070 000007fed70c4f11 System.Data.Common.DbCommand.System.Data.IDbCommand.ExecuteReader()
0000000016b0b0a0 000007fe9a926e4f BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbCommand, AdoConnectionOwnership)
0000000016b0b100 000007fe9a926aa0 BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbConnection, System.Data.IDbTransaction, System.Data.CommandType, System.String, System.Data.IDataParameter[], AdoConnectionOwnership)
0000000016b0b1a0 000007fe9a940026 BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbConnection, System.Data.CommandType, System.String, System.Data.IDataParameter[])
0000000016b0b1f0 000007fe9a93ffd8 BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbConnection, System.Data.CommandType, System.String)
0000000016b0b230 000007fe9a93fcda BaiRong.Provider.Data.SqlServer.DatabaseDAO.GetIntResult(System.String)
0000000016b0b2a0 000007fe9b233e5b SiteServer.CMS.Provider.Data.SqlServer.BackgroundContentDAO.GetCountArrayListUnChecked(Boolean, System.Collections.Generic.List`1, System.Collections.ArrayList, System.String)
0000000016b0b380 000007fe9b233360 SiteServer.CMS.Core.CheckManager.GetUserCountArrayListUnChecked(System.String)
0000000016b0b3f0 000007fe9b23305e SiteServer.CMS.Core.CheckManager.GetUserCountArrayListUnChecked()
0000000016b0b450 000007fe9b23287d SiteServer.CMS.BackgroundPages.BackgroundRight.LoadCheckRepeater()
0000000016b0b540 000007fe9b231a87 SiteServer.CMS.BackgroundPages.BackgroundRight.Page_Load(System.Object, System.EventArgs)
0000000016b0b5b0 000007feda19d507 System.Web.UI.Control.LoadRecursive()
0000000016b0b600 000007feda1be1ea System.Web.UI.Page.ProcessRequestMain(Boolean, Boolean)
0000000016b0b6c0 000007feda1bd469 System.Web.UI.Page.ProcessRequest(Boolean, Boolean)
0000000016b0b730 000007feda1bd2c7 System.Web.UI.Page.ProcessRequest()
0000000016b0b7d0 000007feda1bb9f3 System.Web.UI.Page.ProcessRequest(System.Web.HttpContext)
0000000016b0b820 000007feda1c62a1 System.Web.HttpApplication+CallHandlerExecutionStep.System.Web.HttpApplication.IExecutionStep.Execute()
0000000016b0b900 000007feda18d495 System.Web.HttpApplication.ExecuteStep(IExecutionStep, Boolean ByRef)
0000000016b0b9a0 000007feda1aabda System.Web.HttpApplication+PipelineStepManager.ResumeSteps(System.Exception)
0000000016b0baf0 000007feda18d6a3 System.Web.HttpApplication.BeginProcessRequestNotification(System.Web.HttpContext, System.AsyncCallback)
0000000016b0bb40 000007feda1875de System.Web.HttpRuntime.ProcessRequestNotificationPrivate(System.Web.Hosting.IIS7WorkerRequest, System.Web.HttpContext)
0000000016b0bbe0 000007feda190561 System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
0000000016b0bdf0 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
0000000016b0be40 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
0000000016b0c628 000007fef9eeb42e [InlinedCallFrame: 0000000016b0c628] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016b0c628 000007feda2395eb [InlinedCallFrame: 0000000016b0c628] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016b0c600 000007feda2395eb DomainNeutralILStubClass.IL_STUB_PInvoke(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016b0c6d0 000007feda19074f System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
0000000016b0c8e0 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
0000000016b0c930 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
0000000016b0e288 000007fef9eeb42e [InlinedCallFrame: 0000000016b0e288] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016b0e288 000007feda2395eb [InlinedCallFrame: 0000000016b0e288] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016b0e260 000007feda2395eb DomainNeutralILStubClass.IL_STUB_PInvoke(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016b0e330 000007feda19074f System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
0000000016b0e540 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
0000000016b0e590 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
0000000016b0e788 000007fef9eeb683 [ContextTransitionFrame: 0000000016b0e788] 
OS Thread Id: 0x31c48 (64)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2f6e4 (65)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2ffe8 (66)
        Child SP               IP Call Site
000000001374aec8 0000000077a8d9fa [InlinedCallFrame: 000000001374aec8] .SNIReadSyncOverAsync(SNI_ConnWrapper*, SNI_Packet**, Int32)
000000001374aec8 000007fed6c83ab1 [InlinedCallFrame: 000000001374aec8] .SNIReadSyncOverAsync(SNI_ConnWrapper*, SNI_Packet**, Int32)
000000001374aea0 000007fed6c83ab1 DomainNeutralILStubClass.IL_STUB_PInvoke(SNI_ConnWrapper*, SNI_Packet**, Int32)
000000001374af70 000007fed6c6a35f SNINativeMethodWrapper.SNIReadSyncOverAsync(System.Runtime.InteropServices.SafeHandle, IntPtr ByRef, Int32)
000000001374afe0 000007fed6c6a07e System.Data.SqlClient.TdsParserStateObject.ReadSniSyncOverAsync()
000000001374b090 000007fed6c69f90 System.Data.SqlClient.TdsParserStateObject.TryReadNetworkPacket()
000000001374b0d0 000007fed6c6a760 System.Data.SqlClient.TdsParserStateObject.TryPrepareBuffer()
000000001374b100 000007fed6c6d6a2 System.Data.SqlClient.TdsParserStateObject.TryReadByte(Byte ByRef)
000000001374b140 000007fed6c6c6d6 System.Data.SqlClient.TdsParser.TryRun(System.Data.SqlClient.RunBehavior, System.Data.SqlClient.SqlCommand, System.Data.SqlClient.SqlDataReader, System.Data.SqlClient.BulkCopySimpleResultSet, System.Data.SqlClient.TdsParserStateObject, Boolean ByRef)
000000001374b2e0 000007fed6c792ed System.Data.SqlClient.SqlDataReader.TryConsumeMetaData()
000000001374b350 000007fed6c79276 System.Data.SqlClient.SqlDataReader.get_MetaData()
000000001374b3a0 000007fed6c739ec System.Data.SqlClient.SqlCommand.FinishExecuteReader(System.Data.SqlClient.SqlDataReader, System.Data.SqlClient.RunBehavior, System.String)
000000001374b420 000007fed6c738b7 System.Data.SqlClient.SqlCommand.RunExecuteReaderTds(System.Data.CommandBehavior, System.Data.SqlClient.RunBehavior, Boolean, Boolean, Int32, System.Threading.Tasks.Task ByRef, Boolean, System.Data.SqlClient.SqlDataReader)
000000001374b520 000007fed6c72d2a System.Data.SqlClient.SqlCommand.RunExecuteReader(System.Data.CommandBehavior, System.Data.SqlClient.RunBehavior, Boolean, System.String, System.Threading.Tasks.TaskCompletionSource`1, Int32, System.Threading.Tasks.Task ByRef, Boolean)
000000001374b5f0 000007fed6c72ac8 System.Data.SqlClient.SqlCommand.RunExecuteReader(System.Data.CommandBehavior, System.Data.SqlClient.RunBehavior, Boolean, System.String)
000000001374b680 000007fed6c71930 System.Data.SqlClient.SqlCommand.ExecuteReader(System.Data.CommandBehavior, System.String)
000000001374b740 000007fed6c7157b System.Data.SqlClient.SqlCommand.ExecuteDbDataReader(System.Data.CommandBehavior)
000000001374b7a0 000007fed70c4f11 System.Data.Common.DbCommand.System.Data.IDbCommand.ExecuteReader()
000000001374b7d0 000007fe9a926e4f BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbCommand, AdoConnectionOwnership)
000000001374b830 000007fe9a926aa0 BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbConnection, System.Data.IDbTransaction, System.Data.CommandType, System.String, System.Data.IDataParameter[], AdoConnectionOwnership)
000000001374b8d0 000007fe9a940026 BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbConnection, System.Data.CommandType, System.String, System.Data.IDataParameter[])
000000001374b920 000007fe9a93ffd8 BaiRong.Core.Data.Helper.AdoHelper.ExecuteReader(System.Data.IDbConnection, System.Data.CommandType, System.String)
000000001374b960 000007fe9a93fcda BaiRong.Provider.Data.SqlServer.DatabaseDAO.GetIntResult(System.String)
000000001374b9d0 000007fe9b233e5b SiteServer.CMS.Provider.Data.SqlServer.BackgroundContentDAO.GetCountArrayListUnChecked(Boolean, System.Collections.Generic.List`1, System.Collections.ArrayList, System.String)
000000001374bab0 000007fe9b233360 SiteServer.CMS.Core.CheckManager.GetUserCountArrayListUnChecked(System.String)
000000001374bb20 000007fe9b23305e SiteServer.CMS.Core.CheckManager.GetUserCountArrayListUnChecked()
000000001374bb80 000007fe9b23287d SiteServer.CMS.BackgroundPages.BackgroundRight.LoadCheckRepeater()
000000001374bc70 000007fe9b231a87 SiteServer.CMS.BackgroundPages.BackgroundRight.Page_Load(System.Object, System.EventArgs)
000000001374bce0 000007feda19d507 System.Web.UI.Control.LoadRecursive()
000000001374bd30 000007feda1be1ea System.Web.UI.Page.ProcessRequestMain(Boolean, Boolean)
000000001374bdf0 000007feda1bd469 System.Web.UI.Page.ProcessRequest(Boolean, Boolean)
000000001374be60 000007feda1bd2c7 System.Web.UI.Page.ProcessRequest()
000000001374bf00 000007feda1bb9f3 System.Web.UI.Page.ProcessRequest(System.Web.HttpContext)
000000001374bf50 000007feda1c62a1 System.Web.HttpApplication+CallHandlerExecutionStep.System.Web.HttpApplication.IExecutionStep.Execute()
000000001374c030 000007feda18d495 System.Web.HttpApplication.ExecuteStep(IExecutionStep, Boolean ByRef)
000000001374c0d0 000007feda1aabda System.Web.HttpApplication+PipelineStepManager.ResumeSteps(System.Exception)
000000001374c220 000007feda18d6a3 System.Web.HttpApplication.BeginProcessRequestNotification(System.Web.HttpContext, System.AsyncCallback)
000000001374c270 000007feda1875de System.Web.HttpRuntime.ProcessRequestNotificationPrivate(System.Web.Hosting.IIS7WorkerRequest, System.Web.HttpContext)
000000001374c310 000007feda190561 System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
000000001374c520 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
000000001374c570 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
000000001374cd58 000007fef9eeb42e [InlinedCallFrame: 000000001374cd58] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
000000001374cd58 000007feda2395eb [InlinedCallFrame: 000000001374cd58] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
000000001374cd30 000007feda2395eb DomainNeutralILStubClass.IL_STUB_PInvoke(IntPtr, System.Web.RequestNotificationStatus ByRef)
000000001374ce00 000007feda19074f System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
000000001374d010 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
000000001374d060 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
000000001374e9b8 000007fef9eeb42e [InlinedCallFrame: 000000001374e9b8] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
000000001374e9b8 000007feda2395eb [InlinedCallFrame: 000000001374e9b8] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
000000001374e990 000007feda2395eb DomainNeutralILStubClass.IL_STUB_PInvoke(IntPtr, System.Web.RequestNotificationStatus ByRef)
000000001374ea60 000007feda19074f System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
000000001374ec70 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
000000001374ecc0 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
000000001374eeb8 000007fef9eeb683 [ContextTransitionFrame: 000000001374eeb8] 
省略几十行...
OS Thread Id: 0x2dfac (78)
        Child SP               IP Call Site
0000000016ebb488 0000000077a8db2a [HelperMethodFrame_2OBJ: 0000000016ebb488] System.Security.Cryptography.Utils._AcquireCSP(System.Security.Cryptography.CspParameters, System.Security.Cryptography.SafeProvHandle ByRef)
0000000016ebb580 000007fedcfac684 *** WARNING: Unable to verify checksum for mscorlib.ni.dll
System.Security.Cryptography.CryptoAPITransform..ctor(Int32, Int32, Int32[], System.Object[], Byte[], System.Security.Cryptography.PaddingMode, System.Security.Cryptography.CipherMode, Int32, Int32, Boolean, System.Security.Cryptography.CryptoAPITransformMode)
0000000016ebb640 000007fedd9ab85c System.Security.Cryptography.DESCryptoServiceProvider._NewEncryptor(Byte[], System.Security.Cryptography.CipherMode, Byte[], Int32, System.Security.Cryptography.CryptoAPITransformMode)
0000000016ebb710 000007fedd9ab3d4 System.Security.Cryptography.DESCryptoServiceProvider.CreateDecryptor(Byte[], Byte[])
0000000016ebb760 000007fe9a927aae BaiRong.Core.Cryptography.DESEncryptor.DesDecrypt()
0000000016ebb7d0 000007fe9a925a66 BaiRong.Core.RuntimeUtils.DecryptStringByTranslate(System.String)
0000000016ebb810 000007fe9a92b815 BaiRong.Provider.Data.SqlServer.AdministratorDAO.get_UserName()
0000000016ebb860 000007fe9a935bcf BaiRong.Core.AdminManager.get_Current()
0000000016ebb8b0 000007fe9b0deb55 SiteServer.CMS.Core.CheckManager.GetUserCheckLevel(SiteServer.CMS.Model.PublishmentSystemInfo, Int32)
0000000016ebb910 000007fe9b0deaba SiteServer.CMS.Core.CheckManager.GetUserCheckLevel(SiteServer.CMS.Model.PublishmentSystemInfo, Int32, Int32 ByRef)
0000000016ebb960 000007fe9b233cc6 SiteServer.CMS.Provider.Data.SqlServer.BackgroundContentDAO.GetCountArrayListUnChecked(Boolean, System.Collections.Generic.List`1, System.Collections.ArrayList, System.String)
0000000016ebba40 000007fe9b233360 SiteServer.CMS.Core.CheckManager.GetUserCountArrayListUnChecked(System.String)
0000000016ebbab0 000007fe9b23305e SiteServer.CMS.Core.CheckManager.GetUserCountArrayListUnChecked()
0000000016ebbb10 000007fe9b23287d SiteServer.CMS.BackgroundPages.BackgroundRight.LoadCheckRepeater()
0000000016ebbc00 000007fe9b231a87 SiteServer.CMS.BackgroundPages.BackgroundRight.Page_Load(System.Object, System.EventArgs)
0000000016ebbc70 000007feda19d507 System.Web.UI.Control.LoadRecursive()
0000000016ebbcc0 000007feda1be1ea System.Web.UI.Page.ProcessRequestMain(Boolean, Boolean)
0000000016ebbd80 000007feda1bd469 System.Web.UI.Page.ProcessRequest(Boolean, Boolean)
0000000016ebbdf0 000007feda1bd2c7 System.Web.UI.Page.ProcessRequest()
0000000016ebbe90 000007feda1bb9f3 System.Web.UI.Page.ProcessRequest(System.Web.HttpContext)
0000000016ebbee0 000007feda1c62a1 System.Web.HttpApplication+CallHandlerExecutionStep.System.Web.HttpApplication.IExecutionStep.Execute()
0000000016ebbfc0 000007feda18d495 System.Web.HttpApplication.ExecuteStep(IExecutionStep, Boolean ByRef)
0000000016ebc060 000007feda1aabda System.Web.HttpApplication+PipelineStepManager.ResumeSteps(System.Exception)
0000000016ebc1b0 000007feda18d6a3 System.Web.HttpApplication.BeginProcessRequestNotification(System.Web.HttpContext, System.AsyncCallback)
0000000016ebc200 000007feda1875de System.Web.HttpRuntime.ProcessRequestNotificationPrivate(System.Web.Hosting.IIS7WorkerRequest, System.Web.HttpContext)
0000000016ebc2a0 000007feda190561 System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
0000000016ebc4b0 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
0000000016ebc500 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
0000000016ebcce8 000007fef9eeb42e [InlinedCallFrame: 0000000016ebcce8] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016ebcce8 000007feda2395eb [InlinedCallFrame: 0000000016ebcce8] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016ebccc0 000007feda2395eb DomainNeutralILStubClass.IL_STUB_PInvoke(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016ebcd90 000007feda19074f System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
0000000016ebcfa0 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
0000000016ebcff0 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
0000000016ebe948 000007fef9eeb42e [InlinedCallFrame: 0000000016ebe948] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016ebe948 000007feda2395eb [InlinedCallFrame: 0000000016ebe948] System.Web.Hosting.UnsafeIISMethods.MgdIndicateCompletion(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016ebe920 000007feda2395eb DomainNeutralILStubClass.IL_STUB_PInvoke(IntPtr, System.Web.RequestNotificationStatus ByRef)
0000000016ebe9f0 000007feda19074f System.Web.Hosting.PipelineRuntime.ProcessRequestNotificationHelper(IntPtr, IntPtr, IntPtr, Int32)
0000000016ebec00 000007feda18ff92 System.Web.Hosting.PipelineRuntime.ProcessRequestNotification(IntPtr, IntPtr, IntPtr, Int32)
0000000016ebec50 000007feda8e5dc1 DomainNeutralILStubClass.IL_STUB_ReversePInvoke(Int64, Int64, Int64, Int32)
0000000016ebee48 000007fef9eeb683 [ContextTransitionFrame: 0000000016ebee48] 
OS Thread Id: 0x2e1e4 (79)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2ec68 (80)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2fd7c (81)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
```

除了一个线程还是上次分析的`DES`解密的问题外，其他4个线程毫无例外地都在执行同一个`SQL`查询，切换到任意一个线程，看引用了什么对象，找到正在执行的`SQL`

```
0:000> ~60s
ntdll!ZwWaitForSingleObject+0xa:
00000000`77a8d9fa c3              ret
0:060> !dso
OS Thread Id: 0x2c604 (60)
RSP/REG          Object           Name
r9               0000000480042e10 System.Object[]    (System.Security.SecureString[])
000000001AE5AF28 0000000480042ef0 System.Data.SqlClient.SNIHandle
000000001AE5AF40 0000000480042af0 System.Data.SqlClient.TdsParser
000000001AE5AF48 0000000300e48ac0 System.Data.SqlClient.TdsParser+<>c__DisplayClass8
000000001AE5AF50 0000000300e48918 System.Data.SqlClient.SqlDataReader
000000001AE5AF98 0000000480042af0 System.Data.SqlClient.TdsParser
000000001AE5B040 0000000480042af0 System.Data.SqlClient.TdsParser
000000001AE5B048 0000000300e48ac0 System.Data.SqlClient.TdsParser+<>c__DisplayClass8
000000001AE5B050 0000000300e48918 System.Data.SqlClient.SqlDataReader
000000001AE5B098 0000000480042af0 System.Data.SqlClient.TdsParser
000000001AE5B0D8 0000000480042ef0 System.Data.SqlClient.SNIHandle
000000001AE5B0F0 0000000480042ef0 System.Data.SqlClient.SNIHandle
000000001AE5B178 0000000480042b90 System.Data.SqlClient.TdsParserStateObject
000000001AE5B188 0000000480042b90 System.Data.SqlClient.TdsParserStateObject
000000001AE5B1A0 0000000480042b90 System.Data.SqlClient.TdsParserStateObject
000000001AE5B1A8 0000000480043188 System.Data.SqlClient.SNIPacket
000000001AE5B1D0 0000000480042b90 System.Data.SqlClient.TdsParserStateObject
000000001AE5B200 0000000480042b90 System.Data.SqlClient.TdsParserStateObject
000000001AE5B238 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B260 0000000480042ef0 System.Data.SqlClient.SNIHandle
000000001AE5B268 0000000480043188 System.Data.SqlClient.SNIPacket
000000001AE5B2D0 0000000480042b90 System.Data.SqlClient.TdsParserStateObject
000000001AE5B368 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B370 0000000300e48ac0 System.Data.SqlClient.TdsParser+<>c__DisplayClass8
000000001AE5B3A0 0000000300e48918 System.Data.SqlClient.SqlDataReader
000000001AE5B3C8 0000000300e48918 System.Data.SqlClient.SqlDataReader
000000001AE5B3D0 0000000300e48508 System.String    SELECT COUNT(*) AS TotalNum FROM au_Content WHERE (PublishmentSystemID = 32598 AND NodeID > 0 AND IsChecked = 'False' AND CheckedLevel IN (0,1,2,3,4))
000000001AE5B3D8 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B3F0 0000000480042af0 System.Data.SqlClient.TdsParser
000000001AE5B400 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B408 0000000300e48918 System.Data.SqlClient.SqlDataReader
000000001AE5B418 0000000480042b90 System.Data.SqlClient.TdsParserStateObject
000000001AE5B440 0000000480042af0 System.Data.SqlClient.TdsParser
000000001AE5B448 0000000300e48508 System.String    SELECT COUNT(*) AS TotalNum FROM au_Content WHERE (PublishmentSystemID = 32598 AND NodeID > 0 AND IsChecked = 'False' AND CheckedLevel IN (0,1,2,3,4))
000000001AE5B450 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B470 0000000480042b90 System.Data.SqlClient.TdsParserStateObject
000000001AE5B4B0 0000000300e48918 System.Data.SqlClient.SqlDataReader
000000001AE5B4D8 0000000300e48aa0 System.Data.SqlClient.TdsParser+<>c__DisplayClasse
000000001AE5B508 0000000480042a00 System.WeakReference
000000001AE5B510 0000000300e48508 System.String    SELECT COUNT(*) AS TotalNum FROM au_Content WHERE (PublishmentSystemID = 32598 AND NodeID > 0 AND IsChecked = 'False' AND CheckedLevel IN (0,1,2,3,4))
000000001AE5B530 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B538 0000000300e48918 System.Data.SqlClient.SqlDataReader
000000001AE5B540 0000000300e488e0 System.Data.SqlClient.SqlCommand+<>c__DisplayClass49
000000001AE5B550 0000000480042b90 System.Data.SqlClient.TdsParserStateObject
000000001AE5B568 00000004809b9150 System.Threading.ExecutionContext
000000001AE5B570 00000004809b91b8 System.Runtime.Remoting.Messaging.LogicalCallContext
000000001AE5B578 00000004809b9150 System.Threading.ExecutionContext
000000001AE5B5B8 0000000300e488e0 System.Data.SqlClient.SqlCommand+<>c__DisplayClass49
000000001AE5B5E8 00000004809c4678 System.String    au_Content
000000001AE5B630 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B638 0000000500056be8 System.String    ExecuteReader
000000001AE5B6A8 0000000480042af0 System.Data.SqlClient.TdsParser
000000001AE5B6D8 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B700 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B720 0000000500056be8 System.String    ExecuteReader
000000001AE5B770 0000000500056be8 System.String    ExecuteReader
000000001AE5B7A0 00000004800406a0 System.Data.ProviderBase.DbConnectionPool
000000001AE5B7A8 0000000300e48760 System.Data.ProviderBase.DbConnectionFactory+<>c__DisplayClass4
000000001AE5B7B0 0000000500056be8 System.String    ExecuteReader
000000001AE5B800 0000000480042af0 System.Data.SqlClient.TdsParser
000000001AE5B820 00000004809c4678 System.String    au_Content
000000001AE5B828 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B840 0000000500056b28 System.String    <sc.SqlCommand.ExecuteDbDataReader|API|Correlation> ObjectID%d#, ActivityID %ls
000000001AE5B850 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B860 00000005000563e8 System.String    <prov.DbConnectionHelper.CreateDbCommand|API> %d#
000000001AE5B868 0000000300e48650 System.Data.SqlClient.SqlConnection
000000001AE5B870 0000000500056988 System.String    <sc.SqlCommand.set_Connection|API> %d#, %d#
000000001AE5B888 00000005003ba008 BaiRong.Provider.Data.Helper.SqlServer
000000001AE5B8A0 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B8C8 0000000300e48508 System.String    SELECT COUNT(*) AS TotalNum FROM au_Content WHERE (PublishmentSystemID = 32598 AND NodeID > 0 AND IsChecked = 'False' AND CheckedLevel IN (0,1,2,3,4))
000000001AE5B8D0 0000000300e48650 System.Data.SqlClient.SqlConnection
000000001AE5B8E0 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B8F0 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B8F8 0000000300e48650 System.Data.SqlClient.SqlConnection
000000001AE5B918 00000005003ba008 BaiRong.Provider.Data.Helper.SqlServer
000000001AE5B920 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B940 00000005003ba008 BaiRong.Provider.Data.Helper.SqlServer
000000001AE5B948 0000000300e487a0 System.Data.SqlClient.SqlCommand
000000001AE5B968 0000000300e48508 System.String    SELECT COUNT(*) AS TotalNum FROM au_Content WHERE (PublishmentSystemID = 32598 AND NodeID > 0 AND IsChecked = 'False' AND CheckedLevel IN (0,1,2,3,4))
000000001AE5B9A0 0000000300e48650 System.Data.SqlClient.SqlConnection
000000001AE5B9A8 0000000180c959d8 System.Collections.Generic.List`1[[System.Int32, mscorlib]]
000000001AE5B9B0 000000018054b410 BaiRong.Provider.Data.SqlServer.DatabaseDAO
000000001AE5B9B8 0000000300e48650 System.Data.SqlClient.SqlConnection
000000001AE5B9C0 0000000300e48508 System.String    SELECT COUNT(*) AS TotalNum FROM au_Content WHERE (PublishmentSystemID = 32598 AND NodeID > 0 AND IsChecked = 'False' AND CheckedLevel IN (0,1,2,3,4))
000000001AE5B9C8 00000005003ba008 BaiRong.Provider.Data.Helper.SqlServer
000000001AE5B9E8 0000000300e48650 System.Data.SqlClient.SqlConnection
000000001AE5B9F0 0000000300e48650 System.Data.SqlClient.SqlConnection
000000001AE5BA00 0000000300e48508 System.String    SELECT COUNT(*) AS TotalNum FROM au_Content WHERE (PublishmentSystemID = 32598 AND NodeID > 0 AND IsChecked = 'False' AND CheckedLevel IN (0,1,2,3,4))
000000001AE5BA18 0000000480038b70 System.String    SqlServer
000000001AE5BA40 0000000480038b70 System.String    SqlServer
000000001AE5BA48 0000000300e48650 System.Data.SqlClient.SqlConnection
000000001AE5BA60 00000005003ba008 BaiRong.Provider.Data.Helper.SqlServer
000000001AE5BAA0 0000000300e48650 System.Data.SqlClient.SqlConnection
000000001AE5BAB8 00000004808856e0 System.String    SELECT COUNT(*) AS TotalNum FROM {0} WHERE (PublishmentSystemID = {1} AND NodeID > 0 AND IsChecked = '{2}' AND CheckedLevel IN ({3}))
000000001AE5BAC0 0000000500b36648 System.Collections.ArrayList
000000001AE5BAC8 0000000300e48508 System.String    SELECT COUNT(*) AS TotalNum FROM au_Content WHERE (PublishmentSystemID = 32598 AND NodeID > 0 AND IsChecked = 'False' AND CheckedLevel IN (0,1,2,3,4))
000000001AE5BAF0 0000000400ad8360 SiteServer.CMS.Model.PublishmentSystemInfo
000000001AE5BAF8 0000000500b36648 System.Collections.ArrayList
000000001AE5BB28 00000004809cd9f8 System.Collections.ArrayList+ArrayListEnumeratorSimple
000000001AE5BB38 00000004809c4f40 System.Collections.ArrayList
000000001AE5BB40 00000004809c4f40 System.Collections.ArrayList
000000001AE5BB50 0000000400af50f8 SiteServer.CMS.Model.PublishmentSystemInfo
000000001AE5BB60 0000000300e481e0 System.Collections.ArrayList
000000001AE5BB80 00000001801f8070 System.Web.PipelineModuleStepContainer
000000001AE5BB90 000000020023c0b8 SiteServer.CMS.Provider.Data.SqlServer.BackgroundContentDAO
000000001AE5BBA0 0000000180c959d8 System.Collections.Generic.List`1[[System.Int32, mscorlib]]
000000001AE5BBD0 0000000180c959d8 System.Collections.Generic.List`1[[System.Int32, mscorlib]]
000000001AE5BBD8 0000000500b36648 System.Collections.ArrayList
000000001AE5BBE0 00000004809c4678 System.String    au_Content
000000001AE5BBF0 00000001801f8070 System.Web.PipelineModuleStepContainer
000000001AE5BC08 00000004809c4628 System.Collections.ArrayList
000000001AE5BC20 00000004001135f8 BaiRong.Provider.Data.SqlServer.TableCollectionDAO
000000001AE5BC60 00000004809c4e50 System.Collections.ArrayList+ArrayListEnumeratorSimple
000000001AE5BC68 00000004809c4628 System.Collections.ArrayList
000000001AE5BC70 00000004809bd920 ASP.siteserver_cms_background_right_aspx
000000001AE5BC98 000000038045ccf8 System.String    yyyy-MM-dd
000000001AE5BCE8 000000050006adb8 System.String    Text
000000001AE5BD40 00000001801f8070 System.Web.PipelineModuleStepContainer
000000001AE5BD80 00000004809bd920 ASP.siteserver_cms_background_right_aspx
000000001AE5BD88 000000040004f470 System.EventArgs
000000001AE5BDC8 00000004809bd920 ASP.siteserver_cms_background_right_aspx
000000001AE5BDD0 0000000300031420 System.String    
000000001AE5BE08 00000001801fac70 System.Web.HttpApplication+CallHandlerExecutionStep
000000001AE5BE18 00000004809bcbd0 System.Web.HttpContext
000000001AE5BE20 0000000300031420 System.String    
000000001AE5BEC8 00000001801fac70 System.Web.HttpApplication+CallHandlerExecutionStep
000000001AE5BF00 00000004809bd920 ASP.siteserver_cms_background_right_aspx
000000001AE5BF70 00000004809bd920 ASP.siteserver_cms_background_right_aspx
000000001AE5BF78 00000004808809a8 System.Web.UI.TemplateControl+EventList
000000001AE5BF88 0000000480880c10 System.Web.UI.TemplateControl+SyncEventMethodInfo
000000001AE5BFD0 000000030006cfc8 System.Globalization.CultureInfo
000000001AE5BFD8 000000030006cfc8 System.Globalization.CultureInfo
000000001AE5BFE0 000000048041ee80 System.Threading.Thread
000000001AE5C010 00000004809bd920 ASP.siteserver_cms_background_right_aspx
000000001AE5C020 00000001801f8070 System.Web.PipelineModuleStepContainer
000000001AE5C028 00000001801fac70 System.Web.HttpApplication+CallHandlerExecutionStep
000000001AE5C0B0 00000001801fab70 System.Web.HttpApplication+AsyncEventExecutionStep
000000001AE5C0D8 00000004809bcbd0 System.Web.HttpContext
000000001AE5C110 00000004809bcbd0 System.Web.HttpContext
000000001AE5C120 00000004809bc888 System.Web.Hosting.IIS7WorkerRequest
000000001AE5C128 00000001801fac70 System.Web.HttpApplication+CallHandlerExecutionStep
000000001AE5C140 00000001801fac70 System.Web.HttpApplication+CallHandlerExecutionStep
000000001AE5C1C8 00000001801fac70 System.Web.HttpApplication+CallHandlerExecutionStep
000000001AE5C1E0 00000001801f7318 ASP.global_asax
000000001AE5C1E8 00000001801fac70 System.Web.HttpApplication+CallHandlerExecutionStep
000000001AE5C278 00000004809bd7d0 System.Web.ThreadContext
000000001AE5C280 00000004809bd768 System.Web.LegacyAspNetSynchronizationContext
000000001AE5C288 00000004809bcbd0 System.Web.HttpContext
000000001AE5C308 00000001801f7318 ASP.global_asax
000000001AE5C318 00000004809c0fb8 System.Web.HttpAsyncResult
000000001AE5C320 000000030007bbf8 System.AsyncCallback
000000001AE5C330 00000001801fa900 System.Web.HttpApplication+PipelineStepManager
000000001AE5C370 000000030007b5b0 System.Web.HttpRuntime
000000001AE5C420 000000030007b5b0 System.Web.HttpRuntime
000000001AE5C428 00000004809bc888 System.Web.Hosting.IIS7WorkerRequest
000000001AE5C430 00000004809bcbd0 System.Web.HttpContext
000000001AE5C588 00000004809bcbd0 System.Web.HttpContext
000000001AE5C590 00000004809bc888 System.Web.Hosting.IIS7WorkerRequest
000000001AE5C5C8 00000004809bd120 System.Web.RootedObjects
000000001AE5CA98 00000001801f76c0 System.Web.Security.WindowsAuthenticationModule
000000001AE5CAD8 00000001801f8030 System.Web.PipelineModuleStepContainer
000000001AE5CB60 00000001801f7318 ASP.global_asax
000000001AE5CBC0 000000048041ee80 System.Threading.Thread
000000001AE5CBD0 000000048041ee80 System.Threading.Thread
000000001AE5CDD8 00000001801f7318 ASP.global_asax
000000001AE5CDE8 000000030007b5b0 System.Web.HttpRuntime
000000001AE5CF10 000000030007b5b0 System.Web.HttpRuntime
000000001AE5CFF8 00000004809bd7d0 System.Web.ThreadContext
000000001AE5D078 00000004809bcbd0 System.Web.HttpContext
000000001AE5D080 00000004809bc888 System.Web.Hosting.IIS7WorkerRequest
000000001AE5D0B8 00000004809bd120 System.Web.RootedObjects
000000001AE5D6A0 00000003000dc740 System.String    MapPath_
000000001AE5DAA0 00000002800fb9c8 System.String    ASP.NET_SessionId
000000001AE5DAD0 00000002800fcc08 System.String    timeout
000000001AE5DAF0 00000002800fd0b0 System.Collections.Specialized.NameObjectCollectionBase+NameObjectEntry
000000001AE5DBC0 000000020035d760 System.Web.PipelineModuleStepContainer
000000001AE5DC10 000000020035cdf0 System.Web.SessionState.SessionStateModule
000000001AE5DCD0 000000020035ea78 System.EventHandler
000000001AE5DF80 000000030007bbf8 System.AsyncCallback
000000001AE5E080 000000030007b5b0 System.Web.HttpRuntime
000000001AE5E640 000000020035d780 System.Web.PipelineModuleStepContainer
000000001AE5E6F8 000000020035cef0 System.Web.Security.WindowsAuthenticationModule
000000001AE5E738 000000020035d860 System.Web.PipelineModuleStepContainer
000000001AE5E7C0 000000020035cb48 ASP.global_asax
000000001AE5E820 000000048041ee80 System.Threading.Thread
000000001AE5E830 000000048041ee80 System.Threading.Thread
000000001AE5EA38 000000020035cb48 ASP.global_asax
000000001AE5EA48 000000030007b5b0 System.Web.HttpRuntime
000000001AE5EB70 000000030007b5b0 System.Web.HttpRuntime
000000001AE5EC58 00000004809b90f8 System.Web.ThreadContext
000000001AE5ECD8 00000004809b84f8 System.Web.HttpContext
000000001AE5ECE0 00000004809b81b0 System.Web.Hosting.IIS7WorkerRequest
000000001AE5ED18 00000004809b8a48 System.Web.RootedObjects
```

这里只有一个`SQL`

```
SELECT COUNT(*) AS TotalNum FROM au_Content WHERE (PublishmentSystemID = 32598 AND NodeID > 0 AND IsChecked = 'False' AND CheckedLevel IN (0,1,2,3,4))
```

跟同事核对了，他说`au_Content`这张表很大，那么初步锁定目标：该表被频繁访问，可能在向`SQL Server`请求数据时被阻塞住了，资源无法及时释放出来。

如果说`SELECT COUNT(*)`就造成阻塞，那么正常的`SELECT TOP 20 *`获取分页信息，可能也受影响的。 

最后建议同事，在下次事发的时候，先在`SQL Server`监控数据库的状况，看有没有死锁/阻塞。如果数据库都正常，那么需要再`dump`几个继续分析。

* * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * *

同事在事发地时候，对`SQL Server`进行实时监控，确实发现这些`SQL`有阻塞，而且`wait_type=ASYNC_NETWORK_IO`，这个提示可能是网络故障，或者程序接收数据没完成。

咨询同事数据库是否跟Web部署在同一台机，结果是都部署在同一台机上面，排除了网络问题。坚持Web请求数据库是否没有关闭`SqlConnection`。

还是先看栈是否有等待某个对象

```
0:060> kb
 # RetAddr           : Args to Child                                                           : Call Site
00 000007fe`fd8310dc : 00000000`13774ec0 00000000`138e7050 00000000`00001f40 00000000`1ae5aec0 : ntdll!ZwWaitForSingleObject+0xa
01 000007fe`de09869f : 00000000`000003e5 00000000`1ae5b0b8 00000000`00000000 00000000`000012cd : KERNELBASE!WaitForSingleObjectEx+0x79
02 000007fe`de08bf56 : 00000000`10e68da0 00000000`1ae5b0b8 00000000`1ae5afd0 000007fe`00000000 : System_Data!Np::ReadSync+0x17f
03 000007fe`d6c83ab1 : ffffffff`ffffffff 00000000`137dfe90 00000000`00007530 00000000`1ae5b0c8 : System_Data!SNIReadSync+0x6e
04 000007fe`d6c6a35f : 00000000`00000000 00000000`1ae5b0c8 00000000`1ae5b160 00000004`80042af0 : System_Data_ni!load_config_used+0x41911
05 000007fe`d6c6a07e : 00000004`80042ef0 00000000`1ae5b160 00000000`1ae5b160 00000000`00000000 : System_Data_ni!load_config_used+0x281bf
06 000007fe`d6c69f90 : 00000004`80042b90 00000004`80043188 00000000`1ae5b160 00000000`00000000 : System_Data_ni!load_config_used+0x27ede
07 000007fe`d6c6a760 : 00000000`137dfe90 000007fe`00000000 00000000`021aa1e0 00000000`10eec940 : System_Data_ni!load_config_used+0x27df0
08 000007fe`d6c6d6a2 : 00000000`1ae5b270 000007fe`f9e85a1f 01d19622`13b1d329 00000000`00000002 : System_Data_ni!load_config_used+0x285c0
09 000007fe`d6c6c6d6 : 00000000`1ae5b280 000007fe`d6c692f9 00000004`80042ef0 00000004`80043188 : System_Data_ni!load_config_used+0x2b502
0a 000007fe`d6c792ed : 00000004`80042af0 00000004`00000002 00000003`00e487a0 00000003`00e48918 : System_Data_ni!load_config_used+0x2a536
0b 000007fe`d6c79276 : 00000000`1ae5b701 000007fe`d6c61501 00000004`80042b90 00000000`00000002 : System_Data_ni!load_config_used+0x3714d
0c 000007fe`d6c739ec : 00000003`00e48918 00000000`1ae5b201 00000000`1ae5b490 00000000`00000000 : System_Data_ni!load_config_used+0x370d6
0d 000007fe`d6c738b7 : 00000003`00e487a0 00000003`00e48918 00000003`00e488e0 00000000`00000000 : System_Data_ni!load_config_used+0x3184c
0e 000007fe`d6c72d2a : 00000003`00e487a0 00000005`00056be8 00000003`00e48700 00000001`01000000 : System_Data_ni!load_config_used+0x31717
0f 000007fe`d6c72ac8 : 00000003`00e487a0 000007fe`dcf513cb 00000000`00000000 000007fe`f9f3d3c5 : System_Data_ni!load_config_used+0x30b8a
10 000007fe`d6c71930 : 000007fe`9aad17d8 00000000`00000001 00000004`800406a0 00000003`00e48760 : System_Data_ni!load_config_used+0x30928
11 000007fe`d6c7157b : 00000003`00e487a0 000007fe`d6c746db 00000005`000563e8 00000003`00e48650 : System_Data_ni!load_config_used+0x2f790
12 000007fe`d70c4f11 : 00000000`1ae5b890 000007fe`d70c4dbe 00000000`00000000 00000003`00e48508 : System_Data_ni!load_config_used+0x2f3db
13 000007fe`9a926e4f : 00000003`00e487a0 000007fe`9a926c76 00000003`00e487a0 00000003`00e48650 : System_Data_ni!_bidW103 <PERF> (System_Data_ni+0x5d4f11)
14 000007fe`9a926aa0 : 00000005`003ba008 00000003`00e487a0 00000000`00000000 000007fe`d6c612e9 : 0x000007fe`9a926e4f
15 000007fe`9a940026 : 00000000`00000000 00000003`00e48650 00000003`00e48650 000007fe`d6c607eb : 0x000007fe`9a926aa0
16 000007fe`9a93ffd8 : 00000000`00000000 ffffffff`ffffffff 00000004`80038b70 00000003`00e48650 : 0x000007fe`9a940026
17 000007fe`9a93fcda : 00000000`00000000 00000000`00000000 00000000`00000000 00000945`0a99c8ef : 0x000007fe`9a93ffd8
18 000007fe`9b233e5b : 000007fe`dcab506a 00000000`00000004 00000004`00ad8360 00000005`00b36648 : 0x000007fe`9a93fcda
19 000007fe`9b233360 : 00000004`81434110 00000004`81430701 00000001`80c959d8 00000005`00b36648 : 0x000007fe`9b233e5b
1a 000007fe`9b23305e : 00000004`8142be78 00000000`00000017 00000000`00000000 00000000`00000000 : 0x000007fe`9b233360
1b 000007fe`9b23287d : 00000000`00000000 00000003`8045ccf8 00000004`8142be20 000007fe`dbb7ecdf : 0x000007fe`9b23305e
1c 000007fe`9b231a87 : 00000004`809bd920 00000004`0004f470 000007fe`9b231860 000007fe`da2224e4 : 0x000007fe`9b23287d
1d 000007fe`da19d507 : 00000004`8141df90 000007fe`da1acd5b 00000004`81417078 00000001`801fac70 : 0x000007fe`9b231a87
1e 000007fe`da1be1ea : 00000000`00000015 000007fe`dcfb4c39 00000000`00000000 00000000`00000000 : System_Web_ni+0x2cd507
1f 000007fe`da1bd469 : 00000004`809bd920 00000000`00000001 00000000`1ae5be01 00000004`81416eb8 : System_Web_ni+0x2ee1ea
20 000007fe`da1bd2c7 : 00000004`809bd920 00000004`808809a8 00000004`80880901 00000004`80880c10 : System_Web_ni+0x2ed469
21 000007fe`da1bb9f3 : 00000004`809bd920 00000000`00000000 00000001`801f8070 00000001`801fac70 : System_Web_ni+0x2ed2c7
22 000007fe`da1c62a1 : 00000000`00000000 00000004`81416b08 00000000`1ae5c040 000007fe`da1acd5b : System_Web_ni+0x2eb9f3
23 000007fe`da18d495 : 00000001`801fac70 00000000`1ae5c208 00000000`00000000 00000000`00000000 : System_Web_ni+0x2f62a1
24 000007fe`da1aabda : 00000001`801f7318 00000001`801fac70 00000000`1ae5c2d2 00000000`00000000 : System_Web_ni+0x2bd495
25 000007fe`da18d6a3 : 00000001`801fa900 000007fe`da1a342d 00000000`00000000 00000000`00000000 : System_Web_ni+0x2dabda
26 000007fe`da1875de : 000007fe`d9f37000 00000000`1ae5c340 00000000`00000000 00000000`1ae5c360 : System_Web_ni+0x2bd6a3
27 000007fe`da190561 : 00000003`0007b5b0 00000004`809bc888 00000004`809bcbd0 00000000`00000000 : System_Web_ni+0x2b75de
28 000007fe`da18ff92 : 00000000`00000002 00000000`13a30b60 00000000`13a30f88 00000000`13a30e70 : System_Web_ni+0x2c0561
29 000007fe`da8e5dc1 : 00000000`00000001 000007fe`f9ea040b 00000000`00000000 00000000`00000000 : System_Web_ni+0x2bff92
2a 000007fe`f9eeb42e : 00000000`13a3101c 000007fe`fab12015 00000000`00000001 00000000`1ae5c7d0 : System_Web_ni+0xa15dc1
2b 000007fe`fab11d6e : 00000000`13a30b60 00000000`00eee418 00000000`00000080 00000000`00000000 : clr!UMThunkStub+0x6e
2c 000007fe`fab11cfb : 00000000`00000000 00000000`13a30b60 00000000`00000080 00000000`00000000 : webengine4!W3_MGD_HANDLER::ProcessNotification+0x78
2d 000007fe`fab11bfe : 00000000`13a30b60 00000000`00eee418 00000000`00000080 00000000`13a2fec0 : webengine4!W3_MGD_HANDLER::DoWork+0x34f
2e 000007fe`fab127a9 : 00000000`00000001 00000000`00eee418 00000000`00c29f28 00000000`13a2fec0 : webengine4!RequestDoWork+0x34e
2f 000007fe`f5502de7 : 000007fe`d9f41148 00000000`00c29fa8 00000000`00000000 00000000`00000000 : webengine4!CMgdEngHttpModule::OnExecuteRequestHandler+0x21
30 000007fe`f55046a4 : 00000000`13a2fec8 000007fe`9aaa09a0 00000002`00000069 00000945`45c167b8 : iiscore!NOTIFICATION_CONTEXT::RequestDoWork+0x233
31 000007fe`f550a775 : 00000000`1ae5eac8 00000000`13a2fec0 00000000`13a2fec8 00000000`00000000 : iiscore!NOTIFICATION_CONTEXT::CallModulesInternal+0x174
32 000007fe`f5505a03 : 00000000`00000000 00000001`801f76c0 00000000`00000000 00000000`00000000 : iiscore!NOTIFICATION_CONTEXT::CallModules+0x25
33 000007fe`f550a81c : 00000004`81416b08 00000004`81416eb8 00000000`1ae5ce20 000007fe`da18ff36 : iiscore!W3_CONTEXT::DoWork+0x34d
34 000007fe`fab12734 : 00000000`1ae5ce60 00000000`00000004 00000000`13a30b60 00000000`00000000 : iiscore!W3_CONTEXT::IndicateCompletion+0x8c
35 000007fe`da2395eb : 000007fe`d9f36610 000007fe`d9ed1550 00000945`45c167b8 000007fe`fa550008 : webengine4!MgdIndicateCompletion+0x60
36 000007fe`da19074f : 00000003`0007b5b0 00000004`81416b08 00000004`81416eb8 00000000`1d9b1db0 : System_Web_ni+0x3695eb
37 000007fe`da18ff92 : 00000000`00000004 00000000`13a30b60 00000000`00000004 00000000`00000000 : System_Web_ni+0x2c074f
38 000007fe`da8e5dc1 : 00000000`00c2a080 00000000`13a2fec0 00000000`00000000 00000000`1ae5d2e0 : System_Web_ni+0x2bff92
39 000007fe`f9eeb42e : 00000000`13a3101c 000007fe`fab12015 00000000`00000000 00000000`1ae5d2c0 : System_Web_ni+0xa15dc1
3a 000007fe`fab11d6e : 00000000`13a30b60 00000000`00eeddb8 00000000`00000002 00000000`00000000 : clr!UMThunkStub+0x6e
3b 000007fe`fab11cfb : 000007fe`00000004 00000000`13a30b60 00000000`00000000 000007fe`fab13a73 : webengine4!W3_MGD_HANDLER::ProcessNotification+0x78
3c 000007fe`fab11bfe : 00000000`13a30b60 00000000`00000000 00000000`00000002 00000000`13a2fec0 : webengine4!W3_MGD_HANDLER::DoWork+0x34f
3d 000007fe`fab126bf : 00000000`00c2a0c8 00000000`00eeddb8 00000000`00c4e6d0 00000000`13a2fec0 : webengine4!RequestDoWork+0x34e
3e 000007fe`f5506540 : 00000000`00c4d100 00000000`00000000 00000000`00c2a108 00000000`00000000 : webengine4!CMgdEngHttpModule::OnAuthenticateRequest+0x1f
3f 000007fe`f55046a4 : 00000000`13a2fec8 203a6567`416b6300 000007fe`f5535ba8 00000000`00000000 : iiscore!NOTIFICATION_CONTEXT::RequestDoWork+0xd8
40 000007fe`f550a775 : 00000000`f5535ba8 00000000`13a2fec0 00000000`13a2fec8 00000000`00000000 : iiscore!NOTIFICATION_CONTEXT::CallModulesInternal+0x174
41 000007fe`f5506c80 : 00000000`00000000 00000000`13a2fec0 00000000`00000000 00000000`13a2fec8 : iiscore!NOTIFICATION_CONTEXT::CallModules+0x25
42 000007fe`f553262d : 000007fe`f5535ba8 00000000`13a2fec8 00000000`00000000 00000000`13a2fec0 : iiscore!W3_CONTEXT::DoWork+0x2cd
43 000007fe`f5532a4f : 00000000`00000000 00000000`00000012 00000000`13a2ff70 000007fe`f552288a : iiscore!W3_MAIN_CONTEXT::StartNotificationLoop+0x5d
44 000007fe`f52266ab : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`1ae5dc10 : iiscore!W3_CONTEXT::ExecuteRequest+0x2bf
45 000007fe`f52211f9 : 00000000`00000000 00000000`1ae5e020 00000000`00000001 00000000`00000000 : isapi!ISAPI_REQUEST::ExecuteUrl+0x40f
46 000007fe`f521db5c : 00000000`00000000 00000000`00000001 00000000`00000001 00000000`00000000 : isapi!SSFExecuteUrl+0x85d
47 000007fe`f502e7c0 : 00000000`00000000 00000000`13f162d8 00000000`1ae5e4a0 00000000`00000000 : isapi!ServerSupportFunction+0x964
48 000007fe`f521c540 : 00000000`13f16290 00000000`13f162d8 000007fe`f5213588 00000000`10a1d79c : SafedogIISAuditor64!HttpExtensionProc+0x2b0
49 000007fe`f5224007 : 00000000`13f162d8 00000000`00000000 00000000`13f146b0 00000000`00000001 : isapi!ProcessIsapiRequest+0x13c
4a 000007fe`f521b421 : 00000000`00c2a5a0 00000000`00000000 00000000`13f144d0 00000000`00000000 : isapi!W3_ISAPI_HANDLER::DoWork+0x4fb
4b 000007fe`f521b964 : 00000000`00000001 00000000`00c2a188 00000000`00c2a188 00000000`00000000 : isapi!RequestDoWork+0x40d
4c 000007fe`f5502de7 : 000007fe`d9f41148 00000000`00c29fa8 00000000`00000000 00000000`00000000 : isapi!CIISHttpModule::OnExecuteRequestHandler+0x1c
4d 000007fe`f55046a4 : 00000000`13f144d8 000007fe`9aaa09a0 00000002`00000069 00000945`45c167b8 : iiscore!NOTIFICATION_CONTEXT::RequestDoWork+0x233
4e 000007fe`f550a775 : 00000000`1ae5efc8 00000000`13f144d0 00000000`13f144d8 00000000`00000000 : iiscore!NOTIFICATION_CONTEXT::CallModulesInternal+0x174
4f 000007fe`f5505a03 : 00000000`00000000 00000002`0035cef0 00000000`00000000 00000000`00000000 : iiscore!NOTIFICATION_CONTEXT::CallModules+0x25
50 000007fe`f550a81c : 00000004`81410230 00000004`814105e0 00000000`1ae5ea80 000007fe`da18ff36 : iiscore!W3_CONTEXT::DoWork+0x34d
51 000007fe`fab12734 : 00000000`1ae5eac0 00000000`00000004 00000000`13f155e0 00000000`00000000 : iiscore!W3_CONTEXT::IndicateCompletion+0x8c
52 000007fe`da2395eb : 000007fe`d9f36610 000007fe`d9ed1550 00000945`45c167b8 000007fe`fa550008 : webengine4!MgdIndicateCompletion+0x60
53 000007fe`da19074f : 00000003`0007b5b0 00000004`81410230 00000004`814105e0 000007fe`9a732020 : System_Web_ni+0x3695eb
54 000007fe`da18ff92 : 00000000`1ae5ed08 00000000`13f155e0 00000000`00000000 00000000`00000000 : System_Web_ni+0x2c074f
55 000007fe`da8e5dc1 : 00000000`00000000 000007fe`f9eae3ea 00000000`00000000 000007fe`f9eae3ea : System_Web_ni+0x2bff92
56 000007fe`f9eeb683 : ffffffff`ffffffff 000007fe`00000001 00000000`137dfe90 000007fe`f9eae6d4 : System_Web_ni+0xa15dc1
57 000007fe`f9f503ba : ffffffff`fffffffe 00000000`137dfe90 ffffffff`fffffffe 000007fe`f9f3d2ec : clr!UM2MThunk_WrapperHelper+0x43
58 000007fe`f9f490ef : 00000000`1ae5efc8 00000000`00000001 00000000`137dfe90 00000000`137dfe90 : clr!UM2MThunk_Wrapper+0x5a
59 000007fe`f9f50335 : 00000000`1ae5f101 000007fe`00000002 00000000`00000002 000007fe`fab12234 : clr!Thread::DoADCallBack+0x139
5a 000007fe`f9eeb62d : 00000000`02297f20 ffffffff`ffffffff 00000000`13f155e0 00000000`1ae5f1e0 : clr!UM2MDoADCallBack+0x91
5b 000007fe`fab11d6e : 00000000`00000000 00000000`13f155e0 00000000`00000003 00000000`00000004 : clr!UMThunkStub+0x26d
5c 000007fe`fab12276 : 00000000`00000004 00000000`13f155e0 00000945`00000000 00000000`00000618 : webengine4!W3_MGD_HANDLER::ProcessNotification+0x78
5d 000007fe`f9f48b42 : 00000000`13f155e0 000007fe`fa77c990 000007fe`fab12234 00000000`1ae5f378 : webengine4!ProcessNotificationCallback+0x42
5e 000007fe`f9f0f046 : 00000000`1ae5f380 00000000`1ae5f300 00000000`55cdf836 00000000`00000001 : clr!UnManagedPerAppDomainTPCount::DispatchWorkItem+0x122
5f 000007fe`f9f0ef3a : 00000000`137dfe90 00000000`137dfe90 00000000`00000000 00000000`ffff0000 : clr!ThreadpoolMgr::ExecuteWorkRequest+0x46
60 000007fe`f9fcfcb6 : 00000000`00000000 00000000`0000f101 00000000`00000001 00000000`00000804 : clr!ThreadpoolMgr::WorkerThreadStart+0xf4
61 00000000`77935a4d : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : clr!Thread::intermediateThreadProc+0x7d
62 00000000`77a6b831 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : kernel32!BaseThreadInitThunk+0xd
63 00000000`00000000 : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!RtlUserThreadStart+0x1d
```

看第一行，不出所料，是在等待`ntdll!ZwWaitForSingleObject+0xa`。

检查是否有`SqlConnnection`堆积在`FinalizeQueue`中，通常不应该大于`10`，当数量超过`30`时，说明代码中有使用完`SqlConnection`后忘记调用`Close`或`Dispose`。

```
0:060> !finalizequeue
SyncBlocks to be cleaned up: 0
Free-Threaded Interfaces to be released: 0
MTA Interfaces to be released: 0
STA Interfaces to be released: 0
----------------------------------
------------------------------
Heap 0
generation 0 has 424 finalizable objects (000000001a6befa0->000000001a6bfce0)
generation 1 has 20 finalizable objects (000000001a6bef00->000000001a6befa0)
generation 2 has 210 finalizable objects (000000001a6be870->000000001a6bef00)
Ready for finalization 0 objects (000000001a6bfce0->000000001a6bfce0)
------------------------------
Heap 1
generation 0 has 7 finalizable objects (000000001b3264b8->000000001b3264f0)
generation 1 has 26 finalizable objects (000000001b3263e8->000000001b3264b8)
generation 2 has 163 finalizable objects (000000001b325ed0->000000001b3263e8)
Ready for finalization 0 objects (000000001b3264f0->000000001b3264f0)
------------------------------
Heap 2
generation 0 has 208 finalizable objects (000000001a359358->000000001a3599d8)
generation 1 has 26 finalizable objects (000000001a359288->000000001a359358)
generation 2 has 163 finalizable objects (000000001a358d70->000000001a359288)
Ready for finalization 0 objects (000000001a3599d8->000000001a3599d8)
------------------------------
Heap 3
generation 0 has 551 finalizable objects (0000000013dbab60->0000000013dbbc98)
generation 1 has 25 finalizable objects (0000000013dbaa98->0000000013dbab60)
generation 2 has 271 finalizable objects (0000000013dba220->0000000013dbaa98)
Ready for finalization 0 objects (0000000013dbbc98->0000000013dbbc98)
------------------------------
Heap 4
generation 0 has 112 finalizable objects (000000001b460370->000000001b4606f0)
generation 1 has 11 finalizable objects (000000001b460318->000000001b460370)
generation 2 has 203 finalizable objects (000000001b45fcc0->000000001b460318)
Ready for finalization 0 objects (000000001b4606f0->000000001b4606f0)
------------------------------
Heap 5
generation 0 has 325 finalizable objects (000000001a1dbdb0->000000001a1dc7d8)
generation 1 has 21 finalizable objects (000000001a1dbd08->000000001a1dbdb0)
generation 2 has 195 finalizable objects (000000001a1db6f0->000000001a1dbd08)
Ready for finalization 0 objects (000000001a1dc7d8->000000001a1dc7d8)
------------------------------
Heap 6
generation 0 has 517 finalizable objects (0000000019acfb90->0000000019ad0bb8)
generation 1 has 37 finalizable objects (0000000019acfa68->0000000019acfb90)
generation 2 has 193 finalizable objects (0000000019acf460->0000000019acfa68)
Ready for finalization 0 objects (0000000019ad0bb8->0000000019ad0bb8)
------------------------------
Heap 7
generation 0 has 1320 finalizable objects (000000001ba7f770->000000001ba820b0)
generation 1 has 50 finalizable objects (000000001ba7f5e0->000000001ba7f770)
generation 2 has 228 finalizable objects (000000001ba7eec0->000000001ba7f5e0)
Ready for finalization 0 objects (000000001ba820b0->000000001ba820b0)
Statistics for all finalizable objects (including all objects ready for finalization):
              MT    Count    TotalSize Class Name
000007fedd1809a8        1           32 System.Threading.Gen2GcCallback
000007fedb2b9a48        1           32 Microsoft.Win32.SafeHandles.SafeCspHandle
000007fedd1b2d10        1           40 System.LocalDataStoreSlot
000007fedb2b9b38        1           40 Microsoft.Win32.SafeHandles.SafeCapiKeyHandle
000007fedd12a948        1           48 System.Runtime.CompilerServices.ConditionalWeakTable`2[[System.Threading.Tasks.TaskScheduler, mscorlib],[System.Object, mscorlib]]
000007fedbd156e0        1           48 Microsoft.CSharp.CSharpCodeProvider
000007feda314178        2           48 System.Web.Configuration.ImpersonateTokenRef
000007fedd194928        2           64 Microsoft.Win32.SafeHandles.SafeFileHandle
000007fedbcfb960        2           64 Microsoft.Win32.SafeHandles.SafeFileMapViewHandle
000007fedbcfb190        2           64 Microsoft.Win32.SafeHandles.SafeFileMappingHandle
000007feda3116c8        2           64 System.Web.PerfInstanceDataHandle
000007fed6cd3078        2           64 Bid+AutoInit
000007fedd175c58        3           72 System.Reflection.Emit.DynamicResolver+DestroyScout
000007fedd130e80        2           80 System.Threading.RegisteredWaitHandleSafe
000007fedd13fd60        2           96 System.Runtime.CompilerServices.ConditionalWeakTable`2[[System.Object, mscorlib],[System.Runtime.Serialization.SerializationInfo, mscorlib]]
000007fedd137970        3           96 Microsoft.Win32.SafeHandles.SafeFileMappingHandle
000007fedd1378e0        3           96 Microsoft.Win32.SafeHandles.SafeViewOfFileHandle
000007fedd133840        3           96 System.Threading.TimerQueue+AppDomainTimerSafeHandle
000007fed56812c8        2           96 System.Runtime.CompilerServices.ConditionalWeakTable`2[[System.Transactions.ContextKey, System.Transactions],[System.Transactions.ContextData, System.Transactions]]
000007feda3144f0        2          112 System.Web.Compilation.CompilationMutex
000007fed6cdc8a0        2          112 System.Data.SqlClient.SNILoadHandle
000007fedd1b2df0        6          144 System.LocalDataStoreHolder
000007fedd1b15a0        5          160 Microsoft.Win32.SafeHandles.SafeTokenHandle
000007fedd180930        1          160 System.Threading.PinnableBufferCacheEventSource
000007fedd125c08        1          160 System.Threading.Tasks.TplEtwProvider
000007fef1f59ef8        2          176 System.Runtime.Diagnostics.EtwProvider
000007fed6cd9968        2          176 System.Data.ProviderBase.DbConnectionPool+PoolWaitHandles
000007fed568a2d0        6          192 System.Transactions.SafeIUnknown
000007fed6cdf7c0        7          224 System.Data.SqlClient.SNIPacket
000007fed6cd9f10        7          280 System.Data.SqlClient.SNIHandle
000007feda386aa8        9          288 System.Web.SessionState.OutOfProcSessionStateStore+OutOfProcConnection
000007fedd17eca8        3          312 System.Runtime.Remoting.Contexts.Context
000007fedd1250b8        2          320 System.Collections.Concurrent.CDSCollectionETWBCLProvider
000007fedbcf1698        2          320 System.IO.FileSystemWatcher
000007feda323cf0        2          336 System.Web.AspNetEventSource
000007fedc7fc068        6          384 System.Drawing.SolidBrush
000007fedd198038        7          448 System.Threading.ReaderWriterLock
000007fedd1a7600        3          480 System.Diagnostics.Tracing.FrameworkEventSource
000007fedd13e850       24          576 System.SizedReference
000007fedd1a9190       21          672 Microsoft.Win32.SafeHandles.SafeRegistryHandle
000007fedd1a7a28        9          792 System.Diagnostics.Tracing.EventSource+OverideEventProvider
000007fedd19a5e0       41         1312 Microsoft.Win32.SafeHandles.SafeWaitHandle
000007feda35c968       36         1728 System.Web.Security.FileSecurityDescriptorWrapper
000007fedd136938       49         1960 System.Threading.ThreadPoolWorkQueueThreadLocals
000007fedd1922f8       24         2304 System.Threading.Thread
000007fedd133768      100         2400 System.Threading.TimerHolder
000007fedbcf0e58       20         3520 System.Diagnostics.PerformanceCounter
000007fedd13e360       50         3600 System.Reflection.Emit.DynamicResolver
000007fedd199230      192         4608 System.WeakReference
000007feda356bf8      156         6240 System.Web.HttpResponseUnmanagedBufferElement
000007feda30d6f8      308        12320 System.Web.ApplicationImpersonationContext
000007fedd1749c0      740        23680 System.Security.Cryptography.SafeKeyHandle
000007fedd1360b8      744        23808 System.Security.Cryptography.SafeProvHandle
000007feda311cd8     1193        85896 System.Web.DirMonCompletion
000007fed6cd6998      744       154752 System.Data.SqlClient.SqlConnection
000007fed6cd7ca8      744       172608 System.Data.SqlClient.SqlCommand
Total 5306 objects
```

果然，`System.Data.SqlClient.SqlConnection`的数量达到`744`个，可以用`!dumpheap`便利所有的`SqlConnection`出来

```
0:060> !DumpHeap /d -mt 000007fed6cd6998
         Address               MT     Size
0000000180d371f0 000007fed6cd6998      208   
```

然后再用`!do <Address>`就可以看到链接字符串了，这里涉及隐私就不输出了。

由于`SqlConnection`和`SqlCommand`的数量一样，推断代码中是一起出现的。

检查`SqlCommand`，同样用`!dumpheap`和`!do`，看到了执行的`_commandText`。

```
0:060> !DumpObj /d 0000000500e1e068
Name:        System.Data.SqlClient.SqlCommand
MethodTable: 000007fed6cd7ca8
EEClass:     000007fed6b180c8
Size:        232(0xe8) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\System.Data\v4.0_4.0.0.0__b77a5c561934e089\System.Data.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
000007fedd1913e8  400019b        8        System.Object  0 instance 0000000000000000 __identity
000007fedbd04680  40002d2       10 ...ponentModel.ISite  0 instance 0000000000000000 site
000007fedbd12e28  40002d3       18 ....EventHandlerList  0 instance 0000000000000000 events
000007fedd1913e8  40002d1      1b8        System.Object  0   shared           static EventDisposed
                                 >> Domain:Value  00000000021cd0e0:NotInit  0000000002297f20:NotInit  0000000013aff0a0:NotInit  <<
000007fedd193980  4001055       b0         System.Int32  1 instance            98928 ObjectID
000007fedd190e08  4001056       20        System.String  0 instance 0000000500e1ddc8 _commandText
000007fed6cd73b8  4001057       b4         System.Int32  1 instance                1 _commandType
000007fedd193980  4001058       b8         System.Int32  1 instance               30 _commandTimeout
000007fed6cf82e0  4001059       bc         System.Int32  1 instance                3 _updatedRowSource
000007fedd198c48  400105a       d8       System.Boolean  1 instance                0 _designTimeInvisible
000007fed6ce5578  400105b       28 ...ent.SqlDependency  0 instance 0000000000000000 _sqlDep
000007fedd198c48  400105c       d9       System.Boolean  1 instance                0 _inPrepare
000007fedd193980  400105d       c0         System.Int32  1 instance               -1 _prepareHandle
000007fedd198c48  400105e       da       System.Boolean  1 instance                0 _hiddenPrepare
000007fedd193980  400105f       c4         System.Int32  1 instance               -1 _preparedConnectionCloseCount
000007fedd193980  4001060       c8         System.Int32  1 instance               -1 _preparedConnectionReconnectCount
000007fed6cd66f0  4001061       30 ...rameterCollection  0 instance 0000000500e1e678 _parameters
000007fed6cd6998  4001062       38 ...ent.SqlConnection  0 instance 0000000500e1df18 _activeConnection
000007fedd198c48  4001063       db       System.Boolean  1 instance                0 _dirty
000007fed6ce2130  4001064       cc         System.Int32  1 instance                0 _execType
000007fedd124918  4001065       40      System.Object[]  0 instance 0000000000000000 _rpcArrayOf1
000007fed6cdcd78  4001066       48 ...t._SqlMetaDataSet  0 instance 0000000500e1e488 _cachedMetaData
000007fedd186a20  4001067       50 ...bject, mscorlib]]  0 instance 0000000000000000 _reconnectionCompletionSource
000007fed6cded00  4001068       58 ...+CachedAsyncState  0 instance 0000000500e1e150 _cachedAsyncState
000007fedd193980  4001069       d0         System.Int32  1 instance               -1 _rowsAffected
000007fed6ce5ee8  400106a       60 ...tificationRequest  0 instance 0000000000000000 _notification
000007fedd198c48  400106b       dc       System.Boolean  1 instance                1 _notificationAutoEnlist
000007fed6ce19a8  400106c       68 ...nt.SqlTransaction  0 instance 0000000000000000 _transaction
000007fed6ce8568  400106d       70 ...letedEventHandler  0 instance 0000000000000000 _statementCompletedEventHandler
000007fed6cde778  400106e       78 ...ParserStateObject  0 instance 0000000000000000 _stateObj
000007fedd198c48  400106f       dd       System.Boolean  1 instance                0 _pendingCancel
000007fedd198c48  4001070       de       System.Boolean  1 instance                0 _batchRPCMode
000007fed6b10fd0  4001071       80 ...PC, System.Data]]  0 instance 0000000000000000 _RPCList
000007fedd124918  4001072       88      System.Object[]  0 instance 0000000000000000 _SqlRPCBatchArray
000007fed6b10b40  4001073       90 ...on, System.Data]]  0 instance 0000000000000000 _parameterCollectionList
000007fedd193980  4001074       d4         System.Int32  1 instance                0 _currentlyExecutingBatch
000007fed6ceb260  4001075       98 ...Server.SmiContext  0 instance 0000000000000000 _smiRequestContext
000007fed6cee2d0  4001076       a0 ...+CommandEventSink  0 instance 0000000000000000 _smiEventSink
000007fed6cf2a00  4001077       a8 ...DeferedProcessing  0 instance 0000000000000000 _outParamEventSink
000007fedd193980  4001054      a18         System.Int32  1   shared           static _objectTypeCount
                                 >> Domain:Value  00000000021cd0e0:NotInit  0000000002297f20:98933 0000000013aff0a0:8 <<
000007fedd124918  4001078      ea0      System.Object[]  0   shared           static PreKatmaiProcParamsNames
                                 >> Domain:Value  00000000021cd0e0:NotInit  0000000002297f20:0000000500056858 0000000013aff0a0:000000048045a128 <<
000007fedd124918  4001079      ea8      System.Object[]  0   shared           static KatmaiProcParamsNames
                                 >> Domain:Value  00000000021cd0e0:NotInit  0000000002297f20:00000005000568f0 0000000013aff0a0:000000048045a1c0 <<
000007fed726d410  400107a      eb0 ...er, System.Data]]  0   shared           static CS$<>9__CachedAnonymousMethodDelegate25
                                 >> Domain:Value  00000000021cd0e0:NotInit  0000000002297f20:0000000000000000 0000000013aff0a0:0000000000000000 <<
000007fedd1a1590  400107b      eb8        System.Action  0   shared           static CS$<>9__CachedAnonymousMethodDelegate39
                                 >> Domain:Value  00000000021cd0e0:NotInit  0000000002297f20:0000000000000000 0000000013aff0a0:0000000000000000 <<
000007fedd1a1590  400107c      ec0        System.Action  0   shared           static CS$<>9__CachedAnonymousMethodDelegate46
                                 >> Domain:Value  00000000021cd0e0:NotInit  0000000002297f20:0000000000000000 0000000013aff0a0:0000000000000000 <<
0:060> !DumpObj /d 0000000500e1ddc8
Name:        System.String
MethodTable: 000007fedd190e08
EEClass:     000007fedcab4860
Size:        336(0x150) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\mscorlib\v4.0_4.0.0.0__b77a5c561934e089\mscorlib.dll
String:      SELECT COUNT(*) AS TotalNum FROM cms_ContentVote WHERE (PublishmentSystemID = 48290 AND NodeID > 0 AND IsChecked = 'False' AND CheckedLevel IN (0,1,2,3,4))
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
000007fedd193980  40000ab        8         System.Int32  1 instance              155 m_stringLength
000007fedd1920f0  40000ac        c          System.Char  1 instance               53 m_firstChar
000007fedd190e08  40000ad       18        System.String  0   shared           static Empty
                                 >> Domain:Value  00000000021cd0e0:NotInit  0000000002297f20:NotInit  0000000013aff0a0:NotInit  <<
```

随机检查多个`SqlCommand`，发现都是`SELECT COUNT(*) AS TotalNum FROM`这样的脚本，但是没有发现其他类型的`SQL`。

猜测是在获取文章、评论数量时没有关闭`SqlCommand`。由于这些`SqlCommand`已经处于`FinalizeQueue`，等待销毁，已经不存在栈中，所以无法从`!dso`中找到出事的代码行。
但是根据之前的分析，刚好有4个线程正在执行`SELECT COUNT(*) AS TotalNum FROM`操作，所以还是需要回到那些栈中分析。

发现出事的时候，有`using`的，所以正常来说应该会自动释放的

```
// BaiRong.Provider.Data.SqlServer.DatabaseDAO
public int GetIntResult(string sqlString)
{
	int result = 0;
	using (IDbConnection connection = base.GetConnection())
	{
		connection.Open();
		using (IDataReader dataReader = base.ExecuteReader(connection, sqlString))
		{
			if (dataReader.Read() && !dataReader.IsDBNull(0))
			{
				result = Convert.ToInt32(dataReader[0]);
			}
			dataReader.Close();
		}
	}
	return result;
}
```

这样子就有些蹊跷了。

_题外话：不过按照这个方法的意思，应该是获取一个数值而已，正常应该使用`ExecuteScalar`，而不是`ExecuteReader`。_

