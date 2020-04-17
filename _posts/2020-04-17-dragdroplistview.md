---
layout: post
title: Qt学习-ListView的拖拽
date:   2020-04-17 15:09
description: 使用MouseArea和DropArea对ListView拖拽
comments: true
categories: qt
tags:
- qt,drag,drop,listview
---
最近在学习Qt 里面的QML, 使用DropArea和MouseArea实现了ListView的拖拽.

想起了当年用Delphi, 差不多一样的东西, 不过那是2000了. Delphi也是不争气啊, 多好的IDE, 硬生生发展不起来..... 


```qml
/**
  samples changed from Qt tutorial "dynamicview3"
*/

import QtQuick 2.0
import QtQml.Models 2.1


Rectangle {
    id: root

    width: 300; height: 400

    Component {
        id: dragDelegate

        MouseArea {
            id: dragArea


            anchors { left: parent.left; right: parent.right }
            height: content.height

            // Disable smoothed so that the Item pixel 
            // from where we started the drag remains under the mouse cursor
            drag.smoothed: false


            drag.target:  content

            drag.axis: Drag.YAxis

            Rectangle {
                id: content

                anchors {
                    horizontalCenter: parent.horizontalCenter
                    verticalCenter: parent.verticalCenter
                }
                width: dragArea.width; height: column.implicitHeight + 4

                border.width: 1
                border.color: "lightsteelblue"

                color: dragArea.drag.active ? "lightsteelblue" : "white"

                Behavior on color { ColorAnimation { duration: 100 } }

                radius: 2

                Drag.active: dragArea.drag.active

                Drag.source: dragArea
                Drag.hotSpot.x: width / 2
                Drag.hotSpot.y: height / 2

                states: State {
                    when: content.Drag.active

                    ParentChange { target: content; parent: root }
                    AnchorChanges {
                        target: content
                        anchors { horizontalCenter: undefined; verticalCenter: undefined }
                    }
                }

                Column {
                    id: column
                    anchors { fill: parent; margins: 2 }
                    Text { text: 'Name: ' + name }
                }

            }

            DropArea {
                anchors { fill: parent; margins: 10 }

                onEntered: {
                    visualModel.items.move(
                            drag.source.DelegateModel.itemsIndex,
                            dragArea.DelegateModel.itemsIndex)
                }
            }

        }
    }

    DelegateModel {
        id: visualModel

        model: PetsModel {}
        delegate: dragDelegate
    }

    ListView {
        id: view

        anchors { fill: parent; margins: 2 }

        model: visualModel

        spacing: 4

        cacheBuffer: 50

        moveDisplaced: Transition {
                  NumberAnimation { properties: "x,y"; duration: 200 }
              }

    }

}
```


用的PetModels如下: 
```qml
import QtQuick 2.0

ListModel {
    ListElement {
        name: "Polly"
        type: "Parrot"
        age: 12
        size: "Small"
    }
    ListElement {
        name: "Penny"
        type: "Turtle"
        age: 4
        size: "Small"
    }
    ListElement {
        name: "Warren"
        type: "Rabbit"
        age: 2
        size: "Small"
    }
    ListElement {
        name: "Spot"
        type: "Dog"
        age: 9
        size: "Medium"
    }
    ListElement {
        name: "Schrödinger"
        type: "Cat"
        age: 2
        size: "Medium"
    }
    ListElement {
        name: "Joey"
        type: "Kangaroo"
        age: 1
        size: "Medium"
    }
    ListElement {
        name: "Kimba"
        type: "Bunny"
        age: 65
        size: "Large"
    }
    ListElement {
        name: "Rover"
        type: "Dog"
        age: 5
        size: "Large"
    }
    ListElement {
        name: "Tiny"
        type: "Elephant"
        age: 15
        size: "Large"
    }
}
```