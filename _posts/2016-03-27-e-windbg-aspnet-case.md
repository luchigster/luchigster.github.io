---
layout: post
title: WinDbg排查.Net问题系列-内存分析案例1
date: 2016-03-27
author: luchigster
comments: false
categories: [Blog]
tags: [AspNet,WinDbg,dumpheap,dumpobj,do,da,dumparray,objsize]
excerpt: 真对比较大的对象进行内存占用分析，发现问题是继承了一个通用父类导致，对开发提出修改意见后，得到了预想的内存占用减少效果。
---

工作上，需要分析一些内存占用问题，尝试找到问题所在并优化，所以弄来了个dump。
由于这是真实案例，所以本文对于敏感信息进行了屏蔽。

### 分析内存

```
0:092> !dumpheap -stat
省略几千行
Total 2030264 objects
Fragmented blocks larger than 0.5 MB:
            Addr     Size      Followed by
000000012f9ca960    3.7MB 000000012fd7cc48 System.Byte[]
000000018f9b68e0    3.8MB 000000018fd77e50 System.Threading.OverlappedDataCacheLine
00000001ff8898e8    2.5MB 00000001ffb12838 System.Byte[]
000000024f91ee08    3.9MB 000000024fd0b0a8 System.Collections.Generic.List`1[[CMP.Model.Organization, CMP.Ad]]
000000025f97b410    0.8MB 000000025fa500e0 System.Collections.Hashtable+SyncHashtable
000000025fa506b8    2.3MB 000000025fca9f18 System.Collections.Generic.List`1[[CMP.Model.Message, CMP.Ad]]
000000027f891d78    3.4MB 000000027fbf3710 System.Collections.Hashtable+SyncHashtable
```

随机抽取一个大对象，内存地址为 `000000024fd0b0a8`

```
0:092> !do 000000024fd0b0a8
Name:        System.Collections.Generic.List`1[[CMP.Model.Organization, CMP.Ad]]
MethodTable: 000007ff0091d4d0
EEClass:     000007fef42916e0
Size:        40(0x28) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\mscorlib\v4.0_4.0.0.0__b77a5c561934e089\mscorlib.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
000007fef470ae78  4000c1a        8      System.Object[]  0 instance 000000024f77c2f0 _items
000007fef470c858  4000c1b       18         System.Int32  1 instance               10 _size
000007fef470c858  4000c1c       1c         System.Int32  1 instance               10 _version
000007fef4705ac8  4000c1d       10        System.Object  0 instance 0000000000000000 _syncRoot
000007fef470ae78  4000c1e        0      System.Object[]  0   shared           static _emptyArray
                                 >> Domain:Value dynamic statics NYI 0000000000221320:NotInit dynamic statics NYI 0000000016980ae0:NotInit  <<
