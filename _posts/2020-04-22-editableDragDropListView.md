---
layout: post
title: QT-可拖拽可编辑的多控件ListView
date:   2020-04-22 09:15
description: 可以拖拽,可以编辑行, 一行多个控件...到处搜索的例子
categories: qt
comments: true
tags:
- qt,qml,ListView
---
## 目标
结合前面的2篇文章, 继续升级QML版本的ListView: 又要拖拽, 又要可编辑, 还得支持多个控件.

## 循序渐进

本文基于前一篇的基础: [Qt-可编辑的ListView](/qt/2020/04/18/editable-listview.html) 要循序渐进的学习.

## 几个关键点
* 要用拖拽, 就不能用Layout了 (大部分情况应该是)
* 条条大路通罗马, 但是没有找到官方的示例...只好自己写
* 尽量简洁, 而不是自己控制所有状态(虽然也可以做到, 就是太麻烦)

## 示意图

![](/img/posts/editlistview.jpg "截图")

## 代码如下

main.cpp, main.qml没啥变化, 和之前的一样.

主要的EditDragList.qml代码如下, 里面注释很多, 还有很多调试打印, 自己可以去除: 

```qml
import QtQuick 2.12
import QtQuick.Controls 2.5
import QtQuick.Layouts 1.3
import QtQml.Models 2.1


/**
  1. 通过设置currentIndex, 属性自动变化. 支持键盘
  2. 支持拖拽
*/
Rectangle {
    id: root

    //列表容器
    Rectangle {
        id: itemRoot
        width: root.width
        height: root.height - itemOp.height

        //调试显示
        //color: "blue"
        //border.width: 2

        Component {
            id: delegateItem

            MouseArea {
                id: mouseArea
                width: itemRoot.width

                height: itemRect.height
                anchors {
                    left: parent.left
                    right: parent.right
                }

                //hoverEnabled: true

                //拖拽设置
                drag.smoothed: false
                drag.target: itemRect
                drag.axis: Drag.YAxis



                onClicked: {
                    console.debug("onClicked")

                    //方法1: 设置当前
                    listView.currentIndex = index

                    console.log(("MouseArea Click listview currentIndex: "
                                 + listView.currentIndex + " index: " + index))
                    console.log(("MouseArea Click ListView isCurrentItem: "
                                 + ListView.isCurrentItem))

                    // 在onFocusChanged 中设置了, 此处不需要了
                    //textinput.focus = true;
                }


                onFocusChanged: {
                    if (focus) {
                        console.debug("=====got focus of mouseArea, start")
                        console.debug(("got focus of mouseArea, listview currentIndex: "
                                     + listView.currentIndex + " index: " + index))
                        console.debug("got focus of mouseArea, isCurrentItem: " +  mouseArea.ListView.isCurrentItem)
                        console.debug("got focus of mouseArea, drag is active: " + drag.active)
                        console.debug("got focus of mouseArea, textInput visible: " + textinput.visible)

                        textinput.focus = true

                        console.debug("=====got focus of mouseArea, end!")
                    }
                    else {
                        console.debug(("lost focus of mouseArea, listview currentIndex: "
                                     + listView.currentIndex + " index: " + index))

                        console.debug("lost focus of mouseArea, isCurrentItem: " +  mouseArea.ListView.isCurrentItem)
                    }
                }

                //FIXME: 目前某行处于编辑状态, 然后其他行拖动和此行交换, 则会crash, 原因待查 2020.4.21
                //目前解决的思路: 一旦开始拖拽, 则退出编辑状态
                drag.onActiveChanged: {
                    if (mouseArea.drag.active) {
                        //开始拖动时: 设置当前Item为空
                        listView.currentIndex = -1
                    }
                }

                //实际显示内容
                Rectangle {
                    id: itemRect
                    height: 40
                    width: mouseArea.width

                    anchors {
                        horizontalCenter: parent.horizontalCenter
                        verticalCenter: parent.verticalCenter
                    }

                    //拖拽设置
                    Drag.active: mouseArea.drag.active
                    Drag.source: mouseArea
                    Drag.hotSpot.x: width / 2
                    Drag.hotSpot.y: height / 2


                    //拖拽的状态变化
                    states: State {
                        when: itemRect.Drag.active

                        ParentChange {
                            target: itemRect
                            parent: itemRoot
                        }
                        AnchorChanges {
                            target: itemRect
                            anchors {
                                horizontalCenter: undefined
                                verticalCenter: undefined
                            }
                        }
                    }


                    CheckBox {
                        id: chkbox
                        width: 50
                        //height: parent.height
                        anchors.bottom: parent.bottom
                        //底部留点空间
                        bottomPadding: 3

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
                            anchors.bottom: parent.bottom

                            //控制可见: 不是当前
                            visible: !mouseArea.ListView.isCurrentItem

                            //底部留点空间
                            bottomPadding: 3

                        } //end textShow

                        TextInput {
                            id: textinput

                            anchors.bottom: parent.bottom

                            color: "blue"

                            //底部留点空间
                            bottomPadding: 3

                            text: model.description

                            //控制是否编辑状态
                            visible: mouseArea.ListView.isCurrentItem
                            enabled: mouseArea.ListView.isCurrentItem

                            //focus: true
                            selectByMouse: true

                            //Debug: 不变不会进入的, 例如已经focus, 再次设置不会触发此事件
                            onFocusChanged: {
                                if (focus) {
                                    console.debug("got focus " + "textInput")
                                }
                                else {
                                    console.debug("lost focus " + "textInput")
                                }
                            }

                            onEditingFinished: {
                                console.debug("=== start onEditingFinished ")
                                model.description = textinput.text

                                //方法1: 设置index
                                if (listView.currentIndex == index) {
                                    listView.currentIndex = -1
                                }

                                console.log(("TextInput listview currentIndex: "
                                             + listView.currentIndex + " index: " + index))
                                console.log(("TextInput ListView isCurrentItem: "
                                             + mouseArea.ListView.isCurrentItem))

                                console.debug("=== end onEditingFinished ")
                            }
                        } //end TextInput

                    } //end textSection Rectangle

                    Frame {
                        width: itemRect.width
                        height: 1
                        anchors.bottom: itemRect.bottom
                        //anchors.topMargin: 1

                    }

                } //end itemRect Rectangle


                DropArea {
                    anchors {
                        fill: parent
                        margins: 10
                    }


                    onEntered: {
                        console.debug("===== start DropArea onEntered")
                        console.debug("drag.source.DelegateModel.itemsIndex: " + drag.source.DelegateModel.itemsIndex)
                        console.debug("mouseArea.DelegateModel.itemsIndex: " + mouseArea.DelegateModel.itemsIndex )

                        //移动Delegate
                        visualModel.items.move(
                                    drag.source.DelegateModel.itemsIndex,
                                    mouseArea.DelegateModel.itemsIndex, 1)

                        //移动Model: 不移动的话model和delegate就不同步了
                        visualModel.model.move(
                                    drag.source.DelegateModel.itemsIndex,
                                    mouseArea.DelegateModel.itemsIndex, 1)

                        console.debug("===== end DropArea onEntered")
                    }
                }
                //end DropArea


            } //end MouseArea

        } //end Component

        DelegateModel {
            id: visualModel

            model: MyModel {}
            delegate: delegateItem
        }

        ListView {
            id: listView

            width: parent.width
            height: parent.height

            keyNavigationEnabled: true

            //clip: true
            model: visualModel
            //delegate: delegateItem

            //默认不要是第一个, 否则第一个是可编辑状态(针对方法1)
            Component.onCompleted: {
                currentIndex = -1
            }

            //默认焦点
            focus: true

            spacing: 0
        }
    }

    //操作区容器
    Rectangle {
        id: itemOp

        width: root.width
        //指定高度
        height: 50

        //和上一个区域挨着
        anchors.top: itemRoot.bottom

        Button {
            text: qsTr("Add New Item")
            width: parent.width

            onClicked: {
                var c = listView.model.model.rowCount()
                //插入在第一个
                listView.model.model.insert(0, {
                                                "description": "Buy a new book " + (c + 1),
                                                "done": false
                                            })

                //追加
                //listView.model.model.append({ "description": "Buy a new book " + (c + 1), "done": false })

                //设置焦点, 否则listView就没焦点了
                listView.focus = true
                listView.currentIndex = 0
            }
        }
    }
}

```

## 项目地址

和前一篇一样, 还在同一个项目里面: <https://github.com/cnscud/learn/tree/master/qt/editListView>