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
OS Thread Id: 0x3348c (0)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x29fc0 (1)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x32930 (2)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2f1ec (3)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x3383c (4)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x34220 (5)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2fecc (6)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x3073c (7)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x21188 (8)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2f4b8 (9)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2da7c (10)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2d200 (11)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x3146c (12)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x342c8 (13)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2eeac (14)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2b9f4 (15)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x24980 (16)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x3296c (17)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x31d1c (18)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2bdf4 (19)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2de3c (20)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x33208 (21)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2f7f0 (22)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x1e994 (23)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x33600 (24)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2d948 (25)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x33b44 (26)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x31540 (27)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2f3a0 (28)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2b67c (29)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2e080 (30)
        Child SP               IP Call Site
0000000002e2fb48 0000000077a8df6a [DebuggerU2MCatchHandlerFrame: 0000000002e2fb48] 
OS Thread Id: 0x31364 (31)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x319dc (32)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x193f4 (33)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2a844 (34)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x24fb8 (35)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2f7c0 (36)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2a958 (37)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x3064c (38)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x33f00 (39)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x288c4 (40)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x326a4 (41)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x3052c (42)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x336f0 (43)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x30858 (44)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x261b0 (45)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2cb0c (46)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2f714 (47)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2f940 (48)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2f0b0 (49)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x31e70 (50)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x1b8c (51)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2d090 (52)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2b564 (53)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x33b90 (54)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x30130 (55)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x34368 (56)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x1274c (57)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2ac08 (58)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x31528 (59)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2be04 (60)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x33064 (61)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x1241c (62)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2c604 (63)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2ddb8 (64)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2d93c (65)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2ff54 (66)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
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
OS Thread Id: 0x2d534 (70)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x31c48 (71)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2ca60 (72)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2f6e4 (73)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2ffe8 (74)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x33ce0 (75)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2db9c (76)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x323e4 (77)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2e6c0 (78)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2e920 (79)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x20b80 (80)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x30f98 (81)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2b928 (82)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x34278 (83)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
OS Thread Id: 0x2fab4 (84)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x2f9ec (85)
        Child SP               IP Call Site
GetFrameContext failed: 1
0000000000000000 0000000000000000 
OS Thread Id: 0x30a28 (86)
Unable to walk the managed stack. The current thread is likely not a 
managed thread. You can run !threads to get a list of managed threads in
the process
Failed to start stack walk: 80070057
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