```

是一个列表，继续展开数组`_items`，地址为 `000000024f77c2f0`

```
0:092> !DumpArray /d 000000024f77c2f0
Name:        CMP.Model.Organization[]
MethodTable: 000007fef470ae78
EEClass:     000007fef428fdb8
Size:        160(0xa0) bytes
Array:       Rank 1, Number of elements 16, Type CLASS
Element Methodtable: 000007ff0753c538
[0] 000000024f77a950
[1] 000000024f77abe0
[2] 000000024f77ae70
[3] 000000024f77b100
[4] 000000024f77b390
[5] 000000024f77b620
[6] 000000024f77b8b0
[7] 000000024f77bb40
[8] 000000024f77bdd0
[9] 000000024f77c060
[10] null
[11] null
[12] null
[13] null
[14] null
[15] null
```

有10个对象，随机取最后一个对象，地址为 `000000024f77c060`

```
0:092> !do 000000024f77c060
Name:        CMP.Model.Organization
MethodTable: 000007ff0753c538
EEClass:     000007ff075151b0
Size:        128(0x80) bytes
File:        C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Temporary ASP.NET Files\root\cb60574a\e0f77744\assembly\dl3\d8c349e6\812cd32a_eb84d101\CMP.Ad.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
000007fef470d688  4000036       10       System.Boolean  1 instance                1 _hasUsed
000007fef3a2c568  4000037        8 ...meValueCollection  0 instance 000000024f77c0e0 _values
000007fef4727e30  40005a4       18         System.Int64  1 instance 1 _id
000007fef4727e30  40005a5       20         System.Int64  1 instance 634792633658564898 _organizationId
000007fef470d688  40005a6       11       System.Boolean  1 instance                1 _isEnabled
000007fef4727e30  40005a7       28         System.Int64  1 instance 634771907395829297 _adderId
000007fef4729748  40005a8       70      System.DateTime  1 instance 000000024f77c0d0 _addTime
000007fef470d688  40005a9       12       System.Boolean  1 instance                1 _release
000007fef470d688  40005aa       13       System.Boolean  1 instance                1 _isSendMessage
000007fef470d688  40005ab       5c       System.Boolean  1 instance                0 _isSendUnNormalTimeMsg
000007fef470c858  40005ac       14         System.Int32  1 instance              180 _repeatCardTimeSpan
000007fef4727e30  40005ad       30         System.Int64  1 instance 634801920635396445 _schoolTimeMainId
000007fef470d688  40005ae       5d       System.Boolean  1 instance                0 _isSendStatMessage
000007fef470d688  40005af       5e       System.Boolean  1 instance                0 _isSendNoSwingMessage
000007fef470d688  40005b0       5f       System.Boolean  1 instance                0 _isSendConsumeMessage
000007fef4727e30  40005b1       38         System.Int64  1 instance 634792631967710115 _cityId
000007fef4727e30  40005b2       40         System.Int64  1 instance 634792635250702157 _areaId
000007fef470d688  40005b3       60       System.Boolean  1 instance                0 _isSendStatDorm
000007fef470d688  40005b4       61       System.Boolean  1 instance                0 _isSendUnNormalTimeMsgToClassHeader
000007fef470d688  40005b5       62       System.Boolean  1 instance                1 _smsTimeEnabled
000007fef470d688  40005b6       63       System.Boolean  1 instance                0 _smsMachineEnabled
000007fef470d688  40005b7       64       System.Boolean  1 instance                0 _isVideo
000007fef470d688  40005b8       65       System.Boolean  1 instance                0 _isImage
000007fef470d688  40005b9       66       System.Boolean  1 instance                0 _isSendCorrespondMsgInSchoolTime
000007fef470d688  40005ba       67       System.Boolean  1 instance                0 _isSendWeekReport
000007ff0753c3d8  40005bb       48         System.Int32  1 instance                0 _week
000007fef470c858  40005bc       4c         System.Int32  1 instance                0 _hour
000007fef470c858  40005bd       50         System.Int32  1 instance                0 _minute
000007fef470d688  40005be       68       System.Boolean  1 instance                0 _isEnableLateEarlyout
000007fef470c858  40005bf       54         System.Int32  1 instance                0 _lateInSchool
000007fef470c858  40005c0       58         System.Int32  1 instance                0 _earlyOutSchool
000007fef470d688  40005c1       69       System.Boolean  1 instance                1 _isStatLiveInSchool
000007fef470d688  40005c2       6a       System.Boolean  1 instance                1 _isPlayVoice
```

看起来没什么问题，感觉不像一个大对象，先`!objsize`看看内容大小再说

```
0:092> !objsize 000000024f77c060
sizeof(000000024f77c060) =          824 (       0x338) bytes (CMP.Model.Organization)
```

总大小居然有824字节，再细看里面只有一个引用对象`_values`，地址为 `000000024f77c0e0`

```
0:092> !objsize 000000024f77c0e0
sizeof(000000024f77c0e0) =          696 (       0x2b8) bytes (System.Collections.Specialized.NameValueCollection)
```

不对劲，一个对象就占了内存这么大空间，猫腻，进去看看

```
0:092> !do 000000024f77c0e0
Name:        System.Collections.Specialized.NameValueCollection
MethodTable: 000007fef3a2c568
EEClass:     000007fef3728590
Size:        96(0x60) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_MSIL\System\v4.0_4.0.0.0__b77a5c561934e089\System.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
000007fef470d688  4000657       44       System.Boolean  1 instance                0 _readOnly
000007fef47101d8  4000658        8 ...ections.ArrayList  0 instance 000000024f77c140 _entriesArray
000007fef4713b98  4000659       10 ...IEqualityComparer  0 instance 000000029f4ef2f0 _keyComparer
000007fef4711c38  400065a       18 ...ections.Hashtable  0 instance 000000024f77c168 _entriesTable
000007fef3a2c4d8  400065b       20 ...e+NameObjectEntry  0 instance 0000000000000000 _nullKeyEntry
000007fef3a52cf8  400065c       28 ...se+KeysCollection  0 instance 0000000000000000 _keys
000007fef472a1c8  400065d       30 ...SerializationInfo  0 instance 0000000000000000 _serializationInfo
000007fef470c858  400065e       40         System.Int32  1 instance                2 _version
000007fef4705ac8  400065f       38        System.Object  0 instance 0000000000000000 _syncRoot
000007fef4726450  4000660      6a0 ...em.StringComparer  0   shared           static defaultComparer
                                 >> Domain:Value  0000000000221320:NotInit  0000000016980ae0:000000029f4ef2f0 <<
