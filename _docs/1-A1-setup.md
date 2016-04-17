---
pageid: 1-A1-local-setup
series: buildingf8app
partlabel: 附录 1
type: appendix
title: "本地设置运行 App"
layout: docs
permalink: /tutorials/building-the-f8-app/local-setup/
---

> *这一系列的教程写于 F8 2016 app 开发期间，为的是用简单的文字来介绍 React Native 及其开源生态。教程于 2016 年 4 月 13 号发布到 makeitopen.com，内容翔实很有指导意义，[我](https://github.com/liaohuqiu) 看了之后花了周六日两天时间，翻译成中文，方便大家学习。* 

你可以从 [iOS App Store](https://itunes.apple.com/us/app/f8/id853467066), 或者从 [Google Play Store](https://play.google.com/store/apps/details?id=com.facebook.f8) 下载安装 F8。

如果你在阅读本教程时，你想在本地运行 App，你可根据以下简要的步骤进行设置：

如果你是在 Windows 或者 Linux 上进行设置，请参考这个 React Native Android 版本在 Windows 和 Linux 上的 [相关支持](http://facebook.github.io/react-native/docs/linux-windows-support.html#content)。

>  如果还没配置过 React Native 的，可以看看这个 [React Native 配置和起步](http://www.liaohuqiu.net/cn/posts/react-native-1/)。

### 需求

开始之前，请需要安装这些前置依赖:

1. [React Native](http://facebook.github.io/react-native/docs/getting-started.html)
2. [CocoaPods](http://cocoapods.org) 1.0+ (只有 iOS 需要)
3. [MongoDB](https://www.mongodb.org/downloads) (本地运行 Parse Server 需要)

### 配置

#### 1. **Clone 项目**

```
$ git clone https://github.com/fbsamples/f8app.git
$ cd f8app
```

#### 2. **安装依赖** (npm v3+):

```
$ npm install
$ (cd ios; pod install)        # 只有 ios 需要
```

#### 3. **确认 MongoDB 是已经运行的:**

```
$ lsof -iTCP:27017 -sTCP:LISTEN
```

如果你要使用外部的 MongoDB，请设置 `DATABASE_URI`

```
$ export DATABASE_URI=mongodb://example-mongo-hosting.com:1337/my-awesome-database
```

#### 4. **启动 Parse/GraphQL:**

```
$ npm start
```

#### 5. **导入示例数据** (本地的 Parse Server 应该已经在运行了):

```
$ npm run import-data
```

打开这些链接，确保一切正常：

* Parse Dashboard: [http://localhost:8080/dashboard](http://localhost:8080/dashboard)
    
* Graph*i*QL: [http://localhost:8080/graphql](http://localhost:8080/graphql?query=query+%7B%0A++schedule+%7B%0A++++title%0A++++speakers+%7B%0A++++++name%0A++++++title%0A++++%7D%0A++++location+%7B%0A++++++name%0A++++%7D%0A++%7D%0A%7D)

<img src="static/images/screenshot-server@2x.png">


#### 6. **运行 Android**:

```
$ react-native run-android
$ adb reverse tcp:8081 tcp:8081   # 使用 adb 端口代理，确保设备可以访问 Packager 和 GraphQL server
$ adb reverse tcp:8080 tcp:8080
```


#### 7. **运行 iOS:**

```
$ react-native run-ios
```
