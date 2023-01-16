---
layout: post
title: 如何在使用Jenkins构建时输入发布内容描述
date:   2023-01-16 15:54
description: 使用Jenkins构建时输入一个发布原因
categories: build
comments: true
tags:
- jenkins
---

很多团队使用Jenkins来构建项目, 或者部署上线, 那么上线的时候我们希望知道这次发布的内容是什么, 或者在历史记录里能看到以前都发布过什么, 
而Jenkins的默认记录是空空如也, 什么也看不到, 只有个顺序号和时间而已.

![默认的构建历史](/img/build/ori-buildhistory.png )

那么我们肯定希望能在构建历史里看到发布的内容, 相应的分支, 发布人等等信息, 那就需要用到插件了, 我们可以安装2个插件:
* Build Name and Description Setter: 可以修改构建的名字和描述 https://plugins.jenkins.io/build-name-setter/
* build user vars: 可以使用构建人的信息 https://plugins.jenkins.io/build-user-vars-plugin/

在Jenkins的插件管理中安装上述两个插件, 然后来修改你的构建任务

增加一个用户输入的纯文本参数, 用来描述发布内容

![添加参数](/img/build/add-text-parameter.png )

然后设置我们的组件, 设置一个名字 reason, 以方便在后面作为变量使用

![输入发布内容](/img/build/userinput.png )

然后在构建的步骤里增加一个新的步骤 (安装了插件才有)

![增加构建步骤](/img/build/build-add-step.png )

可以看到有很多步骤, 我们安装插件新增的步骤有 "Changes build name" 和 "Changes build description", 还有一个 "Update build name"
我们现在使用 "Change build name"来自定义我们想显示的名字, 里面支持变量:

![修改名字](/img/build/changebuildname.png )

我们在输入框里使用三个参数:
* ${BUILD_NUMBER}   构建号
* ${BRANCH}  Git分支
* ${reason}  发布原因描述

你还可以使用上面的 "Build user vars"带来的参数:
* BUILD_USER	构建人的 Full name (first name + last name)
* BUILD_USER_FIRST_NAME	构建人的 First name
* BUILD_USER_FIRST_NAME	构建人的 First name
* BUILD_USER_LAST_NAME	构建人的 Last name
* BUILD_USER_ID	构建人的 Jenkins user ID
* BUILD_USER_GROUPS	构建人的 Jenkins user groups
* BUILD_USER_EMAIL	构建人的 Email address

我们还可以通过拖动把这个步骤放到构建的第一步去, 这样可以看到及时的信息更新.

Jenkins构建其实有三个环节
* 构建环境
* 构建
* 构建后操作

我们刚才是在构建环节设置的, 我们也可以把这个设置移动到 "构建环境" 环节, 当然要注意用到的参数和相应的环节要配套, 否则参数可能无法取到值

![设置参数](/img/build/build_env_before.png )
这个看起来更简单一点, 也更提前设置了, 所以我们推荐这个, 除非你用的参数要等到构建时才有值.


配置完毕, 保存后, 我们开始进行构建:

![输入需要的值](/img/build/inputvalue.png )

构建完成后, 我们看看构建历史

![构建历史](/img/build/buildhistory.png )

现在这样就方便多了, 而不是一堆简单的数字了.

这样钉钉推送的通知也就更有意义了, 否则鬼才知道发布了什么.


 

 
