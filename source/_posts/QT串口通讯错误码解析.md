---
title: QT串口通讯错误码解析
abbrlink: 23455
date: 2023-05-01 16:43:26
updated:
tags: 
  - QT
  - 串口
categories: 
  - - QT
    - QSerialPort
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

## 前言

项目使用qt串口接口QSerialPort进行串口通讯，生产反馈产品提示串口通讯异常，查看log发现有3个错误码

分别是PermissionError(2)、WriteError(7)、UnknownError(11)。后面分析日志、了解生产情况找到了问题点。

> - PermissionError原因：串口占用，生产电脑上面先跑了串口调试工具导致串口打开失败
> - WriteError及UnknownError原因：（这连个错误是同时出现的额，先报WriteError后面紧跟UnknownError）通过对比日志发现是Windows自动睡眠后再次唤醒出现的。
>
> ![1682670665137](https://cdn.jsdelivr.net/gh/ouyujia/blog-img/img/1682670665137.png)

### QT串口通讯错误码及机翻

```c++
enum SerialPortError {
        NoError, 				// 没有错误
    
        DeviceNotFoundError,	// 试图打开不存在的设备时发生错误。
    
        PermissionError,		// 1、试图打开另一个进程已打开的设备时
    							// 2、用户没有足够的权限和凭据打开时发生错误。
    
        OpenError,				// 试图打开本进程中已打开的设备时发生错误。
    
        ParityError,			// 读取数据时，硬件检测到奇偶校验错误。
    							// 这个值已经过时了。我们强烈建议不要在新代码中使用它。
    
        FramingError,			// 读取数据时硬件检测到帧错误。
    							// 这个值已经过时了。我们强烈建议不要在新代码中使用它。
    
        BreakConditionError,	// 硬件在输入线上检测到断路情况。
    							// 这个值已经过时了。我们强烈建议不要在新代码中使用它。
    
        WriteError,				// 写入数据时发生I/O错误。
    
        ReadError,				// 读取数据时发生I/O错误。
    
        ResourceError,			// 当资源不可用时发生I/O错误，例如当设备意外地从系统中移除时。
    
        UnsupportedOperationError, //当前操作系统不支持或不禁止所请求的设备操作。
    
        UnknownError,			// 发生了一个无法识别的错误。
    
        TimeoutError,			// 发生超时错误。这个值是在QtSerialPort 5.2中引入的。
    
        NotOpenError			// 当执行的操作只有在设备打开的情况下才能成功执行时，会发生此错误。
            					// 这个值是在QtSerialPort 5.2中引入的。
    };
```

### 串口错误码处理

```c++
m_uart = new QSerialPort();  //不能指定父对象
//! 连接串口通信错误信号
connect(m_uart, SIGNAL(errorOccurred(QSerialPort::SerialPortError)), this,
        SLOT(errorFunc(QSerialPort::SerialPortError)));

//! 串口通信错误吗处理接口 
void UartPortThread::errorFunc(QSerialPort::SerialPortError error)
{
    if (error != QSerialPort::NoError && error != QSerialPort::TimeoutError
            && error != QSerialPort::NotOpenError && error != 				     QSerialPort::DeviceNotFoundError)
    {
        qDebug() << Q_FUNC_INFO << QString::number(error) << "error....";
        m_needReconnect = true;
        emit uartConnectState(false);
        setPortName("");
    }
}
```