000007fef470ae78  400066c       48      System.Object[]  0 instance 0000000000000000 _all
000007fef470ae78  400066d       50      System.Object[]  0 instance 0000000000000000 _allKeys
```

原来这个`System.Collections.Specialized.NameValueCollection`是这么大的啊，进去`_entriesArray`看看有什么内容，地址为`000000024f77c140`

```
0:092> !DumpObj /d 000000024f77c140
Name:        System.Collections.ArrayList
MethodTable: 000007fef47101d8
EEClass:     000007fef42921e0
Size:        40(0x28) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\mscorlib\v4.0_4.0.0.0__b77a5c561934e089\mscorlib.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
000007fef470ae78  4000b84        8      System.Object[]  0 instance 000000024f77c2b0 _items
000007fef470c858  4000b85       18         System.Int32  1 instance                1 _size
000007fef470c858  4000b86       1c         System.Int32  1 instance                1 _version
000007fef4705ac8  4000b87       10        System.Object  0 instance 0000000000000000 _syncRoot
000007fef470ae78  4000b88      4b0      System.Object[]  0   shared           static emptyArray
                                 >> Domain:Value  0000000000221320:00000001ff4ed310 0000000016980ae0:000000029f4e9950 <<
```

还是一个对象，再进去`_items`，地址为`000000024f77c2b0`

```
0:092> !DumpArray /d 000000024f77c2b0
Name:        System.Object[]
MethodTable: 000007fef470ae78
EEClass:     000007fef428fdb8
Size:        64(0x40) bytes
Array:       Rank 1, Number of elements 4, Type CLASS
Element Methodtable: 000007fef4705ac8
[0] 000000024f77c290
[1] null
[2] null
[3] null
```

里面只有一个元素，地址为`000000024f77c290`，查下内容

```
0:092> !DumpObj /d 000000024f77c290
Name:        System.Collections.Specialized.NameObjectCollectionBase+NameObjectEntry
MethodTable: 000007fef3a2c4d8
EEClass:     000007fef3728528
Size:        32(0x20) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_MSIL\System\v4.0_4.0.0.0__b77a5c561934e089\System.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
000007fef4706970  4000669        8        System.String  0 instance 000000024f77a918 Key
000007fef4705ac8  400066a       10        System.Object  0 instance 000000024f77c240 Value
```

还是个对象，先看`Key`，地址为`000000024f77a918`

```
0:092> !DumpObj /d 000000024f77a918
Name:        System.String
MethodTable: 000007fef4706970
EEClass:     000007fef428eec8
Size:        50(0x32) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\mscorlib\v4.0_4.0.0.0__b77a5c561934e089\mscorlib.dll
String:      provinceCode
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
000007fef470c858  40000ed        8         System.Int32  1 instance               12 m_stringLength
000007fef470b398  40000ee        c          System.Char  1 instance               70 m_firstChar
000007fef4706970  40000ef       10        System.String  0   shared           static Empty
                                 >> Domain:Value  0000000000221320:00000001ff4e0488 0000000016980ae0:00000001ff4e0488 <<
