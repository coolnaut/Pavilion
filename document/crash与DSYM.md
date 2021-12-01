# 1. Crash log 获取

## 1.1 XCode获取

> Window -> Devices And Simulators -> View Device Logs

此时XCode会拷贝设备的crash log文件，通过右上角的搜索筛选要查看的APP crash信息。

## 1.2 在项目中恢复

> Window -> Organizer，选择Crashes

选择版本后，会自动下载对应的crash log，选择open in project，就会在项目中打开，并查看具体的函数栈。

## 1.3 收集crash

在iOS程序崩溃时，一般我们是用Bugtags、Bugly、友盟等第三方收集崩溃，官方也有提供NSUncaughtExceptionHandler来收集crash信息。

# 2. Crash log 分析

## 2.1 日志信息

```
Incident Identifier: 1CC772D7-5C76-4D21-9674-DFE1137C13D2
CrashReporter Key:   a048c8ddbb093ae0e704d8fc4bbe3c3321267e25
Hardware Model:      iPhone13,2
Process:             UIBezierPathDemo [14435]
Path:                /private/var/containers/Bundle/Application/84D95884-36F4-4680-A56C-87EC6B0908CC/UIBezierPathDemo.app/UIBezierPathDemo
Identifier:          com.demo.UIBezierPathDemo
Version:             1 (1.0)
Code Type:           ARM-64 (Native)
Role:                Foreground
Parent Process:      launchd [1]
Coalition:           com.demo.UIBezierPathDemo [481]
```

最开始的几行是crash log的描述信息，比如：

- Incident Identifier 唯一标识id，这个和DSYM文件结合使用
- CrashReporter Key 处理过的设备标识符，同一台设备的crash log是一样的

## 2.2 时间及版本信息

```
Date/Time:           2021-08-26 17:50:09.8626 +0800
Launch Time:         2021-08-26 17:50:09.8308 +0800
OS Version:          iPhone OS 14.6 (18F72)
Release Type:        User
Baseband Version:    1.71.01
Report Version:      104
```

## 2.3 crash原因

```
Exception Type:  EXC_BREAKPOINT (SIGTRAP)
Exception Codes: 0x0000000000000001, 0x0000000104740d28
Termination Signal: Trace/BPT trap: 5
Termination Reason: Namespace SIGNAL, Code 0x5
Terminating Process: exc handler [5705]
```

一般的type如下：

```
#define EXC_CRASH       10  /* Abnormal process exit */
#define SIGKILL 9   /* kill (cannot be caught or ignored) */
#define EXC_CORPSE_NOTIFY   13  /* Abnormal process exited to corpse state */
```

在 Termination Reason 中，会有更明显的crash 原因。比如：

> Termination Reason: Namespace SPRINGBOARD, Code 0x8badf00d

0x8badf00d是一个很常见的Code，表示App启动时间过长或者主线程卡住时间过长，导致系统的WatchDog杀掉了当前App。

> EXC_BAD_ACCESS/SIGSEGV/SIGBUS 
>
> 内存访问错误:数组越界，访问一个已经释放的OC对象，尝试往readonly地址写入等
>
> EXC_CRASH/SIGABRT
>
> 进程异常的退出，最常见的是一些没有被处理Objective C/C++异常
>
> EXC_BREAKPOINT/SIGTRAP
>
> 调试器发生了这种异常

## 2.4  线程调用栈

