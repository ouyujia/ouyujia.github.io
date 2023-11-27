---
title: WindowsXP USB通信
abbrlink: 23455
date: 2022-05-05 16:43:26
updated:
tags: 
  - WindowsXP
  - I/O设备
  - usb
categories: usb
keywords:
description:
top_img:
comments:
cover: https://cdn.jsdelivr.net/gh/ouyujia/blogImg/img/202205051748864.jpg
toc:
toc_number:
toc_style_simple:
copyright:
copyright_author:
copyright_author_href:
copyright_url:
copyright_info:
mathjax:
katex:
aplayer:
highlight_shrink:
aside:
---



> 需要包含的库文件:hid.lib、SetupAPI.lib
>
> 需要包含的头文件:hid.h、SetupAPI.h



## 前言

- 因为项目需要支持Windows XP系统，项目之前使用的USB通讯方法在windwos xp系统缺少相应的库文件，故选择使用Windows API 来建立USB通讯。

## 主要函数解析(可参考windows API)

创建或打开文件或者I/O设备。

函数定义如下：

```C++
HANDLE CreateFile(
  [in]           LPCSTR                lpFileName,
  [in]           DWORD                 dwDesiredAccess,
  [in]           DWORD                 dwShareMode,
  [in, optional] LPSECURITY_ATTRIBUTES lpSecurityAttributes,
  [in]           DWORD                 dwCreationDisposition,
  [in]           DWORD                 dwFlagsAndAttributes,
  [in, optional] HANDLE                hTemplateFile
);
// 如果函数成功，则返回值是指定文件、设备、命名管道或邮件槽的打开句柄。如果函数失败，则返回值为 INVALID_HANDLE_VALUE。 要获得更多的错误信息，请调用 GetLastError。
```

常用参数如下

```c++
HANDLE handle = CreateFile(
devDetail->DevicePath,                 //设备路径
GENERIC_READ | GENERIC_WRITE,          //访问方式
FILE_SHARE_READ | FILE_SHARE_WRITE,    //共享模式
NULL,
OPEN_EXISTING,                         //文件不存在时，返回失败
FILE_FLAG_OVERLAPPED,                  //以重叠（异步）模式打开
NULL); 
```

> 注意点：
>
> 1. 在这里，CreateFile用于打开HID设备，其中设备路径通过函数SetupDiGetDeviceInterfaceDetail取得。CreateFile有以下几点需要留意：
> 2. 访问方式： 假如是系统独占设备，例如鼠标、键盘等等，应将此参数设置为0，否则后续函数操纵将失败（譬如HidD_GetAttributes）；也就是说，不能对独占设备进行除了查询以外的任何操纵，所以能够使用的函数也是很有限的，下文的一些函数并不一定适合这些设备。在此顺便列出MSDN上关于此参数的说明：（ If this parameter is zero, the application can query file and device attributes without accessing the device. This is useful if an application wants to determine the size of a floppy disk drive and the formats it supports without requiring a floppy in the drive. It can also be used to test for the file's or directory's existence without opening it for read or write access。)
> 3. 重叠（异步）模式：此参数并不会在此处表现出明显的意义，它主要是对后续的WriteFile，ReadFile有影响。假如这里设置为重叠（异步）模式，那么在使用WriteFile，ReadFile时也应该使用重叠（异步）模式，反之亦然。这首先要求WriteFile，ReadFile的最后一个参数不能为空（NULL）。否则，便会返回87（参数错误）错误号。当然，87号错误并不代表就是此参数不正确，更多的信息将在具体讲述这两个函数时指出。此参数为0时，代表同步模式，即WriteFile，ReadFile操纵会在数据处理完成之后才返回，否则阻塞在函数内部.  

------

从指定的文件或输入/输出 (I/O) 设备读取数据。

函数定义如下：

```C++
BOOL ReadFile(
  [in]                HANDLE       hFile,
  [out]               LPVOID       lpBuffer,
  [in]                DWORD        nNumberOfBytesToRead,
  [out, optional]     LPDWORD      lpNumberOfBytesRead,
  [in, out, optional] LPOVERLAPPED lpOverlapped
);
// 如果函数成功，则返回值为非零 (TRUE) 。如果函数失败或异步完成，则返回值为零， (FALSE) 。 若要获取扩展的错误信息，请调用 GetLastError 函数。注意GetLastError 代码ERROR_IO_PENDING不是失败;它指定读取操作正在异步等待完成。 
```

常用参数如下：