```

是个字符串`"provinceCode"`，再看`Value`，地址为`000000024f77c240`

```
0:092> !DumpObj /d 000000024f77c240
Name:        System.Collections.ArrayList
MethodTable: 000007fef47101d8
EEClass:     000007fef42921e0
Size:        40(0x28) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\mscorlib\v4.0_4.0.0.0__b77a5c561934e089\mscorlib.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
000007fef470ae78  4000b84        8      System.Object[]  0 instance 000000024f77c268 _items
000007fef470c858  4000b85       18         System.Int32  1 instance                1 _size
000007fef470c858  4000b86       1c         System.Int32  1 instance                1 _version
000007fef4705ac8  4000b87       10        System.Object  0 instance 0000000000000000 _syncRoot
000007fef470ae78  4000b88      4b0      System.Object[]  0   shared           static emptyArray
                                 >> Domain:Value  0000000000221320:00000001ff4ed310 0000000016980ae0:000000029f4e9950 <<
```

还是个对象，再进去数组`_items`看看，地址为`000000024f77c268`

```
0:092> !DumpArray /d 000000024f77c268
Name:        System.Object[]
MethodTable: 000007fef470ae78
EEClass:     000007fef428fdb8
Size:        40(0x28) bytes
Array:       Rank 1, Number of elements 1, Type CLASS
Element Methodtable: 000007fef4705ac8
[0] 000000024f77c220
```

只有一个元素，地址为`000000024f77c220`

```
0:092> !DumpObj /d 000000024f77c220
Name:        System.String
MethodTable: 000007fef4706970
EEClass:     000007fef428eec8
Size:        30(0x1e) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\mscorlib\v4.0_4.0.0.0__b77a5c561934e089\mscorlib.dll
String:      11
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
000007fef470c858  40000ed        8         System.Int32  1 instance                2 m_stringLength
000007fef470b398  40000ee        c          System.Char  1 instance               31 m_firstChar
000007fef4706970  40000ef       10        System.String  0   shared           static Empty
                                 >> Domain:Value  0000000000221320:00000001ff4e0488 0000000016980ae0:00000001ff4e0488 <<
```

搞这么久，原来就是一个字符串`"11"`

整个`CMP.Model.Organization`里面的`_values`只包含了一个字符串的键值对`provinceCode:11`，
结果却占用了整个对象`696/824 = 0.84466019417476`，近`85%`的内存空间。再看这个类定义里面的这个`_values`是由于继承了`ClassBase`这个类而产生，
而继承这个`ClassBase`的`Model`类不胜枚举，也难怪内存占用这么厉害了。

### 从源码查原因

再反编译分析`ClassBase`里面是如何实现的

```
using System;
using System.Collections.Specialized;

