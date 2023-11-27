---
title: 本地套接字QLocalSocket
abbrlink: 1441
date: 2023-11-24 19:00:41
updated:
tags: 
  - QT
  - Socket
categories: 
  - - QT
    - net
keywords:
description: 基于QLocalSocket的本地套接字接口
top_img:
comments:
cover:
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

> 工作总用到本地两个进程间通讯，共享内存、信号量、文件、网络通讯都有简单使用过，尝试使用新方法：QLocalSocket，因此自己简单封装了通讯接口。

### server

.pri

```c++
HEADERS += \
    $$PWD/localsocketserver.h

SOURCES += \
    $$PWD/localsocketserver.cpp
```

.h

```c++
#ifndef LOCALSOCKETSERVER_H
#define LOCALSOCKETSERVER_H

#include <QObject>
#include <QLocalSocket>
#include <QByteArray>

#ifdef Q_OS_WIN
#define MAX_LEN 4096
#endif

class QLocalServer;
class QLocalSocket;

class LocalSocketServer : public QObject
{
    Q_OBJECT
public:

    enum ClientCmd
    {
        CLIENT_CMD_UNKNOWN = -1,
        //! 心跳
        CLIENT_CMD_HEARTBEAT,
        //! 测试http连接
        CLIENT_CMD_TEST_HTTP_CONNECT,
        //! 自动同步
        CLIENT_CMD_AUTO_UPLOAD,
        //! 停止自动同步
        CLIENT_CMD_STOP_AUTO_UPLOAD,
        CLIENT_CMD_END
    };

    explicit LocalSocketServer(QObject *parent = nullptr);
    ~LocalSocketServer();

signals:
    void signalCmdHandle(ClientCmd cmd);

private:
    bool listen();

    void handleCmd(const QJsonValue &value);

    void sendData(const QJsonObject &jsonObj);

    void addLog(const QString &info);

    void sendCmd(ClientCmd cmd);

    ClientCmd cmdConvert(const QJsonObject &jsonObj);

private slots:
    void newConnection();
    void readData();
    void errorInformation(QLocalSocket::LocalSocketError state);
    void handleReceiveCmd(LocalSocketServer::ClientCmd cmd, bool ok);

private:
    QLocalServer *m_localServer;
    QLocalSocket *m_localSocket;
    QByteArray m_receiveData;
};

#endif // LOCALSOCKETSERVER_H
```

.cpp

