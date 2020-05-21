---
layout: post
title: 如何移动和缩放一个无边框窗口
date:   2020-05-21 10:44
description: 无边界窗口需要自己来处理窗口管理
categories: qt
comments: true
tags:
- qt,qml,FramelessWindowHint,resize,window

---
一个QT窗口如下可以做到无边框:

```qml
Window {
    id: window

    //Designer 竟然不支持..., 设计模式时要注意
    flags: Qt.FramelessWindowHint

    width: 500
    height: 300


    title: "Window Title"
}
```

不过要注意, 这样QT Designer不支持, 在设计的时候可以先注释掉, 最后在打开.

一旦设置了FramelessWindowHint, 系统就不管你这个窗口的移动和缩放了, 就需要自己来处理了.

那怎么处理哪, 大概有以下思路
1. 增加一个接收拖动事件的组件, 让它跟着鼠标移动
2. 增加一个缩放锚点, 随着鼠标缩放
3. 增加窗口四周的鼠标触发区域, 可以随着鼠标向四个方向缩放 (此文没包括实现) , 可以参考 <https://evileg.com/en/post/280/>{:target="_blank"}


## 我们先来看看如果拖动窗口, 代码如下:
```qml

/**
  头部操作区域.
*/
Rectangle {
    id: titleOpRect
    x: 0
    y: 0
    width: parent.width
    height: 30

    property string title : "Hello Board"

    MouseArea {
        id: mouseMoveWindowArea
        //height: 20
        anchors.fill: parent
        
        acceptedButtons: Qt.LeftButton
        
        property point clickPos: "0,0"
        
        onPressed: {
            clickPos = Qt.point(mouse.x, mouse.y)
            //isMoveWindow = true
        }
        onReleased: {            
            //isMoveWindow = false
        }

        onPositionChanged: {
            var delta = Qt.point(mouse.x - clickPos.x, mouse.y - clickPos.y)
            
            //如果mainwindow继承自QWidget,用setPos
            window.setX(window.x + delta.x)
            window.setY(window.y + delta.y)
        }
    }
    
    Button {
        id: closeButton
        
        width: 25
        height: parent.height
        text: "X"

        anchors.left: parent.left
        anchors.leftMargin: 0
        anchors.bottom: parent.bottom
        anchors.bottomMargin: 0


        flat: true
        font.bold: true
        font.pointSize: 14

        onClicked: {
            window.close()
        }
    }
    
    Text {
        id: titleText

        text: title
        anchors.bottom: parent.bottom

        //底部留点空间
        bottomPadding: 5

    } //end titleText



    
    Button {
        id: newStrikeButton
        
        width: 25
        height: parent.height
        text: "+"

        anchors.right: parent.right
        anchors.rightMargin: 0
        anchors.bottom: parent.bottom
        anchors.bottomMargin: 0

        flat: true
        font.pointSize: 22
    }

    Frame {
        width: titleOpRect.width
        height: 1
        anchors.bottom: titleOpRect.bottom
        Rectangle {
            height: parent.height
            width: parent.width
            color: "blue"
        }
    }
    
}
```

关键代码在MouseArea的onPressed 和 onPositionChanged 事件上. 比较容易.

## 我们再来看看如果缩放窗口

这次我们只是在窗口右下角放一个小矩形区域, 来捕获鼠标移动事件, 以实现缩放.

```QML

/**
  尾部操作区域.
*/
Rectangle {
    id: footOpRect
    anchors.bottom: parent.bottom
    width: parent.width
    height: 30


    //resize区域
    MouseArea {
        id: resize
        anchors {
            right: parent.right
            bottom: parent.bottom
        }
        width: 15
        height: 15
        cursorShape: Qt.SizeFDiagCursor

        property point clickPos: "1,1"

        //保持窗口左上角不动
        property int oldX
        property int oldY

        onPressed: {
            clickPos = Qt.point(mouse.x, mouse.y)
            //oldWidth = window.width
            //oldHeight = window.height
            oldX = window.x
            oldY = window.y
        }

        onPositionChanged: {
            var delta = Qt.point(mouse.x - clickPos.x,
                                 mouse.y - clickPos.y)

            var minWidth = 100;
            var minHeight = 100;

            //最小
            var newWidth = (window.width + delta.x)<minWidth?minWidth:(window.width + delta.x);

            //最小
            var newHeight = (window.height + delta.y)<minHeight?minHeight:(window.height + delta.y);

            window.width = newWidth;
            window.height = newHeight;
            window.x = oldX
            window.y = oldY
        }

        onReleased: {
        }

        Rectangle {
            id: resizeHint
            color: "red"
            anchors.fill: resize
        }
    }
}

```

这段代码网上很常见, 
1. 不过这里判断了窗口的最小高度和宽度, 不会导致窗口太小看不见
2. 鼠标移动会导致负数, 所以这里也做了判断.
3. 保持窗口左上角不动, 否则鼠标跑来跑去有问题.



That's all, Thanks.
