# objc4 编译源码

```diff
@@ 复盘： 所有关于头文件找不到的错误都可以用错误2的办法解决, 选择的版本尽可能要跟当前系统一致。 @@
@@ 由于该开源代码期初使用的苹果内部系统，与发布出来的系统不一致造成了给中编译错误  @@
@@ * 主要是头文件找不到  @@
@@ * 只要把头文件给安排到位了， 头文件中函数实现都在系统中了， 不用担心没有实现的问题  @@
```
在 [Apple open Source](https://opensource.apple.com) 中找到与自己操作系统匹配的目录: macOS > 10.15 > 10.15.6 > 搜索objc4 > objc4-787.1 下载下来。
打开工程target list:
* objc, objc-simulator, objc-trampolines, objc4_tests, objc_executables, objcdt  
objc 依赖 objc-trampolines，编译objc必会编译objc-trampolines（中文意思是蹦床）

## 编译 
```diff
- ERROR 1 : unable to find sdk 'macosx.internal
Build Settings > Base SDK > 选择 macOS (objc,objc-trampolines)
```

```diff
- ERROR 2 : 'sys/reason.h' file not found
```
[google](https://www.google.com.hk) 搜索: "reason.h site:opensource.apple.com"  搜索结果链接：https://opensource.apple.com/source/xnu/xnu-3789.21.4/bsd/sys/reason.h.auto.html
在 xnu 的sys 中能找到相应头文件，在[Apple open Source](https://opensource.apple.com)中寻找当前系统对应的xnu, 把头文件获取下, 在工程下创建 include/sys 文件夹， 并且将include 文件夹作为系统头文件搜索路径。 Build Setting > Search Paths > debug 模式下添加：路径"$(SRCROOT)/include" (objc,objc-trampolines) 编译通过。

```diff
- ERROR 3 : 'mach-o/dyld_priv.h' file not found
[fix] 方法如error2 如法炮制 寻找当前系统对应的dyld
在 dyld-750.6 > include > mach-o > dyld_priv.h

- ERROR 4 : 'os/lock_private.h' file not found
方法如error2 如法炮制 寻找当前系统对应的libplatform
在 libplatform-220.100.1 > private > os > lock_private.h
```

```diff
- ERROR 5 : bridgeos(3.0) Expected ','  dyld_priv.h
也就是说 引入的头文件 dyld_priv.h 报错了
这里的解决办法是直接在dyld_priv.h文件中添加添加一行代码
#define __API_AVAILABLE(...)
```
```diff
- ERROR 6 : 
'os/base_private.h' file not found
'pthread/spinlock_private.h' file not found
'pthread/tsd_private.h' file not found
'System/machine/cpu_capabilities.h' file not found
'System/pthread_machdep.h' file not found
+ 根据错误2去找到对应的头文件即可
```

```diff
- ERROR 7 'CrashReporterClient.h' file not found
依然使用错误2的办法
注释掉CrashReporterClient.h文件中的 //#include_next <CrashReporterClient.h>
```

```diff
- ERROR 8 System/pthread_machdep.h 中 declaration 相关问题
- Typedef redefinition with different types ('int' vs 'volatile OSSpinLock' (aka 'volatile int'))
- Static declaration of '_pthread_has_direct_tsd' follows non-static declaration
解决办法：把重复的c函数以及报错的地方注释即可
```

```diff
- ERROR 9 'objc-shared-cache.h' file not found
依然使用错误2的办法
```

```diff
- ERROR 10 Use of undeclared identifier 'DYLD_MACOSX_VERSION_10_13'
依然使用错误2的办法,找到WTF/WTF-7604.5.6.0.1/wtf/spi/darwin/dyldSPI.h
在Project Headers > objc-os.h 中加入 #include <dyldSPI.h>
dyldSPI.h USE(APPLE_INTERNAL_SDK) 换成0 不然会报错，另外把该文件中的其他错误注释
```

```diff
- ERROR 11 Static_assert failed due to requirement 'bucketsMask >= ((unsigned long long)140737454800896ULL)' 
- "Bucket field doesn't have enough bits for arbitrary pointers."
解决办法：注释掉该断言
```
```diff
- ERROR 12 'kern/restartable.h' file not found
依然使用错误2的办法
```
```diff
- ERROR 13 Source > objc-errors.mm 86行 Declaration of 'crashlog_lock' has a different language linkage
处理办法：申明处添加 extern "C"
```


```diff
- ERROR 14 Use of undeclared identifier 'CRGetCrashLogMessage' 
- ERROR 15 Declaration of '_objc_fatal_with_reason' has a different language linkage
处理办法,在该文件添加
#define CRGetCrashLogMessage() 0
#define CRSetCrashLogMessage(x)
objc-errors.mm中中有其他错误就就注释掉最有少个} 也补上
方法申明处添加 extern "C" （extern "C" void _objc_fatal_with_reason）
```

```diff
- ERROR 15 '_static_assert' declared as an array with a negative size
注释掉
```

```diff
 - ERROR 16 can't open order file: /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/
 Developer/SDKs/MacOSX11.1.sdk/AppleInternal/OrderFiles/libobjc.order
  解决方法： Build Settings->Linking->Order File 改成$(SRCROOT)/libobjc.order
 ```
 
 ```diff
- ERROR 17 library not found for -lCrashReporterClient
删除
```
```diff
- ERROR 18 xcrun:1:1: sh -c '/Applications/Xcode.app/Contents/Developer/usr/bin/xcodebuild -sdk macosx.internal -find clang++ 2> /dev/null' failed with exit code 16384: (null) (errno=No such file or directory)
解决方法：将Target->Build Phases->Run-Script(markgc)里的内容macosx.internal改为macosx
```

编译通过。