```c++
#include "localsocketserver.h"
#include <QLocalServer>
#include <QDataStream>
#include <QJsonDocument>
#include <QJsonObject>

LocalSocketServer::LocalSocketServer(QObject *parent)
    : QObject(parent),
      m_localServer(nullptr),
      m_localSocket(nullptr)
{
    m_receiveData.clear();

    listen();
}

LocalSocketServer::~LocalSocketServer()
{
    if(m_localServer != nullptr)
    {
        m_localServer->close();
        m_localServer->deleteLater();
        m_localServer = nullptr;
    }

    if(m_localSocket != nullptr)
    {
        m_localSocket->disconnectFromServer();
        m_localSocket->deleteLater();
        m_localSocket = nullptr;
    }
}

bool LocalSocketServer::listen()
{
    if (m_localServer == nullptr)
    {
        m_localServer = new QLocalServer(this);
    }

    // 设置通讯地址，类似tcp的客户端IP及地址
    m_localServer->listen("Pulmolab");
    connect(m_localServer, &QLocalServer::newConnection, this, &LocalSocketServer::newConnection);
    return true;
}

void LocalSocketServer::handleCmd(const QJsonValue &value)
{
    if(value.isString()
            && (value.toString().indexOf("queryHttpConnect:") >= 0))
    {
        QJsonObject jsonObj;
        jsonObj.insert("cmd", "queryHttpConnect");

        sendCmd(cmdConvert(jsonObj));
        // 回复信息
        sendData(jsonObj);
    }

    if(value.isString() && value.toString() == "autoUpload:true")
    {
        QJsonObject jsonObj;
        jsonObj.insert("cmd", "autoUpload:true");

        sendCmd(cmdConvert(jsonObj));
        sendData(jsonObj);
    }

    if(value.isString() && value.toString() == "autoUpload:false")
    {
        QJsonObject jsonObj;
        jsonObj.insert("cmd", "autoUpload:false");

        sendCmd(cmdConvert(jsonObj));
        sendData(jsonObj);
    }
}

void LocalSocketServer::sendData(const QJsonObject &jsonObj)
{
    if (m_localSocket == nullptr)
    {
        addLog("localSocket disconnect send cmd failed");
        return ;
    }

    QByteArray array =  QJsonDocument(jsonObj).toJson(QJsonDocument::Compact);
    array.prepend("@START");
    array.append("@END");
    m_localSocket->write(array);
    QString info =  "send cmd ";
    info += array;
    addLog(info);
}

void LocalSocketServer::addLog(const QString &info)
{
    qDebug () << info;
}

void LocalSocketServer::sendCmd(LocalSocketServer::ClientCmd cmd)
{
    emit signalCmdHandle(cmd);
}

LocalSocketServer::ClientCmd LocalSocketServer::cmdConvert(const QJsonObject &jsonObj)
{
    QString strCmd = jsonObj.value("cmd").toString();
    ClientCmd cmd = CLIENT_CMD_UNKNOWN;
    if ("queryHttpConnect" == strCmd)
    {
        cmd = CLIENT_CMD_TEST_HTTP_CONNECT;
    }
    else if("autoUpload:true" == strCmd)
    {
        cmd = CLIENT_CMD_AUTO_UPLOAD;
    }
    else if("autoUpload:false" == strCmd)
    {
        cmd = CLIENT_CMD_STOP_AUTO_UPLOAD;
    }
    return cmd;
}

void LocalSocketServer::newConnection()
{
    m_localSocket = m_localServer->nextPendingConnection();
    connect(m_localSocket, &QLocalSocket::readyRead, this, &LocalSocketServer::readData);
    connect(m_localSocket, &QLocalSocket::disconnected, m_localSocket, &QLocalSocket::deleteLater);
    // 处理连接
    addLog("New client connected.");
}

void LocalSocketServer::readData()
{
    QDataStream DS(m_localSocket);
    char data[MAX_LEN];

    qint64 iLen = m_localSocket->read(data, MAX_LEN);

    int index_start = 0,index_end = 0;
    QVector<QJsonObject> receiveData;
    QByteArray array(data, iLen);
    QByteArray tmp;

    m_receiveData.append(array);

    while (true)
    {
        index_start = m_receiveData.indexOf("@START");
        index_end = m_receiveData.indexOf("@END");

        if ( index_start >= 0 && index_end >= 0 )
        {
            if (index_end > index_start)
            {
                // 正常情况
                tmp = m_receiveData.mid(index_start + static_cast<int>(strlen("@START")), index_end - index_start
                                        - static_cast<int>(strlen("@START")));
                QJsonDocument jsondoc = QJsonDocument::fromJson(tmp);
                if (jsondoc.isObject())
                {
                    receiveData.append(jsondoc.object());
                }
            }
        }

        if (index_end >= 0)
        {
            // 移除“@END”和前面的内容
            QByteArray newarray = m_receiveData.right(m_receiveData.size() -
                                                      (index_end + static_cast<int>(strlen("@END"))));
            m_receiveData.swap(newarray);
        }

        if (m_receiveData.size() > (MAX_LEN << 2))
        {
            m_receiveData.clear(); // 防止垃圾越来越多
        }

        if (index_start < 0 ||  index_end < 0)
        {
            break;
        }
    }

    if (receiveData.isEmpty())
    {
        return ;
    }

    foreach (const QJsonObject&jsonobj, receiveData)
    {
        if(jsonobj.contains("cmd") )
        {
            handleCmd(jsonobj["cmd"]);
        }
    }
}

void LocalSocketServer::errorInformation(QLocalSocket::LocalSocketError state)
{
    qDebug() <<(QString("错误码:%1").arg(QString::number(state)));
}

void LocalSocketServer::handleReceiveCmd(LocalSocketServer::ClientCmd cmd, bool ok)
{
    QJsonObject jsonObj;

    switch (cmd)
    {
    case CLIENT_CMD_TEST_HTTP_CONNECT:
    {
        jsonObj.insert("cmd", "queryHttpConnect");
    }
        break;
    case CLIENT_CMD_AUTO_UPLOAD:
    {
        jsonObj.insert("cmd", "autoUpload:true");
    }
        break;
    case CLIENT_CMD_STOP_AUTO_UPLOAD:
    {
        jsonObj.insert("cmd", "autoUpload:false");
    }
        break;
    default:
        return;
    }

    QString result = ok ? "succeed" : "failed";
    jsonObj.insert("result", result);

    sendData(jsonObj);
}
```

