---
title: XML解析
abbrlink: 1441
date: 2023-05-04 10:00:41
updated:
tags:
  - QT
  - XML
categories:
  - - QT
    - QDomDocument
keywords:
description: 其余QT的XML解析接口	
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

> 在项目开发中需要使用xml文件作为配置文件，阅读公司前辈写的xml解析接口后，梳理使用方法及实现。 

## xml

[XML](https://developer.mozilla.org/zh-CN/docs/Web/XML/XML_Introduction)（Extensible Markup Language）是一种类似于 HTML，但是没有使用预定义标记的语言。因此，可以根据自己的设计需求定义专属的标记。这是一种强大将数据存储在一个可以存储、搜索和共享的格式中的方法。最重要的是，因为 XML 的基本格式是标准化的，如果你在本地或互联网上跨系统或平台共享或传输 XML，由于标准化的 XML 语法，接收者仍然可以解析数据。

有许多基于 XML 的语言，包括 XHTML、MathML、SVG、RSS 和 RDF (en-US)。

## 使用

### QT使用

> Qt中使用xml需要添加模块：xml
>
> ```c++
> QT       += core gui xml
> ```

------

> 注：此接口使用的是QDomDocument实现读大型文件将会造成卡顿且使用大量内存，不关系内存使用情况，大型文件可在独立线程处理避免卡顿。

.h

```c++
#ifndef XMLPARSER_H
#define XMLPARSER_H

#include <QDomDocument>
#include <QFile>
#include <QMutex>
#include <QMap>
#include <QList>
#include <QString>

/**
 * @brief The XmlParser class
 * 1.支持从配置文件中读取配置，保存在内存中；
 * 2.支持保存xml配置到本地内存中；
 * 3.支持对配置结点数据的操作：增、删、改
 */
class XmlParser
{
public:
    /**
     * @brief XmlParser  xml文件解析者，支持对xml文件的读写、修改操作
     */
    XmlParser();
    virtual ~XmlParser();

    /**
     * @brief open  打开XML文件。
     * @param fileName fileName文件名。
     * @return  true，成功；false，打开失败或XML文件解析失败。
     */
    bool open(const QString &fileName);

    /**
     * @brief ifDnoExistCreate 如果不存在，则创建
     * @param fileName
     */
    void ifDnoExistCreate(const QString &fileName);
    /**
     * @brief saveToFile 保存当前文件到本地内存
     * @return  true，保存成功；false，保存失败
     */
    bool saveToFile();

    /**
     * @brief saveToFile  保存指定文件到本地内存
     * @param fileName  待保存的文件名
     * @return  true，保存成功；false，保存失败
     */
    bool saveToFile(const QString &fileName);
    //! 关闭打开的文件
    void closeFile();
    /**
     * @brief getNode  获取指定路径下的结点数据
     * @param indexStr 结点元素所在的路径，格式：string1|string2|string3；
     * @param tag 返回获取结点数据
     * @return   true，成功，false，结点为空数据或结点不存在
     * @attention \b 不推荐使用，在存在并列节点时会有问题
     */
    bool getNode(const QString &indexStr, QDomElement &tag);

    /**
     * @brief setNode  设置指定路径下的结点数据
     * @param indexStr  结点元素所在的路径，格式：string1|string2|string3；
     * @param srcTag 设置的结点数据
     * @return   true，成功，false，结点为空数据或父结点不存在
     * @attention \b 在存在并列节点时会有问题
     */
    bool setNode(const QString &indexStr, QDomElement &srcTag);
    /**
     * @brief addNode  添加一个新结点
     * @param indexStr 结点路径，格式：string1|string2|string3；
     * @param tagName  结点名字
     * @param value  结点值
     * @param attrs 结点属性
     * @return   true，成功；false，父结点不存在或xml文件损坏。
     * @attention \b 在存在并列节点时会有问题
     */
    bool addNode(const QString &indexStr, const QString &tagName, const QString &value = QString(),
                 const QMap<QString, QString> &attrs = QMap<QString, QString>());
    /**
     * @brief removeNode 删除一个结点，默认删除结点路径下的首个结点，如匹配结点属性时，则会删除匹配属性的结点
     * @param indexStr 结点路径，格式：string1|string2|string3；
     * @param attrs 结点属性，非空时会判断结点属性
     * @return  true，成功；false，结点或父结点不存在。
     * @note 支持存在多个相同结点名称时，删除其中匹配结点属性的一个结点
     * @attention \b 在存在并列节点时会有问题
     */
    bool removeNode(const QString &indexStr, const QMap<QString, QString> &attrs = QMap<QString, QString>());

    /**
     * @brief setValue 设置结点值，默认设置结点路径下首个结点的值，如匹配结点属性时，则会设置匹配属性结点的值
     * @param indexStr 结点路径，格式：string1|string2|string3；
     * @param value 带设置的结点值
     * @param attar 结点属性，非空时会判断结点属性
     * @return true，成功；false，结点或父结点不存在。
     * @note 结点没有值时，会新增一个结点
     * @attention \b 在存在并列节点时会有问题
     */
    bool setValue(const QString &indexStr, const QString &value,
                  const QMap<QString,QString>&attrs=QMap<QString,QString>());

    /**
     * @brief getValue 获取结点值，默认获取结点路径下首个结点的值，如匹配结点属性时，则会获取匹配属性结点的值
     * @param indexStr  结点路径，格式：string1|string2|string3；
     * @param value  返回读取结点的值
     * @param attar 结点属性，非空时会判断结点属性
     * @return  true，成功；false，结点不存在。
     * @attention \b 在存在并列节点时会有问题
     */
    bool getValue(const QString &indexStr, QString&value,
                  const QMap<QString,QString>&attrs=QMap<QString,QString>());

    /**
     * @brief getValue 获取结点路径下，所有并列结点的值
     * @param indexStr 结点路径，格式：string1|string2|string3；
     * @param valueList 返回所有并列结点的值
     * @return  成功；false，结点不存在。
     * @attention \b 在存在并列节点时会有问题
     */
    bool getValue(const QString &indexStr, QStringList &valueList);

    /**
     * @brief setAttr  设置结点路径下首个结点的属性
     * @param indexStr 结点元素所在的路径，格式：string1|string2|string3；
     * @param attr  结点属性名称；
     * @param value  要设定属性的值。
     * @return   true，成功；false，结点不存在。
     * @attention \b 在存在并列节点时会有问题
     */
    bool setAttr(const QString &indexStr, const QString &attr, const QString &value);
    /**
     * @brief getAttr  获取结点路径下首个结点的属性
     * @param indexStr 结点元素所在的路径，格式：string1|string2|string3；
     * @param attr  结点属性名称；
     * @param value 要获取的值。
     * @return  true，成功；false，结点元素不存在。
     * @attention \b 在存在并列节点时会有问题
     */
    bool getAttr(const QString &indexStr, const QString &attr, QString &value);

    /**
     * @brief getAttrs  获取结点路径下所有属性值
     * @param indexStr 结点元素所在的路径，格式：string1|string2|string3；
     * @param attr  结点属性名称；
     * @param value 要获取的值。
     * @return  true，成功；false，结点元素不存在。
     * @attention \b 在存在并列节点时会有问题
     */
    bool getAttrs(const QString &indexStr, const QString &attr, QStringList &valueList);



    /**
      @example 下面为读写XML新的接口
            //!获取DomElement
            QDomElement dee = db_config_xml.getDomElement("item");
            //!根据指定的DomElement 获取子的DomElement
            dee = db_config_xml.getDomElement(dee,"type");
            qDebug()<<dee.tagName()<<dee.attribute("name")<<dee.text();

            //!获取指定的节点
            QMap<QString,QString> map;
            map.insert("name","sex");
            QDomElement dee_special = db_config_xml.getDomElement("item", map );
            for(int i=0;i<dee_special.childNodes().size();i++)
            {
                qDebug()<<dee_special.childNodes().at(i).toElement().text();
            }

            //!从指定节点中节获取name_show节点
            QDomElement name_show = db_config_xml.getDomElement(dee_special,"name_show");
            for(int i=0;i<name_show.childNodes().size();i++)
            {
                qDebug()<<name_show.childNodes().at(i).toElement().text()<<name_show.childNodes().at(i).toElement().attribute("language");
            }

            //!获取所有的DomElement
            QList<QDomElement> dlist  = db_config_xml.getDomElementList("item");
            qDebug()<<dlist.size();
            dee.setAttribute("type_test","liuyinggui");

            //!创建一个全新的QDomElement
            QDomElement addele = db_config_xml.createDomElement("mytest");

            //!设置属性
            addele.setAttribute("add_mytest","2022");

            //!设置值
            db_config_xml.setValue(addele,"0123456789");

            QDomNode pnode  = dee.parentNode();
            QDomNode p_pnode  = pnode.parentNode();
            qDebug()<<p_pnode.isNull();

            //!设置到节点
            db_config_xml.setDomElement(addele,pnode);

            //db_config_xml.setDomElement(addele,p_pnode);
            //db_config_xml.setDomElement(dee);
            db_config_xml.saveToFile();
      */

    //!设置domele 下的值,需要确保domele下再无子节点，或者指定attrs后无多个节点，否则设置会出错
    bool setValue(QDomElement&domele, const QString&value,
                  const QMap<QString, QString> &attrs= QMap<QString, QString>());
    /**
     * @brief getDomElement 从parent 根据tagname 来获取QDomElement
     * @param parent
     * @param tagname 指定属性
     * @param attr
     * @return
     * @attention 注意如果有多个节点，默认只查找第一个。
     */
    QDomElement getDomElement(const QDomElement &parent, const QString &tagname, const QMap<QString,QString>&attr=QMap<QString,QString>());

    /**
     * @brief getDomElement 从整个XML文件根据tagname来获取QDomElement
     * @param tagname
     * @param attr 指定属性
     * @return
     * @attention 注意如果有多个节点，默认只查找第一个。
     */
    QDomElement getDomElement(const QString &tagname, const QMap<QString,QString>&attr=QMap<QString,QString>());

    //!先查看parent是否有对应的元素，如果没有，则创建一个
    QDomElement getDomElementNoNull(QDomElement &parent, const QString &tagname, const QMap<QString,QString>&attr=QMap<QString,QString>());

    /**
     * @brief getDomElementList 获取指定 tagname 和attr QDomElement
     * @param tagname
     * @param attr
     * @return
     */
    QList<QDomElement> getDomElementList(const QString &tagname, const QMap<QString,QString>&attr=QMap<QString,QString>());

    //!在parent 范围内查找符合要求的QDomElement
    QList<QDomElement> getDomElementList(const QDomElement &parent,const QString &tagname, const QMap<QString,QString>&attr=QMap<QString,QString>());
    /**
     * @brief createDomElement
     * 用于创建一个空的QDomElement ，一般配合\ref setDomElement使用
     * @param tagName
     * @return
     */
    QDomElement createDomElement(const QString&tagName);
    /**
     * @brief setDomElement 更新(替换)，增加(新) QDomElement ，更新的原则:tagname是否相同，属性是否相同
     * @param new_ele 新的元素
     * @param parent 父节点
     * @param attr 属性，如果不用关注属性，默认
     * @param append 是否在节点后面追加
     * @return
     * @attention  此函数不适用于有相同tagname的并列节点，如果有此类节点，不要使用此函数
     * 简单的方法是parent.appendchild(new_ele); 直接在父节点后增加
     */
    bool setDomElement(QDomElement&new_ele,QDomNode& parent,
                       const QMap<QString,QString>&attr=QMap<QString,QString>());
private:
    /**
     * @brief copyNode  拷贝源结点数据到目标结点
     * @param dstTag 目标结点数据
     * @param srcTag 源结点数据
     * @return  true，拷贝成功；false，拷贝失败
     */
    bool copyNode(QDomElement &dstTag, QDomElement &srcTag);

    QDomElement findFirstElement(const QString &indexStr);
    /**
     * @brief hasNode  检查结点是否存在
     * @param indexStr 结点元素所在的路径，格式：string1|string2|string3；
     * @return  true，成功；false，结点元素不存在。
     */
    bool hasNode(const QString &indexStr);
private:
    QDomDocument m_xml;
    QMutex m_locker;
    QString m_fileName;
};

#endif // XMLPARSER_H

```

.cpp

```c++
#include "xmlParser.h"
#include "securityapi.h"
#include <QMutexLocker>
#include <QDebug>
#include <QTextStream>
#include <QStringRef>

//#define START_AES_DECRYPT 1 //开启加密

XmlParser::XmlParser()
    : m_fileName("")
{

}

XmlParser::~XmlParser()
{
    closeFile();
}

bool XmlParser::open(const QString &fileName)
{
    if (fileName.isEmpty())
    {
        qDebug() << Q_FUNC_INFO << "file name:" << fileName << "is empty!";
        return false;
    }
#ifdef QT_DEBUG
//    qDebug() << "--------------------Xml parser open file:" << fileName
//             << "--------------------";
#endif
    QMutexLocker locker(&m_locker);
    // 将filename包装成一个File格式的文件
    QFile file(fileName);
    // 以文本方式打开只读
    if (!file.open(QFile::ReadOnly | QFile::Text))
    {
        qDebug() << "Open config file failed! err: " + file.errorString()<< fileName;
        return false;
    }

    // 解密、以及MD5校验处理；（注意需要先进行校验） TODO：


    // 初始化_xml对象
    m_xml.clear();
    QString error;
    // 判断是否读取XML文件成功
#ifdef START_AES_DECRYPT
    if (!m_xml.setContent(SecurityAPI::aesDecrypt(file.readAll()), false, &error))
    {
        qDebug() << Q_FUNC_INFO << "Parse config file failed! err: " + error;
        file.close();
        return false;
    }
#else
    int errorLine = -1;
    int errorColumn = -1;
    if (!m_xml.setContent(&file, false, &error, &errorLine, &errorColumn))
    {
        qDebug() << Q_FUNC_INFO << "Parse config file failed! err: " + error
                 << " error line: " << errorLine << " error column: " << errorColumn
                 << " file name: "<< fileName;
        file.close();
        return false;
    }
#endif

    // 更新XML文件路径
    m_fileName = fileName;
    return true;
}

void XmlParser::ifDnoExistCreate(const QString &fileName)
{
    if(QFile::exists(fileName)) return ;
    QFile file(fileName);
    if(!file.open(QFile::WriteOnly | QFile::Text))
        return;

    QDomDocument doc;
    QDomProcessingInstruction instruction;
    instruction=doc.createProcessingInstruction("xml","version=\"1.0\" encoding=\"UTF-8\"");
    doc.appendChild(instruction);

    QDomElement root=doc.createElement("config");
    doc.appendChild(root);

    QDomElement version = doc.createElement("version");
    QDomText text;
    text=doc.createTextNode("1.0");
    version.appendChild(text);

    root.appendChild(version);

    QTextStream out_stream(&file);
    doc.save(out_stream,4,QDomNode::EncodingFromDocument);
    file.close();
}

bool XmlParser::saveToFile()
{
    return saveToFile(m_fileName);
}

bool XmlParser::saveToFile(const QString &fileName)
{
    if (m_xml.isNull() || fileName.isEmpty())
        return false;

    QMutexLocker locker(&m_locker);

    QFile file(fileName);
#ifdef START_AES_DECRYPT            // 加密功能
    if (!file.open(QFile::ReadWrite | QFile::Truncate))
    {
        qDebug() << Q_FUNC_INFO << "save to"
                 << fileName <<"failed:" << file.errorString();
        return false;
    }
    QByteArray encryptContent = SecurityAPI::aesEncrypt(m_xml.toByteArray());
    file.resize(0);     //清空原有文件内容
    file.write(encryptContent, encryptContent.size());
#else
    if (!file.open(QFile::WriteOnly | QFile::Text))
    {
        qDebug() << Q_FUNC_INFO << "save to"
                 << fileName <<"failed:" << file.errorString();
        return false;
    }

    // 加密文件、以及增加MD5校验 TODO：

    QTextStream write(&file);
    m_xml.save(write, 4, QDomNode::EncodingFromDocument);
#endif
    file.close();
    return true;
}

void XmlParser::closeFile()
{
    QMutexLocker locker(&m_locker);

    QFile file(m_fileName);
    file.close();
}

bool XmlParser::hasNode(const QString &indexStr)
{
    return !findFirstElement(indexStr).isNull();
}

bool XmlParser::getNode(const QString &indexStr, QDomElement &tag)
{
    if (!hasNode(indexStr))
        return false;

    tag = findFirstElement(indexStr);
    return true;
}


bool XmlParser::setNode(const QString &indexStr, QDomElement &srcTag)
{
    QMutexLocker locker(&m_locker);

    QDomElement dstTag = findFirstElement(indexStr);
    return copyNode(dstTag, srcTag);
}


bool XmlParser::addNode(const QString &indexStr, const QString &tagName,
                              const QString &value, const QMap<QString, QString> &attrs)
{
    QMutexLocker locker(&m_locker);

    QDomElement tag = findFirstElement(indexStr);
    if (tag.isNull())
        return false;

    QDomElement newTag = m_xml.createElement(tagName);
    // 设置结点属性
    QMap<QString, QString>::ConstIterator iters;
    for (iters = attrs.constBegin(); iters != attrs.constEnd(); ++iters)
    {
        newTag.setAttribute(iters.key(), iters.value());
    }

    // 父结点增加子结点
    if (tag.appendChild(newTag).isNull())
        return false;

    // 新增的结点创建值结点，并设置值数据。
    return !newTag.appendChild(m_xml.createTextNode(value)).isNull();
}

bool XmlParser::removeNode(const QString &indexStr, const QMap<QString, QString> &attrs)
{
    QMutexLocker locker(&m_locker);
    QDomElement tag = findFirstElement(indexStr);
    if (!tag.isNull() && !tag.parentNode().isNull())
    {
        if (attrs.isEmpty())
        {
            QDomNode node = tag.parentNode().removeChild(tag);
            return !node.isNull();
        }

        int tag_count = tag.parentNode().childNodes().count();
        for (int i = 0; i < tag_count; ++i)
        {
            QDomElement child_tag = tag.parentNode().childNodes().at(i).toElement();
            bool same_attr = true;
            QMap<QString, QString>::ConstIterator iters;
            for (iters = attrs.constBegin(); iters != attrs.constEnd(); ++iters)
            {
                same_attr &= (child_tag.attribute(iters.key()) == iters.value());
            }
            if (same_attr)
            {
                QDomNode node = tag.parentNode().removeChild(child_tag);
                return !node.isNull();
            }
        }
    }
    return false;
}

bool XmlParser::setValue(const QString &indexStr, const QString &value, const QMap<QString, QString> &attrs)
{
    QMutexLocker locker(&m_locker);

    QDomElement tag = findFirstElement(indexStr);
    if (tag.isNull())
        return false;

    // 结点属性为空，设置首个结点的值
    if (attrs.isEmpty())
    {
        QDomText domText = tag.firstChild().toText();
        if (!domText.isNull())
        {
            domText.setData(value);
        }
        else
        {
            // 节点没有内容时，新增一个结点
            tag.appendChild(m_xml.createTextNode(value));
        }
        return true;
    }

    // 结点属性不为空，设置匹配结点属性的值
    int tag_count = tag.parentNode().childNodes().count();
    for (int i = 0; i < tag_count; ++i)
    {
        QDomElement child_tag = tag.parentNode().childNodes().at(i).toElement();
        bool same_attr = true;
        QMap<QString, QString>::ConstIterator iters;
        // 判断结点属性是否一致
        for (iters = attrs.constBegin(); iters != attrs.constEnd(); ++iters)
        {
            same_attr &= (child_tag.attribute(iters.key()) == iters.value());
        }
        if (same_attr)
        {
            QDomText domText = child_tag.firstChild().toText();
            if (!domText.isNull())
            {
                domText.setData(value);
            }
            else
            {
                // 节点没有内容时，新增一个结点
                child_tag.appendChild(m_xml.createTextNode(value));
            }
            return true;
        }
    }
    // 未找到匹配属性的结点，返回false
    return false;
}

bool XmlParser::setValue(QDomElement&domele,const QString&value,const QMap<QString, QString> &attrs)
{
    QDomElement& tag = domele;

    // 结点属性为空，设置首个结点的值
    if (attrs.isEmpty())
    {
        QDomText domText = tag.firstChild().toText();

        if (!domText.isNull())
        {
            domText.setData(value);
        }
        else
        {
            // 节点没有内容时，新增一个结点
            tag.appendChild(m_xml.createTextNode(value));
        }
        return true;
    }

    // 结点属性不为空，设置匹配结点属性的值
    int tag_count = tag.parentNode().childNodes().count();
    for (int i = 0; i < tag_count; ++i)
    {
        QDomElement child_tag = tag.parentNode().childNodes().at(i).toElement();
        bool same_attr = true;
        QMap<QString, QString>::ConstIterator iters;
        // 判断结点属性是否一致
        for (iters = attrs.constBegin(); iters != attrs.constEnd(); ++iters)
        {
            same_attr &= (child_tag.attribute(iters.key()) == iters.value());
        }
        if (same_attr)
        {
            QDomText domText = child_tag.firstChild().toText();
            if (!domText.isNull())
            {
                domText.setData(value);
            }
            else
            {
                // 节点没有内容时，新增一个结点
                child_tag.appendChild(m_xml.createTextNode(value));
            }
            return true;
        }
    }
    // 未找到匹配属性的结点，返回false
    return false;

}

bool XmlParser::getValue(const QString &indexStr, QString &value, const QMap<QString, QString> &attrs)
{
    QMutexLocker locker(&m_locker);
    QDomElement tag = findFirstElement(indexStr);
    if (tag.isNull())
        return false;

    // 结点属性为空，获取首个结点的值
    if (attrs.isEmpty())
    {
        value = tag.text();
        return true;
    }

    // 结点属性不为空，获取匹配结点属性的值
    int tag_count = tag.parentNode().childNodes().count();
    for (int i = 0; i < tag_count; ++i)
    {
        QDomElement child_tag = tag.parentNode().childNodes().at(i).toElement();
        bool same_attr = true;
        QMap<QString, QString>::ConstIterator iters;
        // 判断结点属性是否一致
        for (iters = attrs.constBegin(); iters != attrs.constEnd(); ++iters)
        {
            same_attr &= (child_tag.attribute(iters.key()) == iters.value());
        }
        if (same_attr)
        {
            value = child_tag.text();
            return true;
        }
    }
    // 未找到匹配属性的结点，返回false
    return false;

}

bool XmlParser::getValue(const QString &indexStr, QStringList &valueList)
{
    QMutexLocker locker(&m_locker);
    QDomElement tag = findFirstElement(indexStr);
    if (tag.isNull() || tag.parentNode().isNull())
        return false;

    for (int i = 0; i < tag.parentNode().childNodes().count(); ++i)
    {
        valueList.append(tag.parentNode().childNodes().at(i).toElement().text());
    }

    return true;
}

bool XmlParser::setAttr(const QString &indexStr, const QString &attr, const QString &value)
{
    QMutexLocker locker(&m_locker);
    QDomElement tag = findFirstElement(indexStr);
    if (!tag.isNull())
    {
        if (tag.hasAttribute(attr))
        {
            tag.setAttribute(attr, value);
            return true;
        }
    }
    return false;
}

bool XmlParser::getAttr(const QString &indexStr, const QString &attr, QString &value)
{
    QMutexLocker locker(&m_locker);
    QDomElement tag = findFirstElement(indexStr);
    if (!tag.isNull())
    {
        if (tag.hasAttribute(attr))
        {
            value = tag.attribute(attr);
            return true;
        }
    }
    value = "";
    return false;
}

bool XmlParser::getAttrs(const QString &indexStr, const QString &attr, QStringList &valueList)
{
    QMutexLocker locker(&m_locker);
    QDomElement tag = findFirstElement(indexStr);
    if (tag.isNull() || tag.parentNode().isNull())
        return false;
    for (int i = 0; i < tag.parentNode().childNodes().count(); ++i)
    {
        valueList.append(tag.parentNode().childNodes().at(i).toElement().attribute(attr));
    }
    return true;
}

QDomElement XmlParser::getDomElement(const QDomElement& parent, const QString &tagname, const QMap<QString, QString> &attr)
{
    QDomElement de;
    if (parent.isNull())  return de;

    QDomNodeList list = parent.elementsByTagName(tagname);
    QStringList keys = attr.keys();
    QString attr_str;
    for (int i = 0; i < list.size(); ++i)
    {
        de = list.at(i).toElement();
        if (de.isNull())
        {
            continue;
        }
        int count =0;
        foreach(const QString&key,keys)
        {
            attr_str = de.attribute(key);
            if( attr[key] == attr_str )
            {
                count++;
            }
        }
        if( keys.size() == count )
        {
            //返回第一次找到的那个
            return de;
        }
    }
    return QDomElement();
}

QDomElement XmlParser::getDomElement(const QString &tagname, const QMap<QString, QString> &attr)
{
    QDomElement de;
    if(m_xml.isNull())  return de;

    QDomNodeList list = m_xml.elementsByTagName(tagname);
    QStringList keys = attr.keys();
    QString attr_str;
    for (int i = 0; i < list.size(); ++i)
    {
        de = list.at(i).toElement();
        if (de.isNull())
        {
            continue;
        }
        int count =0;
        foreach(const QString&key,keys)
        {
            attr_str = de.attribute(key);
            if( attr[key] == attr_str )
            {
                count++;
            }
        }
        if( keys.size() == count )
        {
            //返回第一次找到的那个
            return de;
        }
    }
    return QDomElement();
}

QDomElement XmlParser::getDomElementNoNull(QDomElement &parent, const QString &tagname, const QMap<QString, QString> &attr)
{
    QDomElement dom = getDomElement(parent,tagname,attr);
    if(dom.isNull())
    {
        dom = createDomElement(tagname);
        if (!attr.isEmpty())
        {
            QMap<QString, QString>::ConstIterator iter = attr.begin();
            for (; iter != attr.end(); ++iter)
            {
                dom.setAttribute(iter.key(), iter.value());
            }
        }
        parent.appendChild(dom);
    }
    return dom;
}


template<class T>
QList<QDomElement> _getDomElementList(const T &parent, const QString &tagname, const QMap<QString, QString> &attr)
{
    QList<QDomElement> delist;
    QDomElement de;
    if (parent.isNull())  return delist;

    QDomNodeList list = parent.elementsByTagName(tagname);
    QStringList keys = attr.keys();
    QString attr_str;
    for (int i = 0; i < list.size(); ++i)
    {
        de = list.at(i).toElement();
        if (de.isNull())
        {
            continue;
        }
        if(keys.isEmpty())
        {
            delist.append(de);
            continue;
        }
        int count =0;
        foreach(const QString&key,keys)
        {
            attr_str = de.attribute(key);
            if( attr[key] == attr_str )
            {
                count++;
            }
        }
        if( keys.size() == count )
        {
            //返回第一次找到的那个
            delist.append(de);
        }
    }
    return delist;
}

QList<QDomElement> XmlParser::getDomElementList(const QString &tagname, const QMap<QString, QString> &attr)
{
    return _getDomElementList(m_xml,tagname,attr);
}



QList<QDomElement> XmlParser::getDomElementList(const QDomElement &parent, const QString &tagname, const QMap<QString, QString> &attr)
{
    return _getDomElementList(parent,tagname,attr);
}


QDomElement XmlParser::createDomElement(const QString&tagName)
{
    return m_xml.createElement(tagName);
}

bool XmlParser::setDomElement(QDomElement &new_ele, QDomNode &parent,const QMap<QString,QString>&attr)
{
    if(parent.isNull()) return false;
    if(new_ele.isNull()) return false;
    //查找相同的tag
    QDomNodeList nlist = parent.toElement().childNodes();
    bool operatorOne=false;
    if(attr.isEmpty())
    {
        for(int i=0;i<nlist.size();i++)
        {
            if(nlist.at(i).toElement().tagName() == new_ele.tagName())
            {
                if(!operatorOne)
                {
                    parent.replaceChild(new_ele,nlist.at(i));
                    operatorOne = true;
                }
                else
                    parent.removeChild(nlist.at(i));
            }
        }
        if(!operatorOne)
        {
            operatorOne = true;
            parent.appendChild(new_ele);
        }
    }
    else
    {
        for(int i=0;i<nlist.size();i++)
        {
            QDomElement de= nlist.at(i).toElement();
            if(de.tagName() != new_ele.tagName() ) continue;

            QStringList keys = attr.keys();
            int count=0;
            foreach(const QString&key,attr.keys())
            {
                if( de.attribute(key) == attr[key] )
                    count++;
            }
            if(count && count == keys.size())
            {
                if(!operatorOne)
                {
                    parent.replaceChild(new_ele,nlist.at(i));
                    operatorOne = true;
                }
                else
                    parent.removeChild(nlist.at(i));
            }
        }
        //!没有符合条件的项数据，在后面append
        if(!operatorOne)
        {
            operatorOne = true;
            parent.appendChild(new_ele);
        }
    }

    return true;
}

bool XmlParser::copyNode(QDomElement &dstTag, QDomElement &srcTag)
{
    QDomNodeList dstList = dstTag.childNodes();
    QDomNodeList srcList = srcTag.childNodes();
    if (dstList.count() != srcList.count())
        return false;

    QDomElement dstTag1, srcTag1;
    for (int i = 0; i < dstList.count(); ++i)
    {
        dstTag1 = dstList.at(i).toElement();
        srcTag1 = srcList.at(i).toElement();
        bool dstTag1HasChile = dstTag1.hasChildNodes();
        bool srcTag1HasChile = srcTag1.hasChildNodes();
        if (dstTag1HasChile && srcTag1HasChile)
        {
            if (dstTag1.hasAttributes())
            {
                QDomNamedNodeMap map = srcTag1.attributes();
                for (int j = 0; j < map.count(); ++i)
                {
                    dstTag1.setAttribute(map.item(j).nodeName(), map.item(j).nodeValue());
                }
            }

            QDomText domText = dstTag1.firstChild().toText();
            if (!domText.isNull())
            {
                domText.setData(srcTag1.text());
            }

            copyNode(dstTag1, srcTag1);
        }
        else if (dstTag1HasChile)
        {
            dstTag1.firstChild().toText().setData(QString());
        }
    }

    return true;
}

QDomElement XmlParser::findFirstElement(const QString &indexStr)
{
    QDomElement tag;
    if (m_xml.isNull())  return tag;

    QStringList list = indexStr.split("|", QString::SkipEmptyParts);
    if (list.size() < 1) return tag;

    tag = m_xml.firstChildElement();
    for (int i = 0; i < list.size(); ++i)
    {
        if (tag.isNull())
        {
            return tag;
        }
        tag = tag.firstChildElement(list.at(i));
    }
    return tag;
}
```

### 总结

> 1. 接口未实现“命名空间”；
> 2. 接口未实现“xml文档类型设置”；



