---
layout: post
title: Qt-可编辑的ListView
date:   2020-04-18 21:01
description: 一个ListView, 使用Text/TexInput
categories: qt
comments: true
tags:
- qt,listview,textinput
---
新建一个QML项目, main.cpp不动如下: 
```cpp
#include <QGuiApplication>
#include <QQmlApplicationEngine>

int main(int argc, char *argv[])
{
    QCoreApplication::setAttribute(Qt::AA_EnableHighDpiScaling);

    QGuiApplication app(argc, argv);

    QQmlApplicationEngine engine;
    const QUrl url(QStringLiteral("qrc:/main.qml"));
    QObject::connect(&engine, &QQmlApplicationEngine::objectCreated,
        &app, [url](QObject *obj, const QUrl &objUrl) {
            if (!obj && url == objUrl)
                QCoreApplication::exit(-1);
        }, Qt::QueuedConnection);
    engine.load(url);

    return app.exec();
}

```

主界面main.qml如下

```qml
import QtQuick 2.12
import QtQuick.Window 2.12

Window {
    visible: true
    width: 640
    height: 480
    title: qsTr("Hello World")


    DemoList {
        anchors.centerIn: parent
        width: parent.width
        height: parent.height
    }

}
```

用到的Model文件如下: 

```qml
import QtQuick 2.0

ListModel {
    ListElement {
        description: "Wash the car"
        done: true
    }
    ListElement {
        description: "Read a book"
        done: false
    }

    ListElement {
        description: "Buy a cup"
        done: false
    }

}

```

主要界面:

```qml
import QtQuick 2.12
import QtQuick.Controls 2.5
import QtQuick.Layouts 1.3
import QtQml.Models 2.1


/**
  方法2: 通过设置currentIndex, 属性自动变化. 支持键盘
*/
ColumnLayout {

    RowLayout {

        Layout.fillHeight: true

        Layout.leftMargin: 10
        Layout.rightMargin: 20
        Layout.alignment: Qt.AlignHCenter | Qt.AlignTop

        Component {
            id: delegateItem

            Rectangle {
                id: dragRect
                height: 50
                width: parent.width

                onFocusChanged: {
                    if(focus){
                        console.debug("got focus dragRect" );
                        textinput.focus = true;
                    }
                }

                CheckBox {
                    id: chkbox
                    width: 50
                    checked: model.done
                    onClicked: model.done = checked
                }


                Rectangle {
                    id: textSection
                    height: parent.height

                    width: parent.width - chkbox.width
                    anchors.left: chkbox.right

                    Text {
                        id: textShow

                        text: model.description
                        font.underline: true
                        font.bold: true
                        anchors.verticalCenter: parent.verticalCenter

                        visible: !dragRect.ListView.isCurrentItem
                        color: "black"


                    } //end textShow


                    TextInput {
                        id: textinput

                        anchors.verticalCenter: parent.verticalCenter

                        color: "blue"

                        text:  model.description
                        visible: dragRect.ListView.isCurrentItem
                        enabled: dragRect.ListView.isCurrentItem
                        focus: true

                        onFocusChanged: {
                            if(focus){
                                console.debug("got focus" + "textInput");
                            }
                        }

                        onEditingFinished: {
                            model.description = textinput.text

                            //方法1: 设置index
                            if(listView.currentIndex == index){
                                listView.currentIndex = -1;
                            }


                            console.log(("TextInput listview currentIndex: " + listView.currentIndex +  " index: " + index ))
                            console.log(("TextInput ListView isCurrentItem: " + dragRect.ListView.isCurrentItem))

                        }

                    } //end TextInput

                } //end textSection Rectangle

                MouseArea {
                    id: mouseArea
                    width: parent.width
                    height: parent.height
                    hoverEnabled: true

                    z: -1 //避免遮住checkbox

                    onClicked: {
                        //方法1: 设置当前
                        listView.currentIndex = index

                        console.log(("MouseArea listview currentIndex: " + listView.currentIndex +  " index: " + index ))
                        console.log(("MouseArea ListView isCurrentItem: " + dragRect.ListView.isCurrentItem))

                        // 在dragRect的 onFocusChanged 中设置了, 此处不需要了
                        //textinput.focus = true;
                    }
                }


            }
        } //end Row Rectangle

        ListView {
            id: listView
            Layout.fillWidth: true
            Layout.fillHeight: true
            keyNavigationEnabled: true

            //clip: true
            model: MyModel {}
            delegate: delegateItem

            //默认不要是第一个, 否则第一个是可编辑状态(针对方法1)
            Component.onCompleted : {
                currentIndex = -1;
            }

            //默认焦点
            focus: true
        }
    }

    RowLayout {
        Button {
            text: qsTr("Add New Item")
            Layout.fillWidth: true

            onClicked: {
                var c = listView.model.rowCount();
                listView.model.append({
                                          "description": "Buy a new book " + (c + 1),
                                          "done": false
                                      })

                //设置焦点, 否则listView就没焦点了
                listView.focus = true;
                listView.currentIndex = c;
            }
        }
    }
}

```

## 几个关键点, 详见代码: 
* Text/TextInput的visible属性
* MouseArea的点击事件: 设置 listView.currentIndex = index
* MouseArea避免遮挡: z: -1 //避免遮住checkbox
* ListView的 Component.onCompleted, 设置默认不选中
* ListView默认设置: focus: true
* **TextInput的onEditingFinished, 处理编辑完成事件**
* 组件dragRect的onFocusChanged, 使TextInput获得焦点

## 项目代码
<https://github.com/cnscud/learn/tree/master/qt/editListView>