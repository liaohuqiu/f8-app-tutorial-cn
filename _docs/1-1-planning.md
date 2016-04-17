---
pageid: 1-1-planning
series: buildingf8app
partlabel: 第一章
title: App 的筹备
layout: docs
permalink: /tutorials/building-the-f8-app/planning/
intro: >
  在这第一部分，我们将谈谈，我们是如何筹备这个 app 的，和进行技术选型的。
---

> *这一系列的教程写于 F8 2016 app 开发期间，为的是用简单的文字来介绍 React Native 及其开源生态。* 


在第一部分，我们会谈谈我们是如何筹备这个 app 的，在后面的部分，我们会一起看一些代码片段，讨论我们跨平台设计的考虑，分析我们 app 的数据层，阐释我们单元测试的策略。

### 迁移到 React Native

在 2015 年的 F8 会议上，React Native for Android 发布。这一年的 F8 iOS 版本是用 React Native 开发的。Android 版本还是原生开发。 在这年之前，iOS 和 Android 都是原生开发。React Native for Android 发布，意味着我们有机会削减 app 的业务逻辑和 UI 代码了。一些 Facebook 的团队使用 React Native，[重用了大约 85% 的代码](https://code.facebook.com/posts/1189117404435352/react-native-for-android-how-we-built-the-first-cross-platform-react-native-app/)

[在第二部分]({{ site.baseurl }}/tutorials/building-the-f8-app/design/) 我们还会谈到，React Native 还使得可以和 UI 设计师一起进行可视化组件的快速原型设计。

现在让我们看看，如果要迁移到 React Native，我们需要考虑些什么:

### 选择数据层

2014 和 2015 的 app 都采用 [Parse Cloud Code](https://parse.com/) 作为数据后端。在筹备 2016 年的应用时，使用 Parse 可以复用已经存在的数据结构，项目可以快速开始。

当然，使用 Parse 还有另外一些原因。在会前甚至会议期间，在 app 中显示的内容需要很高的更新频率。更新这些内容，无需很强的专业技能，比如对电子表格极其熟悉。Parse Cloud Code 的管理后台，完美地满足了这些需求。  

考虑到以上因素, Parse 成了我们这个 app 数据后端的最佳的选择。根据 [Parse Cloud Code 的关闭通知](http://blog.parse.com/announcements/moving-on/), 我们决定过渡到新的开源 [Parse Server](http://blog.parse.com/announcements/introducing-parse-server-and-the-database-migration-tool/) 以及 [Parse Dashboard](https://github.com/ParsePlatform/parse-dashboard) 。

React Native 无需和数据层紧密关联，我们在实现 UI 界面和业务逻辑的时候，可以使用一些假数据。只要数据结构一样，紧要一些很小的调整就可以更换 app 的数据源。对于 F8 这个 app 来说，开发完之后，从 Parse Cloud Code 过渡到开源的 Parse Server 将会很简单，我们在 [数据集成章节]({{ site.baseurl }}/tutorials/building-the-f8-app/data/) 会详细说明。

<h3 id='data-access-with-react-native'>React Native 的数据访问</h3>

现在已经有一个包 [Parse + React](https://github.com/ParsePlatform/ParseReact) 可让 React Native 使用 Parse。但有一个问题就是，这个包不支持离线数据同步。而会场的 wi-fi 又极不稳定，app 必须能在无网络的条件下工作。所以我们必须要自己实现离线数据支持。

当时我们团队的规模也是我们做这个决定的重要因素。对于大规模的开发团队来说，Relay 会更加合适。但 F8 app 仅由一个人开发，另外还有人做些设计支持。这将会很大程度上影响采用哪种数据访问方式。

那么采用 [GraphQL](http://graphql.org/) 和 [Relay](https://facebook.github.io/relay/) 如何呢？虽然他们和 React Native 结合得很好，但是 [那个时候](https://github.com/facebook/relay/blob/master/meta/roadmaps/2016-H1.md) Relay 没有内置的离线数据支持。Parse 对 GraphQL 也没有现成的支持。如果要使用他们，我们需要开发 GraphQL-Parse 的 API，同时还需要实现 Relay 的离线数据存储。

另外，GraphQL 服务器的配置对于一个人在很短的开发时间内来说，还是相对复杂了。考虑到 app 已经在开发，想尽快提交应用市场，我们想要最简单有最快速的方案，那我们还有什么选择呢？

考虑到以上这些因素，当时对于这个 app 来说，[Redux](https://github.com/rackt/redux) 是最佳的选择。 Redux 对 [Flux 架构](https://facebook.github.io/flux/) 做了简单的实现，对数据如何存储和缓存，也提供了更多的控制，这本质上就使得 app 可以和 Parse Cloud 单向同步。

对于 app 的 store，Redux 提供了功能性和应用行的最佳平衡。app 发布之后，我们重新回顾了这个 app，在部分功能上，我们用上了 Relay 和 GraphQL。这在 [Relay 章节](tutorials/building-the-f8-app/relay/) 会有介绍。

### 我们开发所涉及的技术栈

我们使用 React Native 作为 app 框架，Redux 作为数据层，因此我们需要选用一些支持性的技术和工具：


* 开源的 [Parse Server](https://github.com/ParsePlatform/parse-server) 做数据存储 - 运行在 [Node.js](https://nodejs.org/en/) 上。
* [Flow](http://flowtype.org/) 用来做 React Native 的 JavaScript 输入错误检查，防止低级的输入错误。
* 使用 [Jest framework](http://facebook.github.io/jest/) 来做单元测试。
* 我们使用 [React Native Facebook SDK](https://github.com/facebook/react-native-fbsdk) 来做 iOS 和 Android 客户端的快速 Facebook 集成。
* 我们使用 Facebook 在 Mac 上的编辑器 [Nuclide](http://nuclide.io/) ，这个编辑器有 [内置的 React Native 支持](http://nuclide.io/docs/platforms/react-native/)。
* 用 git 来做版本管理，项目托管在 [GitHub](https://github.com/fbsamples/f8app)。

另外，我们还使用了其他的一些小的包，我们在后续章节会指出。

*如果你不了解 React 和 React Native，在开始阅读后续章节之前，我们推荐你看看 [React.js 项目本身的指南](http://facebook.github.io/react/docs/tutorial.html) - 尤其是了解 React 的 [模块化组件的概念](http://facebook.github.io/react/docs/thinking-in-react.html#step-1-break-the-ui-into-a-component-hierarchy) 以及 [JSX 语法](http://facebook.github.io/react/docs/jsx-in-depth.html)。*

*然后你还需要了解 [React Native](http://facebook.github.io/react-native/docs/tutorial.html#content) ，看看 React Native 是如何在手机应用上运行的。*