### 客户端

.pri

```c++
HEADERS += \
    $$PWD/localsocketclient.h

SOURCES += \
    $$PWD/localsocketclient.cpp

```

.h

```c++
#ifndef LOCALSOCKETCLIENT_H
#define LOCALSOCKETCLIENT_H

#include <QObject>
#include <qsystemdetection.h>
#ifdef Q_OS_WIN
#include <qlocalsocket.h>
#endif
#include <qobject.h>
#include <qjsonvalue.h>
#include <QTimer>

#ifdef Q_OS_WIN
#define MAX_LEN 4096
#endif

class LocalSocketClient : public QObject
{
    Q_OBJECT
public:
    explicit LocalSocketClient(QObject *parent = nullptr);
    ~LocalSocketClient();

private:

    void connectServer();

    void sendData(const QJsonObject &value);

    //! 服务器发送指令
   void handleSocketSendCmd(const QJsonValue &value);
    //! 服务器回复指令
    void handleSocketReceiveCmd(const QJsonObject &obj);

    void heartbeat();

private slots:
    //!错误信息
    void errorInformation(QLocalSocket::LocalSocketError state);
    //!断开链接
    void disconnectFromServer();
    void connectSuccess();
    void readData();
signals:

private:
    //!判断连接服务器是否成功
    bool m_isConnectedServer;
    QLocalSocket *m_localSocket;
//    QTimer *m_heartbeat;
    QByteArray m_receiveData;
};

#endif // LOCALSOCKETCLIENT_H
```

.cpp

