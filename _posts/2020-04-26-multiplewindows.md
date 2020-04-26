---
layout: post
title: QML用同一模版多开主窗口
date:   2020-04-26 16:46
description: 使用QML创建多个一级窗口
categories: qt
comments: true
tags:
- qt,qml,window
---

如何动态地创建多个长的一样的主窗口哪(数据当然不一样), 用QML也是可以实现的.

简单的地说, 就是调用多次load即可.
```cpp
    QCoreApplication::setAttribute(Qt::AA_EnableHighDpiScaling);

    QGuiApplication app(argc, argv);
    QQmlApplicationEngine engine;

    const QUrl url(QStringLiteral("qrc:/part1.qml"));

    engine.load(url);
    //多次调用
    engine.load(url);

    return app.exec();

```

那多开了多个窗口之后, 如何设置不同的数据哪? 如何区分不同的窗口哪


```cpp
    QObject *rootObj1 = engine.rootObjects()[0];
    QObject *rootObj2 = engine.rootObjects()[1];
``` 
就可以找到刚刚创建的窗口, 以及里面的控件, 例如下面修改输入框的文字和颜色

```cpp
    //直接修改控件属性: 控件的objectName为 textInput, 如果只有一个TextInput, 则省略参数
    QQuickItem *textInput2 = rootObj1->findChild<QQuickItem*>("textInput");
    if (textInput2){
        textInput2->setProperty("text", "I am second one");
        textInput2->setProperty("color", QColor(Qt::blue));
    }

```

之前我们用ListView的时候, 设置的setContextProperty该如何独立设置哪?
```cpp
    engine.rootContext()->setContextProperty("myModel", &model);
```

**经过试验, 发现这个rootContext独一份, 也就是说这是全局统一的rootContext,而通过contextForObject获取的Context是只读的, 没有办法修改.** 

那看来就只能通过控件本身的setProperty修改了

例如, 修改listView的model数据:
```cpp
    QQuickItem *listView1 = rootObj1->findChild<QQuickItem*>("listView");
    if (listView1){
        qDebug("find listview1");
        listView1->setProperty("model", QVariant::fromValue(model1));
    }

```

注意, 为了声明ListView的Model的数据类型, 需要做一些定义

```cpp
    //注册类型: 让视图知道注入的是啥类型
    qmlRegisterType<MyListModel>("MyList", 1, 0, "MyListModel");
```

视图QML部分: 
```qml
import QtQuick 2.12
import QtQuick.Window 2.12
import QtQuick.Layouts 1.3
import QtQuick.Controls 2.5

import MyList 1.0


Window {
    id: rectangle1
    visible: true
    width: 340
    height: 280
    objectName: "window"

    property string keyname: "whoknows"

    ListView{
        width: 100; height: 100

        id: listView
        objectName: "listView"

        Layout.fillWidth: true
        Layout.fillHeight: true

        //默认数据为空, 但是声明了类型
        model: MyListModel{}

        delegate: Rectangle {
            height: 25
            width: 100
            Text { text: "hello " + model.name }
        }

    }

}

```

其中 MyListModel 是一个继承自 QAbstractListModel的模型类, 此处仅为演示, 完整的类需要考虑增删修改等各种函数实现.

项目代码:  <https://github.com/cnscud/learn/tree/master/qt/multiplewindows>