[Serializable]
public class ClassBase
{
        private bool _hasUsed = true;
        private NameValueCollection _values = new NameValueCollection();
        public virtual string this[string index]
        {
                get
                {
                        return this.GetValue(index);
                }
                set
                {
                        this.SetValue(index, value);
                }
        }
        public bool HasUsed
        {
                get
                {
                        return this._hasUsed;
                }
                set
                {
                        this._hasUsed = value;
                }
        }
        public virtual string GetValue(string key)
        {
                if (this._values[key] != null)
                {
                        return this._values[key];
                }
                return string.Empty;
        }
        public virtual void SetValue(string key, string value)
        {
                this._values[key] = value;
        }
}
```

继承了这个类的每个对象都会创建一个`NameValueCollection`，从上面知道，这个对象占用内存空间比较大。
对于字符串字段比较少的`Model`，性价比就不高了。仅当字符串字段相当多的时候，可以用来存储字符串。
好处可能是，一次性初始化对象时，可以用`NameValueCollection`来将所有字符串连续存储（其实未必，连续存储的是字符串的指针而已），并提供默认值。
但是性能真的有提高么？继续深入。

### 性能测试

从里面提取刚才的对象`CMP.Model.Organization`，拷贝一份为`CMP.Model.OrganizationV2`，移除对`ClassBase`的依赖，并修改字符串属性为`public string ProvinceCode { get; set; } = string.Empty;`

对类初始化进行测试，不出所料，继承了`ClassBase`的类在运行速度上拖后腿。

```
int count = 1;
Console.WriteLine("运行次数\t旧对象运行时间(毫秒)\t新对象运行时间(毫秒)");
for (int pow = 0; pow < 7; pow++)
{
    count = (int)Math.Pow(10, pow);

    var now = DateTime.Now;
    for (int i = 0; i < count; i++)
    {
        var d = new Organization { ProvinceCode = "11" };
    }
    double s1 = DateTime.Now.Subtract(now).TotalMilliseconds;

    now = DateTime.Now;
    for (int i = 0; i < count; i++)
    {
        var d = new OrganizationV2 { ProvinceCode = "11" };
    }

    double s2 = DateTime.Now.Subtract(now).TotalMilliseconds;
    Console.WriteLine(count + "\t" + s1 + "\t" + s2);
}
```

分不同的数量级测试，随机5次测试结果显示，继承了`ClassBase`的类初始化比干净的类慢，次数越多，对比更明显（数据量少时偶尔会有偏差）

```
运行次数 旧对象运行时间(毫秒)    新对象运行时间(毫秒)
1       2.0014               0
10      0                    0
100     0                    0
1000    2.0015               0
10000   17.0121              0
100000  151.3354             5.0024
1000000 1315.2548            34.0245

运行次数 旧对象运行时间(毫秒)    新对象运行时间(毫秒)
1       1.0019               1.0016
10      0                    0
100     0                    0
1000    0.9916               0
10000   19.0124              0
100000  155.1168             3.5025
1000000 1353.4418            40.0325

运行次数 旧对象运行时间(毫秒)    新对象运行时间(毫秒)
1       1.0007               0
10      0                    0
100     0                    1.0004
1000    1.0013               0
10000   14.0088              0
100000  163.6223             3.505
1000000 1322.3125            39.5931

运行次数 旧对象运行时间(毫秒)    新对象运行时间(毫秒)
1       1.0018               0
10      0                    0
100     0                    0
1000    2.0005               1.0007
10000   23.0216              0.9955
100000  156.7727             3.0018
1000000 1473.2277            34.0245

运行次数 旧对象运行时间(毫秒)    新对象运行时间(毫秒)
1       1.0001               1.0007
10      0                    0
100     0                    0
1000    2.0012               0
10000   20.0142              1.001
100000  168.1295             5.003
1000000 1354.638             34.5249
```

初始化慢，未必代表读取慢，于是，再做了一次测试，分别初始化一个对象，然后统计读取属性的速度

```
var d1 = new Organization { ProvinceCode = "11" };
var d2 = new OrganizationV2 { ProvinceCode = "11" };

int count = 1;
Console.WriteLine("运行次数\t旧对象读取时间(毫秒)\t新对象读取时间(毫秒)");
for (int pow = 0; pow < 7; pow++)
{
    count = (int)Math.Pow(10, pow);

    var now = DateTime.Now;
    for (int i = 0; i < count; i++)
    {
        var d = d1.ProvinceCode;
    }
    double s1 = DateTime.Now.Subtract(now).TotalMilliseconds;

    now = DateTime.Now;
    for (int i = 0; i < count; i++)
    {
        var d = d2.ProvinceCode;
    }

    double s2 = DateTime.Now.Subtract(now).TotalMilliseconds;
    Console.WriteLine(count + "\t" + s1 + "\t" + s2);
}
```

同样随机测试5次，对比结果同样，也是继承了`ClassBase`的类，读取字符串速度比较慢得多，测试次数越高，对比更明显。

```
运行次数 旧对象读取时间(毫秒)    新对象读取时间(毫秒)
1       0.999                0
10      0                    0
100     0                    0
1000    1.003                0
10000   10.0139              0
100000  97.0906              0
1000000 862.3558             4.0125

