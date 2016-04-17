---
pageid: 1-A2-relay
series: buildingf8app
partlabel: 附录 2
type: appendix
title: 使用 Relay 和 GraphQL
layout: docs
permalink: /tutorials/building-the-f8-app/relay/
---

在开始筹备这个 app，[考虑数据层的选择时](tutorials/building-the-f8-app/planning/#data-access-with-react-native)，我们对比了 [Redux](https://github.com/rackt/redux) 和 Facebook 的另一个开源框架 [Relay](https://facebook.github.io/relay/)，最后选择了前者。

那时因为 Redux 的实现较为简单，和 Parse 的数据的集成也容易一些。现在，我们项目上线了发布了，我们想再回顾一下当时的选择，看看 Relay 能在我们的 app 中怎么用。

### 逐步地演变

传统的 app 开发，变换一个数据层的选择，通常会导致整个 app 的改动，

React Native 不太一样，我们可以保持现有的数据层实现：Redux， Parse，以及相关的绑定。然后在某个独立的 View 中，引入新的数据层。我们可以仅仅改动一部分，其他部分保持不动。

持续开发，持续改进，大大地减少维护和更新的开销，这些好处是值得大书特书的。

现在我们看看使用 Relay + GraphQL 做数据模型和 Redux 有什么区别。

### Relay 和 GraphQL

[Relay](https://facebook.github.io/relay/) 是 app 中的一个数据框架，[GraphQL](http://graphql.org/) 是 Relay 用来做数据表示的查询语言。GraphQL 同时还有一个 npm 包，可在服务器上运行，提供可以和 Relay 交互的数据源（关于 GraphQL 的具体设置细节教程，我们后续会更新，敬请期待）。

Relay 不是从 Flux 架构分化来了，它只和 GraphQL 有关。这也就是说，这和 Redux 的模型有着巨大的区别。我们在 [数据集成章节](tutorials/building-the-f8-app/data/) 提到的 Store/Reducer/Component 的交互，在 Relay 中就不存在了。用 Relay 做数据集成时，需要用另外一种方式，我们之前要做的很多工作都可以不用了。

在 Relay + GraphQL 的模型中，每个组件指定自己需要的数据。Relay 调用数据，数据更新时，提供给组件最新的数据，并在客户端做缓存。app 需要更新数据时，在 Action 中创建一个 [GraphQL 变更](https://facebook.github.io/relay/docs/guides-mutations.html#content)，这和 Redux 类似。

### F8 App 中的例子

因为在 React Native 中，我们可以逐步地更改我们 app 中的一些小地方，为了验证可以将 Redux 更换成 Relay 这个概念，我们选择了 F8 app 中的 Info View，如下：

![Info view of F8 iOS app](static/images/info_view.png)

这个 View 和 其他部分几乎是完全没关系的，有大量的非交互的内容，是做尝试的不二之选。

这个 View 包含了一个非常简单的 `<InfoList>`，如下：

```js
/* from js/tabs/info/F8InfoView.js */
function InfoList({viewer: {config, faqs, pages}, ...props}) {
  return (
    <PureListView
      renderEmptyList={() => (
        <View>
          <WiFiDetails
            network={config.wifiNetwork}
            password={config.wifiPassword}
          />
          <CommonQuestions faqs={faqs} />
          <LinksList title="Facebook pages" links={pages} />
          <LinksList title="Facebook policies" links={POLICIES_LINKS} />
        </View>
      )}
      {...props}
    />
  );
}
```

这个是一个非常普通的布局，里面嵌套了另外一些布局，看起来普普通通，但是其中的 `props` 以及其他参数是哪来的呢？在同一个文件中，我们有一段 GraphQL 相关的代码：

```js
/* from js/tabs/info/F8InfoView.js */
InfoList = Relay.createContainer(InfoList, {
  fragments: {
    viewer: () => Relay.QL`
      fragment on User {
        config {
          wifiNetwork
          wifiPassword
        }
        faqs {
          question
          answer
        }
        pages {
          title
          url
          logo
        }
      }
    `,
  },
});
```

我们用一段 GraphQL 来定义 `<InfoList>` 组件显示时需要的数据，这个和 GraphQL server 上的 GraphQL 对象结构是一致的。

```js
/* from server/schema/schema.js */
var F8UserType = new GraphQLObjectType({
  name: 'User',
  description: 'A person who uses our app',
  fields: () => ({
    id: globalIdField('User'),
    name: {
      type: GraphQLString,
    },
    ...
    faqs: {
      type: new GraphQLList(F8FAQType),
      resolve: () => new Parse.Query(FAQ).find(),
    },
    pages: {
      type: new GraphQLList(F8PageType),
      resolve: () => new Parse.Query(Page).find(),
    },
    config: {
      type: F8ConfigType,
      resolve: () => Parse.Config.get(),
    }
  }),
  ...
});
```

你可以看到，GraphQL server 是如何取数据的，Relay 根据定义去取这些数据。当数据准备完成时，使用做为 `viewer` 参数传入到 `<InfoList>` 中，同样使用 [destructuring assignments（解构赋值）](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment) 创建 `config`,`faqs`,`pages` 等变量，在组件内使用。

幸亏 Relay 内置的逻辑，我们不需要考虑到数据订阅，数据缓存之类的事情，我们只需要告诉 Relay 我们的组件需要什么样的数据，然后使用 React 标准的方式来写组件就好了。GraphQL server 的设置好了之后，我们所要做的，也确实就这些了。

在我们这个 view 中，我们没有数据变更，如果你想了解关于数据变更方便的细节，你可以阅读 [Relay 的数据变更相关的文档](https://facebook.github.io/relay/docs/guides-mutations.html#content)。