```c++
#include "localsocketclient.h"
#include <QJsonObject>
#include <QJsonDocument>
#include <QDataStream>

LocalSocketClient::LocalSocketClient(QObject *parent)
    : QObject(parent),
      m_isConnectedServer(false),
      m_localSocket(NULL)/*,
      m_heartbeat(NULL)*/
{    
    if (m_localSocket == NULL)
    {
        m_localSocket = new QLocalSocket(this);
//        connect(m_socket,SIGNAL(error(QLocalSocket::LocalSocketError)),this,SLOT(errorInformation(QLocalSocket::LocalSocketError)));
        connect(m_localSocket,SIGNAL(connected()),this,SLOT(connectSuccess()));
        connect(m_localSocket,SIGNAL(disconnected()),this,SLOT(disconnectFromServer()));
        connect(m_localSocket,SIGNAL(readyRead()),this,SLOT(readData()));
    }

//    if (m_heartbeat == NULL)
//    {
//        m_heartbeat = new QTimer(this);
//        m_heartbeat->setInterval(1000);
//        connect(m_heartbeat, &QTimer::timeout, this, &LocalSocketClient::heartbeat);
//    }
}

LocalSocketClient::~LocalSocketClient()
{
    if (m_localSocket != NULL)
    {
        m_localSocket->disconnectFromServer();
        m_localSocket->deleteLater();
        m_localSocket = NULL;
    }

//    if (m_heartbeat != NULL)
//    {
//        m_heartbeat->stop();
//        m_heartbeat->deleteLater();
//        m_heartbeat = NULL;
//    }
}

void LocalSocketClient::connectServer()
{
    if (m_localSocket == NULL)
    {
        m_localSocket = new QLocalSocket(this);
//        connect(m_socket,SIGNAL(error(QLocalSocket::LocalSocketError)),this,SLOT(errorInformation(QLocalSocket::LocalSocketError)));
        connect(m_localSocket,SIGNAL(connected()),this,SLOT(connectSuccess()));
        connect(m_localSocket,SIGNAL(disconnected()),this,SLOT(disconnectFromServer()));
        connect(m_localSocket,SIGNAL(readyRead()),this,SLOT(readData()));
    }

    if (!m_isConnectedServer)
    {
        // 设置服务器地址，类似tcp通讯中的服务器IP及端口
        m_localSocket->connectToServer("Pulmolab");
    }
}

void LocalSocketClient::sendData(const QJsonObject &jsonObj)
{
    if (!m_isConnectedServer)
    {
        connectServer();
    }

    int reconnectCount = 3;

    while(!m_isConnectedServer && reconnectCount > 0)
    {
        connectServer();
        --reconnectCount;
    }

    if (!m_isConnectedServer)
    {
        return;
    }

    QByteArray array =  QJsonDocument(jsonObj).toJson(QJsonDocument::Compact);
    array.prepend("@START");
    array.append("@END");
    m_localSocket->write(array);
    qDebug () << "send cmd " << array;
}

void LocalSocketClient::handleSocketSendCmd(const QJsonValue &value)
{
   if(value.isString() && value.toString() == "queryHttpConnectSucceed")
   {
       qDebug() << "handle cmd queryHttpConnectSucceed";
   }
}

void LocalSocketClient::handleSocketReceiveCmd(const QJsonObject &obj)
{
    if(obj.value("cmd").isString() && obj.value("result").isString())
    {
        QString cmd = obj.value("cmd").toString();
        QString result = obj.value("result").toString();

        qDebug () << Q_FUNC_INFO << "cmd: " << cmd << ", result:" << result;
    }
}

void LocalSocketClient::heartbeat()
{
    QJsonObject jsonObj;
    jsonObj.insert("cmd", "heartbeat");
    sendData(jsonObj);
}

void LocalSocketClient::errorInformation(QLocalSocket::LocalSocketError state)
{
    Q_UNUSED(state)
    qDebug() <<(QString("错误码:%1").arg(QString::number(state)));
}

void LocalSocketClient::disconnectFromServer()
{
    m_isConnectedServer = false;
}

void LocalSocketClient::connectSuccess()
{
    m_isConnectedServer = true;
}

void LocalSocketClient::readData()
{
    QDataStream DS(m_localSocket);
    char data[MAX_LEN];

    qint64 iLen = m_localSocket->read(data, MAX_LEN);

    int index_start = -1, index_end = -1;
    QVector<QJsonObject> receiveData;
    QByteArray array(data, iLen);
    QByteArray tmp;

    m_receiveData.append(array);

    while (true)
    {
        index_start = m_receiveData.indexOf("@START");
        index_end = m_receiveData.indexOf("@END");

        if ( index_start>=0 && index_end>= 0 )
        {
            if (index_end > index_start)
            {
                // 正常情况
                tmp = m_receiveData.mid(index_start + static_cast<int>(strlen("@START")),
                                        index_end-index_start - static_cast<int>(strlen("@START")));
                QJsonDocument jsondoc = QJsonDocument::fromJson(tmp);
                if (jsondoc.isObject())
                {
                    receiveData.append(jsondoc.object());
                }
            }
        }

        if (index_end>= 0)
        {
            // 移除“@END”和前面的内容，继续解析下一条指令
            QByteArray newarray = m_receiveData.right(m_receiveData.size()
                                                      - (index_end + static_cast<int>(strlen("@END"))));
            m_receiveData.swap(newarray);
        }

        if (m_receiveData.size() > (MAX_LEN << 2))
        {
            m_receiveData.clear(); // 防止垃圾越来越多
        }

        if (index_start<0 ||  index_end< 0)
        {
            break;
        }
    }

    if (receiveData.isEmpty())
    {
        return ;
    }

    foreach(const QJsonObject&jsonobj, receiveData)
    {
        if(jsonobj.contains("cmd") && jsonobj.contains("result"))
        {
            handleSocketReceiveCmd(jsonobj);
        }
        else if(jsonobj.contains("cmd") && !(jsonobj.contains("result")))
        {
            handleSocketSendCmd(jsonobj);
        }
    }
}
```

