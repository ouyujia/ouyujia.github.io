---
title: QTHttp通讯
abbrlink: 25247
date: 2023-11-25 19:52:29
updated:
tags:
  - QT
  - HTTP
categories:
  - - QT
    - net
keywords:
description: QT http通讯接口
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

> 记录自己使用http通讯经历。

### QT使用http通讯开发环境

>需要添加模块：network

### Http通讯处理思路及注意事项

1. QT已经封装了http通讯的接口，网络上demo很多，这里不再赘述，需要注意的是get()/post()返回的指针需要手动销毁释放资源，否则会内存泄漏。

   ```c++
   if(m_reply != nullptr)
   {
       m_reply->deleteLater();
       m_reply = nullptr;
   }
   m_reply = m_manager.post(request, m_httpItem.getHttpRequestPara());
   //m_reply = m_manager.get(request);
   ```

2. 对于数据的处理不能只依赖与QNetworkReply::finished，需要结合QNetworkReply::readyRead，否则可能读取的数据不全。

   ```c++
   void HttpClient::replyFinished()
   {
       if(m_reply == NULL)
       {
           qDebug() << Q_FUNC_INFO << "m_reply is a null pinter";
           return;
       }  
   
       if(m_reply->error() == QNetworkReply::NoError)
       {
           emit signalSetHttpItemResendCount(0);
           QByteArray data = m_replyData;
           qDebug() << Q_FUNC_INFO << "full response:" << data;
   
           m_httpItem.processReplyFunction(data, m_httpItem.getHttpReplyArgument());
       }
       else
       {        
           emit signalSetHttpItemResendCount(m_httpItem.getHttpResendCount() - 1);
   		
           // 请求存在错误信息，输出错误信息
           qDebug() << Q_FUNC_INFO << " Error:" << m_reply->errorString() << " Error code:" << m_reply->error();
           qDebug() << Q_FUNC_INFO << "reply header" << m_reply->header(QNetworkRequest::KnownHeaders::ContentTypeHeader).toString();
       }
   	
       m_replyData.clear();
       // 请求完成后释放资源
       m_reply->deleteLater();
       m_reply = NULL;
   }
   
   void HttpClient::replyReadyRead()
   {
       if(m_reply == NULL)
       {
           qDebug() << Q_FUNC_INFO << ("m_reply is a null pinter");
           return;
       }
   
       QByteArray data = m_reply->readAll();
   //    qDebug() << Q_FUNC_INFO << "response:" << data;
       m_replyData.append(data);
   }
   ```

3. 对于http通讯需要用到的URL、参数列表、请求方式、协议头、超时时间、超时重复次数等信息可以定义一个HttpItem类集中管理。

4. 一般Http都不会只发一条，在有多条Http请求需要顺序发送时可以添加一个HttpClientManager管理类，里面通过一个队列、栈或其他顺序容器维护Http请求。通过一个简单的状态机来处理请求队列。

### Http参数处理

> 在实际运用中，开发者可能与不同的平台对接，不同平台的参数名大多不同，或者对接方与开发者自己的数据名称不同（如：开发者数据名称：name，对接方数据名称：XM）此时需要进行数据名称映射转换。如果每一个对接平台都需要再代码里映射将导致代码冗余，增加维护难度，且在更改对接平台后需要重新编译发布版本，给开发者添加太多工作量，如果频繁更改对接平台，需要重复工作的开发者应该会暴走吧（笑）。笔者工作就需要进行多平台的对接，在团队前辈的提醒下使用一个简单方法处理此问题，这里分享给米娜桑。

#### 参数处理思路

通过配置文件来映射参数，下面提供一个简单实例给米娜桑参考。

httpRequestConfig.xml

```xml
<?xml version='1.0' encoding='GB2312'?>
<config>
	<version>1.0</version>
	<http_config>
		<!-- ip: 请求ip -->
		<ip>192.168.44.24</ip>
		<!-- port: 请求端口 -->
		<port>8080</port>
	</http_config>
	<protocols>		
		<!-- protocol name ：请求协议名-->
		<protocol name="patient_query"> 
			<request_info>
				<!-- request_path ：请求路径 -->
				<request_path>PatientService/getPatientInfoJson</request_path>
				<!-- request_type：请求类型 -->
				<request_type>get</request_type>
				<!-- request_header ： 请求头协议 -->
				<request_header>application/json</request_header>
			</request_info>
			
			<!-- items：请求参数 -->
			<items>
				<!-- items：具体请求参数项 -->
				<item>  
					<!-- key： 协议中的请求参数名 -->
					<key>testNum</key>
					<!-- db_key：大肺数据库请求参数名 -->
					<db_key>patient_id</db_key>
					<!-- db_table：大肺数据库表名 -->
					<db_table>patient</db_table>
					<!--  type ： 请求参数数据类型-->
					<type>string</type>
					<!--  db_key_type ： 请求数据库参数类型-->
					<db_key_type>string</db_key_type>
					<!-- encryption：是否加密 true：是 -->
					<encryption>false</encryption>
					<!-- remark ：特殊类型映射值备注 项与项之间用,(英文逗号)隔开-->
					<remark>M:1,F:0</remark>
				</item>						
			</items>		
		</protocol>						
	</protocols>
</config>
		
```

httpReplyConfig.xml

```xml
<?xml version='1.0' encoding='GB2312'?>
<config>
	<version>1.0</version>
	<protocols>		
		<!--  protocol name： 协议名-->
		<protocol name="patient_query">  
			<!--  data : 回复数据 -->
			<data>  
					<!-- key: 协议使用的回复数据参数名 -->
					<key>retDdata</key>
					<!-- db_key: 大肺软件中使用的回复数据参数名 -->
					<db_key>data</db_key>
					<!-- type :回复数据类型 根据通讯双方自定义类型设置如：json，string，array-->
					<type>array</type>
			</data>	

			<!--  msg : 回复消息-->
			<msg>  
					<key>msg</key>
					<db_key>msg</db_key>
					<type>string</type>
			</msg>	

			<!-- retCode : 回复code -->
			<retCode>  
					<key>retCode</key>
					<db_key>retCode</db_key>
					<type>int</type>
			</retCode>	

			<!-- success : 请求是否成功 -->
			<success>  
					<key>success</key>
					<db_key>success</db_key>
					<type>string</type>
			</success>

			<!-- items : 回复数据内容 -->
			<items>
				<!-- item: 协议具体字段 -->
				<item>  
					<!-- key : 协议参数名 -->
					<key>testNum</key>
					<!-- db_key： 大肺数据库字段名 -->
					<db_key>patient_id</db_key>
					<!-- db_table: 参数对于数据库表名 -->
					<db_table>patient</db_table>
					<!-- type ： 参数类型 -->
					<type>string</type>
					<!--  db_key_type ： 请求数据库参数类型-->
					<db_key_type>string</db_key_type>
					<!-- encryption：是否加密 true：是 -->
					<encryption>false</encryption>
					<!-- remark ：特殊类型映射值备注 项与项之间用,(英文逗号)隔开，内容不能留空格-->
					<remark>M:1,F:0</remark>
				</item>				
			</items>		
		</protocol>									
	</protocols>
</config>
		
```