运行次数 旧对象读取时间(毫秒)    新对象读取时间(毫秒)
1       0                    1.0022
10      0                    0
100     0                    0
1000    1.0007               0
10000   12.0074              0
100000  106.6033             0
1000000 872.2289             4.0032

运行次数 旧对象读取时间(毫秒)    新对象读取时间(毫秒)
1       2.0014               0
10      0                    0
100     0                    0
1000    1.0007               0
10000   11.0075              0
100000  110.0833             1.0004
1000000 878.7267             4.0037

运行次数 旧对象读取时间(毫秒)    新对象读取时间(毫秒)
1       1.0081               0
10      0                    0
100     0                    0
1000    0.9936               0
10000   11.0073              1.0195
100000  98.0553              0
1000000 877.1091             4.0122

运行次数 旧对象读取时间(毫秒)    新对象读取时间(毫秒)
1       1.0008               0
10      0                    0
100     0                    0
1000    1.0007               0
10000   13.0127              0
100000  118.0898             1.0005
1000000 967.3941             4.004
```

这个结果没有出意外，但是没有整明白，为什么`ClassBase`里面有一个`NameValueCollection`，就会导致性能变差。

反编译`System.dll`没有看到对应托管代码的实现，猜想是微软调用非托管代码实现的。
采用键值对的`NameValueCollection`，联想到类似的`Hashtable`、`Dictionary`，在查询数据时，取决于查询`Key`的算法效率，
在托管代码里面是`GetHashCode()`，`System.Object`对象有默认实现。猜测`NameValueCollection`底层也是类似的实现。
但是归根到底，属性`{get;set;}`器在编译器已经做了优化，而键查找无论怎么优化，在运行时始终性能要更差。

### 结论

测试结果总结：
    
* 空间上，继承了`ClassBase`的类比不继承的占内存更多
* 初始化速度上，继承了`ClassBase`的类，比不继承的运行时间更长
* 读取速度上，继承了`ClassBase`的类，比不继承的运行时间更长

于是对开发提出了修改建议：

* 移除所有`Model`类对`ClassBase`的依赖（可全部替换为一个统一的空类）
* 所有字符串属性改为带默认值的形式：`{get;set;}=string.Empty;`
 
* * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * *

开发按照建议，对主要的几个`Model`类做了修改，然后再作一次对比测试。
由于需要准确测试对象修改后的效果，所以在开发环境上作隔离测试，用相同的用户对象来比较。 

### 开发修改前

```
0:065> !dumpheap -stat -type User
Statistics:
              MT    Count    TotalSize Class Name
00007ff85642f980        1          144 CMP.DataModel.User
Total 1 objects
Fragmented blocks larger than 0.5 MB:
            Addr     Size      Followed by
0000025617ef7a90    1.4MB 000002561806a678 System.Byte[]
00000256980a96f8    1.1MB 00000256981be580 Bid+BindingCookie
```

看到一个用户对象，显示地址出来

```
0:065> !DumpHeap /d -mt 00007ff85642f980
         Address               MT     Size
0000025697fa4dd8 00007ff85642f980      144     

Statistics:
              MT    Count    TotalSize Class Name
00007ff85642f980        1          144 CMP.DataModel.User
Total 1 objects
Fragmented blocks larger than 0.5 MB:
            Addr     Size      Followed by