```C++
bool bReadResult = ReadFile(
hDev,                       //设备句柄，即CreateFile的返回值
recvBuffer,                 //用于接收数据的buffer
IN_REPORT_LEN,              //要读取数据的长度
&recvBytes,                 //实际收到的数据的字节数
&readOverlapped);           //异步模式 (为NULL表示同步模式)
```

> 注意点:
>
> 1. ReadFile的调用不会引起设备的任何反应，即HID设备与主机之间的中断IN传输不与ReadFile打交道。实际上主机会在最大间隔时间（由设备的端点描述符来指定）内轮询设备，发出中断IN传输的请求。“读取”即意味着从某个buffer里面取回数据，实际上这个buffer就是HID设备驱动中的buffer。这个buffer的大小可以通过HidD_SetNumInputBuffers来改变。在XP上缺省值是32（个报告）。
> 2. 读取的数据对象是输进报告，也即通过中断输进管道传进的数据。所以，假如设备不支持中断IN传输，那么是无法使用此函数来得到预期结果的。实际上这种情况不可能在HID中出现，由于协议指明了至少要有一个中断IN端点。
> 3. IN_REPORT_LEN代表要读取的数据的长度（实际的数据正文+一个byte的报告ID），这里是一个常数，主要是由于设备通讯协议决定，按照协议读取多少数据（也就是报告的长度）；不过也可以通过另外的函数（HidD_GetPreparsedData）来事先取得报告的长度，这里不做具体讨论。此参数假如设置过大，不会有实质性的错误，在recvBytes参数中会输出实际读到的长度；假如设置过小，即小于报告的长度，会返回1784号错误（用户提供的buffer无效）。
> 4. 关于异步模式。前面已经提过，此参数的设置必须与CreateFile时的设置相对应，否则会返回87号错误（参数错误）。假如不需要异步模式，此参数需置为NULL。在这种情况下，ReadFile会一直等待直到数据读取成功，所以会阻塞住程序确当前过程。  

------

将数据写入指定的文件或输入/输出 (I/O) 设备。

函数定义如下：

```C++
BOOL WriteFile(
  [in]                HANDLE       hFile,
  [in]                LPCVOID      lpBuffer,
  [in]                DWORD        nNumberOfBytesToWrite,
  [out, optional]     LPDWORD      lpNumberOfBytesWritten,
  [in, out, optional] LPOVERLAPPED lpOverlapped
);
// 如果函数成功，则返回值为非零 (TRUE) 。如果函数失败或异步完成，则返回值为零， (FALSE) 。 若要获取扩展的错误信息，请调用 GetLastError 函数。注意GetLastError 代码ERROR_IO_PENDING不是失败;它指定写入操作正在异步完成。 
```

常用参数如下：

```C++
bool bWriteResult = WriteFile(
hDev,                                 //设备句柄，即CreateFile的返回值
reportBuf,                            //存有待发送数据的buffer
OUT_REPORT_LEN,           			  //待发送数据的长度
&sendBytes,                           //实际收到的数据的字节数
&writeOverlapped);                    //异步模式 (为NULL表示同步模式)
```

> 注意点：
>
> 1. 与ReadFile不同，WriteFile函数被调用后，固然也是经过驱动程序，但是终极会反映到设备中。也就是说，调用WriteFile后，设备会接收到输出报告的请求。假如设备使用了中断OUT传输，则WriteFile会通过中断OUT管道来进行传输；否则会使用SetReport请求通过控制管道来传输。
> 2. OUT_REPORT_LEN代表要写进的数据长度（实际的数据正文+一个byte的报告ID）。硬度计假如大于实际报告的长度，则使用实际报告长度；假如小于实际报告长度，会返回1784号错误（用户提供的buffer无效）。
> 3. reportBuf[0]必须存有待发送报告的ID，并且此报告ID指示的必须是输出报告，否则会返回87号错误（参数错误）。这种情况可能轻易被程序员忽略，结果不知错误号所反映的是什么，网上也经常有类似疑问的帖子。顺便指出，输进报告、输进报告、特征报告这些报告类型，是反映在HID设备的报告描述符中。后文将做举例讨论。
> 4. 关于异步模式。前面已经提过，此参数的设置必须与CreateFile时的设置相对应，否则会返回87号错误（参数错误）。假如不需要异步模式，此参数需置为NULL。在这种情况下，WriteFile会一直等待直到数据读取成功，所以会阻塞住程序确当前过程。  

------

参考：

[(70条消息) HID ReadFile()使用_Phenixyf的博客-CSDN博客](https://blog.csdn.net/phenixyf/article/details/39929033)

[(70条消息) 关于GetOverlappedResult函数的一些知识_kemaWCZ的博客-CSDN博客](https://blog.csdn.net/kemawcz/article/details/50704978)

