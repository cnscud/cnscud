---
layout: post
title: QML用Instantiator动态创建顶级窗口
date:   2020-04-27 17:48
description: 用Instantiator动态创建多个顶级窗口
categories: qt
comments: true
tags:
- qt,qml
---
### 关键点
* 使用Model驱动Instantiator
* QML里面的hashmap: QQmlPropertyMap


上一次说到用 QQmlApplicationEngine 多次load的方式创建多个一级窗口 [详见这里](https://blog.cnscud.com/qt/2020/04/26/multiplewindows.html){:target="_blank"}, 
但是窗口数据需要自己设置, 不如Model设置方式方便, 窗口如果比较复杂, 数据设置起来比较麻烦,而且管理窗口也会比较麻烦.  

这里就说说用 Instantiator 这个QML里面的组件, 这个组件是根据模版用来动态创建多个QML组件的 
(A Instantiator can be used to control the dynamic creation of objects, or to dynamically create multiple objects from a template.). 
只是没想到的是竟然可以来创建多个一级窗口.

先看一下简单的: 
```qml
import QtQuick 2.12
import QtQuick.Controls 2.5
import QtQuick.Window 2.3


Instantiator {
    id: windowInstantiator

    model: ListModel {
        id: windowModel
        ListElement { title: "Initial Window"; x: -200 }
        ListElement { title: "Second Window"; x:300 }
    }

    delegate: Window {
        id: window
        visible: true
        width: 640
        height: 480
        x: Screen.width/2 - window.width/2 + model.x
        y: Screen.height / 2 - window.height/2

        title: model.title

        Rectangle {
            width: 150
            height: 50
            Button {
                text: qsTr("Duplicate Window")
                anchors.horizontalCenter: parent.horizontalCenter
                anchors.bottom: parent.bottom
                onClicked: windowModel.append({ "title": "Window #" + (windowModel.count +1)})
            }
        }
    }
}

```

* 里面直接用了ListModel来简化代码, 
* delegate里面用model来读取数据, 然后还用了model的count属性来看一共有多少个窗口. 
* 其中用了Screen类来判断窗口的位置.
* 一开始仅有2个窗口, 点击按钮则可以动态增加窗口(也就是修改Model数据)


那接下来我们看看如何直接通过Model来控制创建多个窗口, 为了与真实情况接轨, 里面放一个ListView, 视图QML如下: 
```qml
import QtQuick 2.12
import QtQuick.Controls 2.5
import QtQuick.Window 2.3
import QtQuick.Layouts 1.3

Instantiator {
    id: windowInstantiator

    model: rootModel

    delegate: Window {
        id: window
        visible: true
        width: 640
        height: 480
        x: Screen.width/2 - window.width/2 + modelData.x
        y: Screen.height / 2 - window.height/2


        title: modelData.title + " (Window Count: " + count + " )"

        ListView{
            width: 100; height: 100

            id: listView
            objectName: "listView"

            Layout.fillWidth: true
            Layout.fillHeight: true

            model: modelData.listModel

            delegate: Rectangle {
                height: 25
                width: 100
                Text { text: "hello " + model.name }
            }

        }

    }
}

```

其中Instantiator的model是 rootModel, 而ListView的Model用的是 listModel数据.

接下来我们看看如何设置数据, 有两种方法:
* 实现一个 QAbstractListModel, 这个直接用QT新建一个QT Model就可以.
* 直接用QList来包装一个列表, 可以自己创建一个类, 或者直接用HashMap?

## 实现一个简单的类
```cpp

class WindowModel : public QObject
{
  Q_OBJECT
  Q_PROPERTY(QString title READ title WRITE setTitle NOTIFY titleChanged)
  Q_PROPERTY(int x READ getX WRITE setX NOTIFY xChanged)
  Q_PROPERTY(MyListModel* listModel READ getListModel WRITE setListModel NOTIFY listModelChanged)


 public:
  explicit WindowModel(QObject *parent = nullptr);

  QString title() const;
  void setTitle(const QString &title);

  MyListModel *getListModel() const;
  void setListModel(MyListModel *value);

  int getX() const;
  void setX(int x);

 Q_SIGNALS:
  void titleChanged();
  void xChanged();
  void listModelChanged();


private:
  QString _title = "";

  int m_x = 0;
  MyListModel* m_listModel;
};

```

然后直接放到QList里面: 
```cpp
    //用自己定义Model
    WindowModel *win1 = new WindowModel();
    win1->setTitle( "First");
    win1->setX(-100);
    win1->setListModel(model1);

    WindowModel *win2 = new WindowModel();
    win2->setTitle( "Second");
    win2->setX(300);
    win2->setListModel(model2);

    //QVariantList list;
    QList<WindowModel *> winlist;
    winlist << win1 << win2;
```


## 或者用HashMap形式
看了看QHash, 并不能直接在QML里面使用, 可以用QQmlPropertyMap, 则可以在QML里面用modelData.xxx的方式直接调用数据.

```cpp
    //用类似Map结构的QQmlPropertyMap
    QQmlPropertyMap hash1;
    hash1.insert("title", "First");
    hash1.insert("x",  QVariant::fromValue(-100));
    hash1.insert("listModel", QVariant::fromValue(model1));

    QQmlPropertyMap hash2;
    hash2.insert("title", "Second");
    hash2.insert("x",  QVariant::fromValue(300));
    hash2.insert("listModel", QVariant::fromValue(model2));

    //QVariantList list;
    QList<QQmlPropertyMap*> list;
    list << &hash1 << &hash2;


    QQmlApplicationEngine engine;
    engine.rootContext()->setContextProperty("rootModel", QVariant::fromValue(list));
```



### 注意modelData与model的区别
* 用QList做Model, 在QML里面调用modelData.xxx
* 用QAbstractListModel的子类做Model, 在QML里面调用model.xxx

官方说明详见 <https://doc.qt.io/qt-5/qtquick-modelviewsdata-cppmodels.html>{:target="_blank"}

  
### 更详细的请查看项目源码
<https://github.com/cnscud/learn/tree/master/qt/windowByInstantiator>{:target="_blank"}