0000025617ef7a90    1.4MB 000002561806a678 System.Byte[]
00000256980a96f8    1.1MB 00000256981be580 Bid+BindingCookie
```

找到这个用户对象的地址`00007ff85642f980`

```
0:065> !DumpObj /d 0000025697fa4dd8
Name:        CMP.DataModel.User
MethodTable: 00007ff85642f980
EEClass:     00007ff8564094a0
Size:        144(0x90) bytes
File:        C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Temporary ASP.NET Files\app\85571114\74b0b8a4\assembly\dl3\3efafa25\d9352e19_7586d101\CMP.BASE.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007ff8b3c2a2f0  4000036       10       System.Boolean  1 instance                1 _hasUsed
00007ff8a041dac8  4000037        8 ...meValueCollection  0 instance 0000025697fa4e90 _values
00007ff8b3c08548  40012ce       18        System.String  0 instance 0000025697fa5140 _password
00007ff8b3c06390  40012cf       20         System.Int64  1 instance 634872879982677108 _userId
00007ff856267d78  40012d0       38         System.Int32  1 instance               10 _userType
00007ff8b3c06390  40012d1       28         System.Int64  1 instance 1 _customId
00007ff8b3c2a2f0  40012d2       11       System.Boolean  1 instance                1 _isEnabled
00007ff8b3c06390  40012d3       30         System.Int64  1 instance 634780509405312487 _adderId
00007ff8b3c27bb8  40012d4       40      System.DateTime  1 instance 0000025697fa4e18 _addTime
00007ff8b3c27bb8  40012d5       48      System.DateTime  1 instance 0000025697fa4e20 _updateTime
00007ff8b3c2a2f0  40012d6       12       System.Boolean  1 instance                1 _release
00007ff8562e9c20  4001363       50         System.Int32  1 instance                0 _sex
00007ff8b3c27bb8  4001364       68      System.DateTime  1 instance 0000025697fa4e40 _birthday
00007ff8562eb000  4001365       54         System.Int32  1 instance                0 _property
00007ff85642ac80  4001366       58         System.Int32  1 instance                9 _payType
00007ff85642cb40  4001367       5c         System.Int32  1 instance               10 _state
00007ff8b3c0af70  4001368       60         System.Int32  1 instance                0 _useType
00007ff8b3c05b60  4001369       70 ...olean, mscorlib]]  1 instance 0000025697fa4e48 _powerUserDisplayMobile
00007ff856fad3b0  4001361       78 ...ngLong.CMP.BASE]]  0 instance 0000025697fa4e68 _positions
00007ff8b3c08548  4001362       80        System.String  0 instance 0000000000000000 <SyncUserId>k__BackingField
0:065> !objsize 0000025697fa4dd8
sizeof(0000025697fa4dd8) = 4632 (0x1218) bytes (CMP.DataModel.User)
```

现在大小是`4632`字节。

```
0:065> !objsize 0000025697fa4e90
sizeof(0000025697fa4e90) = 4344 (0x10f8) bytes (System.Collections.Specialized.NameValueCollection)
```

其中`_values`占用了`4344`字节，可见是因为这个`_values`导致的。

### 开发修改后

使用同样操作方法。

```
0:069> !do 0000021daf717cc8
Name:        CMP.DataModel.User
MethodTable: 00007ff8562db608
EEClass:     00007ff8562aab68
Size:        280(0x118) bytes
File:        C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Temporary ASP.NET Files\root\aa796ebd\cba45ee8\assembly\dl3\e2fcee26\efe73e29_7286d101\CMP.BASE.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007ff8b3c2a2f0  4001230        8       System.Boolean  1 instance                0 <HasUsed>k__BackingField
00007ff8b3c08548  40012d0       10        System.String  0 instance 0000021daf71e640 _password
00007ff8b3c06390  40012d1       48         System.Int64  1 instance 634872879982677108 _userId
00007ff856110020  40012d2       60         System.Int32  1 instance               10 _userType
00007ff8b3c06390  40012d3       50         System.Int64  1 instance 1 _customId
00007ff8b3c2a2f0  40012d4        9       System.Boolean  1 instance                1 _isEnabled
00007ff8b3c06390  40012d5       58         System.Int64  1 instance 634780509405312487 _adderId
00007ff8b3c27bb8  40012d6       68      System.DateTime  1 instance 0000021daf717d30 _addTime
00007ff8b3c27bb8  40012d7       70      System.DateTime  1 instance 0000021daf717d38 _updateTime
00007ff8b3c2a2f0  40012d8        a       System.Boolean  1 instance                1 _release
00007ff8b3c08548  40012d9       18        System.String  0 instance 0000021daf71e5b8 <Name>k__BackingField
00007ff8b3c08548  40012da       20        System.String  0 instance 0000021daf71e5d8 <Account>k__BackingField
00007ff8b3c08548  40012db       28        System.String  0 instance 0000021daf71e608 <CustomAccount>k__BackingField
00007ff8b3c08548  40012dc       30        System.String  0 instance 000002202f211420 <ImportPassword>k__BackingField
00007ff8b3c08548  40012dd       38        System.String  0 instance 0000021daf71e690 <AccountOA>k__BackingField
00007ff8b3c08548  40012de       40        System.String  0 instance 0000021daf71e6b8 <AreaCode>k__BackingField
00007ff856193ef0  400136e       d8         System.Int32  1 instance                0 _sex
00007ff8b3c27bb8  400136f       f0      System.DateTime  1 instance 0000021daf717db8 _birthday
00007ff8561952d0  4001370       dc         System.Int32  1 instance                0 _property
00007ff8562d6860  4001371       e0         System.Int32  1 instance                9 _payType
00007ff8562d8750  4001372       e4         System.Int32  1 instance               10 _state
00007ff8b3c0af70  4001373       e8         System.Int32  1 instance                0 _useType
00007ff8b3c05b60  4001374       f8 ...olean, mscorlib]]  1 instance 0000021daf717dc0 _powerUserDisplayMobile
00007ff8b3c08548  4001375       78        System.String  0 instance 000002202f211420 _mobile
00007ff8b3c08548  4001376       80        System.String  0 instance 000002202f211420 _mobile2
00007ff8b3c08548  4001377       88        System.String  0 instance 000002202f211420 <CompanyPhone>k__BackingField
00007ff8b3c08548  4001378       90        System.String  0 instance 000002202f211420 <Fax>k__BackingField
00007ff8b3c08548  4001379       98        System.String  0 instance 000002202f211420 <Email>k__BackingField
00007ff8b3c08548  400137a       a0        System.String  0 instance 000002202f211420 <Code>k__BackingField
00007ff8b3c08548  400137b       a8        System.String  0 instance 000002202f211420 <Sign>k__BackingField
00007ff8b3c08548  400137c       b0        System.String  0 instance 000002202f211420 <NickName>k__BackingField
00007ff8b3c08548  400137d       b8        System.String  0 instance 000002202f211420 <IdCardNum>k__BackingField
00007ff8b3c08548  400137e       c0        System.String  0 instance 000002202f211420 <UserCode>k__BackingField
00007ff8b3c08548  400137f       c8        System.String  0 instance 000002202f211420 <UserNumber>k__BackingField
00007ff8b3c08548  4001380       d0        System.String  0 instance 000002202f211420 <Remark>k__BackingField
00007ff856ed78a0  400136c      100 ...ngLong.CMP.BASE]]  0 instance 0000021daf717de0 _positions
00007ff8b3c08548  400136d      108        System.String  0 instance 000002202f211420 <SyncUserId>k__BackingField
0:069> !objsize 0000021daf717cc8
sizeof(0000021daf717cc8) = 744 (0x2e8) bytes (CMP.DataModel.User)
```

看到了，同样的数据，精简到`744`字节了

### 说在后面

简单计算下，一个用户对象节省`4632-744=3888`字节，`10`万用户节省`3888*100000=388,800,000`字节，约`388MB`。
一个对象就可以达到这个效果，这个算是很可观的成效了，反观软件开发中的坑真的多。