```
0   libsystem_kernel.dylib        	0x00000001c45db334 0x1c45b2000 + 168756
1   libsystem_pthread.dylib       	0x00000001e2025a9c 0x1e2023000 + 10908
2   libsystem_c.dylib             	0x000000019f758b90 0x19f6e1000 + 490384
3   libc++abi.dylib               	0x00000001aaf52bb8 0x1aaf3f000 + 80824
4   libc++abi.dylib               	0x00000001aaf43ec8 0x1aaf3f000 + 20168
5   libobjc.A.dylib               	0x00000001aae5005c 0x1aae49000 + 28764
6   libc++abi.dylib               	0x00000001aaf51fa0 0x1aaf3f000 + 77728
7   libc++abi.dylib               	0x00000001aaf54eac 0x1aaf3f000 + 89772
8   libobjc.A.dylib               	0x00000001aae71904 0x1aae49000 + 166148
9   CoreFoundation                	0x000000019631d44c 0x196281000 + 640076
10  GraphicsServices              	0x00000001ad95b734 0x1ad958000 + 14132
11  UIKitCore                     	0x0000000198d98584 0x1981ce000 + 12363140
12  UIKitCore                     	0x0000000198d9ddf4 0x1981ce000 + 12385780
13  UIBezierPathDemo              	0x0000000104e3861c 0x104dfc000 + 247324
14  libdyld.dylib                 	0x0000000195fd9cf8 0x195fd8000 + 7416

Thread 1 name:  com.apple.uikit.eventfetch-thread
Thread 1:
0   libsystem_kernel.dylib        	0x00000001c45b64fc 0x1c45b2000 + 17660
1   libsystem_kernel.dylib        	0x00000001c45b5884 0x1c45b2000 + 14468
2   CoreFoundation                	0x0000000196323eb0 0x196281000 + 667312
3   CoreFoundation                	0x000000019631dd50 0x196281000 + 642384
4   CoreFoundation                	0x000000019631d360 0x196281000 + 639840
5   Foundation                    	0x000000019760afdc 0x197603000 + 32732
6   Foundation                    	0x000000019760aea8 0x197603000 + 32424
```

后面则是具体的线程调用栈了，如果是crash log符号文件，则需要借助 DYSM文件解析。

**XCode会尝试进行符号化，也就是不是像现在这种地址信息**

## 2.5 可执行文件

```
Binary Images:
0x104dfc000 - 0x104ecbfff UIBezierPathDemo arm64  <98d769befbe03394a8b0b0bdfab40d5e> /var/containers/Bundle/Application/E1603C7F-DFF6-497F-B24E-52D5DC2B1E47/UIBezierPathDemo.app/UIBezierPathDemo
0x104fd4000 - 0x104fdffff libBacktraceRecording.dylib arm64e  <39dff2236c56318cba0de17b5b64c56c> /Developer/usr/lib/libBacktraceRecording.dylib
0x104ff0000 - 0x105027fff libViewDebuggerSupport.dylib arm64e  
```

最后面则是可执行文件，可以看到加载的库。

Binary Images 的组成如下：

> 加载的地址范围 +  命中 + 架构 + UUID

# 2. 符号化

## 2.1 UUID

**（Universally Unique IDentifier）**

UUID 是由一组 32 位数的十六进制数字所构成。每一个可执行程序都有一个 build UUID 唯一标识。`.crash`日志包含发生 crash 的这个应用的 build UUID 以及 crash 发生时，应用加载的所有库文件的 build UUID。

>Binary Images、DSYM、.app的 UUID 是一致的
>
>不一致，则不能正确的解析出来

## 2.2 符号化工具

atos是一个命令行工具，可以用来符号化单个地址，命令格式如下：

```sh
atos -arch <Binary Architecture> -o <Path to dSYM file>/Contents/Resources/DWARF/<binary image name> -l <load address> <address to symbolicate>

# 比如：
atos -arch arm64 -o UIBezierPathDemo.app.dSYM/Contents/Resources/DWARF/UIBezierPathDemo -l 0x1000e4000 0x00000001000effdc
```

symbolicatecrash是XCode内置的符号化整个Crash Log的工具

```sh
/Applications/Xcode.app/Contents/SharedFrameworks/DVTFoundation.framework/Versions/A/Resources/symbolicatecrash /Download/log/test.crash youApp.dSYM >result.log

# 简化一下就是
symbolicatecrash xx.crash xx.dsym > xxx
```

> 可能会有环境变量的错误，尝试下面的方法：
>
> ```sh
> export DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer
> ```