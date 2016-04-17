---
pageid: 1-4-testing
series: buildingf8app
partlabel: 第四章
title: 测试
layout: docs
permalink: /tutorials/building-the-f8-app/testing/
intro: >
  探索如何使用 Nuclide，Flow，以及 Jest，以提高编写 React Native 时的代码质量。
---

> *这一系列的教程写于 F8 2016 app 开发期间，为的是用简单的文字来介绍 React Native 及其开源生态。教程于 2016 年 4 月 13 号发布到 makeitopen.com，内容翔实很有指导意义，[我](https://github.com/liaohuqiu) 看了之后花了周六日两天时间，翻译成中文，方便大家学习。* 

在传统的软件开发周期中，测试经常被当做是开发接近完成后，一个特别的阶段。对于新生的开源框架，这种看法似乎更接近于事实，因为这些项目的发布时，有时没有相关的测试技术。

幸好 Facebook 在开发 React Native 之初就持续地将测试技术放在心中。在这一部分，我们将会展示如何使用 [Nuclide](http://nuclide.io)，[Flow](http://flowtype.org) 和 [Jest](http://facebook.github.io/jest/) 提高你编写的 React Native 的代码质量的。

### Flow：使用类型检查以免写出糟糕的代码

[Flow](http://flowtype.org) 提供了对 JavaScript 的 [静态类型检查](http://flowtype.org/docs/about-flow.html#_)。Flow 允许以一种渐进的方式，逐步将它的特性运用到我们的代码中。当我们仅仅相对部分代码引入类型检查时，这点特别重要，否则我们就要重改整个 app 的代码了。

在 F8 app 中，我们决定从一开始就完全采用 Flow，在每一个必要的地方都加上 [类型注解](http://flowtype.org/docs/type-annotations.html#_)，然后 Flow 替我们完成这些检查。

比如，让我来看一个 [数据集成章节]({{ site.baseurl }}/tutorials/building-the-f8-app/data/) 提到的简单的 Action：

```js
/* from js/actions/login.js */

/*
 * @flow
 */

...

function skipLogin(): Action {
  return {
    type: 'SKIPPED_LOGIN',
  };
}
```

在第 13 行，我们加了一个 `@flow` 标志，让 [Flow 来检查这些代码](http://flowtype.org/docs/getting-started.html#_))。然后，我们用 Flow 的 [类型注解](http://flowtype.org/docs/type-annotations.html#_) 来标明 `skipLogin()` 的返回值必须是 `Action` 类型。但这个类型不是 React Native 或者 Redux 的内置类型，我们需要自己定义：

```js
/* from js/actions/types.js */

export type Action =
    { type: 'LOADED_ABOUT', list: Array<ParseObject> }
  | { type: 'LOADED_NOTIFICATIONS', list: Array<ParseObject> }
  | { type: 'LOADED_MAPS', list: Array<ParseObject> }
  | { type: 'LOADED_FRIENDS_SCHEDULES', list: Array<{ id: string; name: string; schedule: {[key: string]: boolean}; }> }
  | { type: 'LOADED_CONFIG', config: ParseObject }
  | { type: 'LOADED_SESSIONS', list: Array<ParseObject> }
  | { type: 'LOADED_SURVEYS', list: Array<Object> }
  | { type: 'SUBMITTED_SURVEY_ANSWERS', id: string; }
  | { type: 'LOGGED_IN', data: { id: string; name: string; sharedSchedule: ?boolean; } }
  | { type: 'RESTORED_SCHEDULE', list: Array<ParseObject> }
  | { type: 'SKIPPED_LOGIN' }
  | { type: 'LOGGED_OUT' }
  | { type: 'SESSION_ADDED', id: string }
  | { type: 'SESSION_REMOVED', id: string }
  | { type: 'SET_SHARING', enabled: boolean }
  | { type: 'APPLY_TOPICS_FILTER', topics: {[key: string]: boolean} }
  | { type: 'CLEAR_FILTER' }
  | { type: 'SWITCH_DAY', day: 1 | 2 }
  | { type: 'SWITCH_TAB', tab: 'schedule' | 'my-schedule' | 'map' | 'notifications' | 'info' }
  | { type: 'TURNED_ON_PUSH_NOTIFICATIONS' }
  | { type: 'REGISTERED_PUSH_NOTIFICATIONS' }
  | { type: 'SKIPPED_PUSH_NOTIFICATIONS' }
  | { type: 'RECEIVED_PUSH_NOTIFICATION', notification: Object }
  | { type: 'SEEN_ALL_NOTIFICATIONS' }
  | { type: 'RESET_NUXES' }
  ;

```

这里，我们创建了一些 [Flow 的类型别名](http://flowtype.org/docs/type-aliases.html#_)，`Action` 必须只能是这一些列不同的对象形式中的一种。

比如 `SKIPPED_LOGIN` 这个 Action 只能包含一个 type 标签；而 `LOADED_SURVEYS` Action 出了 type 还必须返回一个列表。

我们可以看到对应的 Action creator 确实也是这样做的：

```js
/* from js/actions/surveys.js */
async function loadSurveys(): Promise<Action> {
  const list = await Parse.Cloud.run('surveys');
  return {
    type: 'LOADED_SURVEYS',
    list,
  };
}
```

我们在 app 中用了许多的 Action，强类型检查让我们不至于犯下一些比如拼写错误这样的低级的错误。更为重要的是，数据格式错误时，我们也能及时发现。

对于 Reducer，我们也做了一样的强类型检查：

```js
/* from js/reducers/surveys.js */
function surveys(state: State = [], action: Action): State {
  if (action.type === 'LOADED_SURVEYS') {
    return action.list;
  }
  ...
  return state;
}
```

因为 `action` 参数指定为了 `Action` 类型，Reducer 函数必须使用一个有效的 `action.type`。当我们定义 `state` 树的部分结构时，我们也使用了类型别名去定义 `state` 的结构：

```js
/* from js/reducers/user.js */
export type State = {
  isLoggedIn: boolean;
  hasSkippedLogin: boolean;
  sharedSchedule: ?boolean;
  id: ?string;
  name: ?string;
};

const initialState = {
  isLoggedIn: false,
  hasSkippedLogin: false,
  sharedSchedule: null,
  id: null,
  name: null,
};

function user(state: State = initialState, action: Action): State {
  ...
}
```

在 [数据整合章节]({{ site.baseurl }}/tutorials/building-the-f8-app/data/)，我们看到了这个 `initialState` 对象，你现在将会看到，我们是如何让 `state` 树的部分去遵从这个 Flow 类型的，任何不满足类型要求的 `state`，都会有 Flow 类型检查错误。

Note that Flow checks are run at compile-time only, and the React Native packager 

请注意，Flow 的检查只是编译时的。 React Native packager [会自动将这些移除](https://github.com/facebook/react-native/blob/master/babel-preset/configs/main.js#L32)。这也就是说，使用 Flow 不会导致运行时的任何性能损失。

当然，目前每次我们想测试代码的时候，我们必须手动运行 [Flow 命令行](http://flowtype.org/docs/cli.html#_) 来进行类型检查。但我们也可以用 Nuclide 在 **编码的时候** 来完成这些验证。

### Nuclide：React Native 的 IDE

[Nuclide 的网站上](http://nuclide.io/docs/platforms/react-native/) 有对其提供的针对 React Native 量身定制的功能的全面介绍。我只想说，Nuclide 确实是开发 React Native 的一流 IDE。是为那些在 Facebook 写 React Native，以及使用 React Native 来开发 app 的人们开发的。

Flow 的集成是我们尤其感兴趣的，我们这里可以看一些 Reducer 中的代码：

```js
  if (action.type === 'SKIPPED_LOGIN') {
    return {
      isLoggedIn: false,
      hasSkippedLogin: true,
      sharedSchedule: null,
      id: null,
      name: null,
    };
  }
```

之前我们提到，我们会对返回类型进行强制检查。在 Nuclide 中，当错误发生时，我们可以实时地看到：

<video width="1174" height="1002" autoplay loop>
  <source src="static/videos/flow.mp4" type="video/mp4">
  Your browser does not support the HTML5 video tag.
</video>

在快速开发 app 的过程中，我们可能会不小心地，却又经常地遗漏掉 State 类型的某个部分，现在我们可以及时地得到反馈。

Nuclide 会执行所有的类型检查，我们在写代码的时候，我们就可以发现这些错误并及时更正，而不是等到开发接近完成时，这些错误才暴露出来。

这也许并不直观，但这确实提高了开发速度。要想发现没有写的，被遗漏的代码，相当困难。在 app 都接近开发完成时，面对一大堆代码，这变得更加麻烦。

### Jest: 单元测试

[Jest](http://facebook.github.io/jest/) 是针对 JavaScript 的一套测试框架，用它来做 [React Native 的测试](http://facebook.github.io/react-native/docs/testing.html#content) 效果不错。

我们用这些单元测试来做回归测试，确保已经完成的功能性代码，不会引入新的 bug。

比如我们想用一个单元测试来保证 Reducer 能够正确地处理地图数据：

```js
jest.autoMockOff();

const Parse = require('parse');
const maps = require('../maps');

describe('maps reducer', () => {

  it('is empty by default', () => {
    expect(maps(undefined, {})).toEqual([]);
  });

  it('populates maps from server', () => {
    let list = [
      new Parse.Object({mapTitle: 'Day 1', mapImage: new Parse.File('1.png')}),
      new Parse.Object({mapTitle: 'Day 2', mapImage: new Parse.File('2.png')}),
    ];

    expect(
      maps([], {type: 'LOADED_MAPS', list})
    ).toEqual([{
      id: jasmine.any(String),
      title: 'Day 1',
      url: '1.png',
    }, {
      id: jasmine.any(String),
      title: 'Day 2',
      url: '2.png',
    }]);
  });

});
```

Jest 简单易读，但我们还是会做进一步的分解。

在第 4 行，我们引入了地图的 Reducer 函数 `js/reducers/maps.js`，以便我们可以在单元测试中用到。因为 Reducer 函数是 [pure functions](https://en.wikipedia.org/wiki/Pure_function)，所以很容易进行测试。

第 8 行的测试代码，确保返回一个空数组。因为在 `js/reducers/maps.js` 我们没有定义初始的 `state`，所以返回一个空数组就好了。

第 12 行，我们用假数据来替换 API 返回的数据来进行测试，确保数据可以被 Reducer 转换成正确的 `state` 结构。这可避免因为 API 的网络原因导致测试失败。

现在我们已经把单元测试变为开发流程中的一部分了，比如我们会在 Git 提交前做检查。这样我们就可以保证我们的代码变更不会使得 app 默默地就挂掉了。

只有 Redux Reducer 改变 `state` 树对于没有 bug 被引入是至关重要的。即使有，关于 `state` 改变的 bug 也不会导致功能不可用，仅仅是错误的数据被发送到 Parse Server 罢了。

Reducer 的 pure function 性质，使得他们成为回归测试的理想对象，因为每次我们都可以预知他们的返回值。

### 调试

如果你想定位或者修复一个 bug，如果手边有些工具可用的话，那就最好了。上一章节，我们提到 [如果调整 UI 元素]({{ site.baseurl }}/tutorials/building-the-f8-app/design/#the-design-iteration-cycle)，那么对于数据呢？

我们使用 [Chrome 的开发者工具](https://facebook.github.io/react-native/docs/debugging.html#chrome-developer-tools) 通过 [Nuclide](http://nuclide.io/docs/features/debugger/) 和 [Redux Logger](http://fcomb.github.io/redux-logger/) middleware 在控制台查看信息。

![Redux Logger Middleware in action with additional console context](static/images/redux_logger.png)

在 `configureStore` 中，你可以看到我们是如果设置的：

```js
/* from js/store/configureStore.js */
var createLogger = require('redux-logger');
...
var isDebuggingInChrome = __DEV__ && !!window.navigator.userAgent;
var logger = createLogger({
  predicate: (getState, action) => isDebuggingInChrome,
  collapsed: true,
  duration: true,
});

var createF8Store = applyMiddleware(thunk, promise, array, logger)(createStore);

function configureStore(onComplete: ?() => void) {
  const store = autoRehydrate()(createF8Store)(reducers);
  persistStore(store, {storage: AsyncStorage}, onComplete);
  if (isDebuggingInChrome) {
    window.store = store;
  }
  return store;
}
```

第 5 行，我们使用 [一些选项](https://github.com/fcomb/redux-logger#options-1) 创建了 Logger middleware。 在第 10 行，我们通过 Redux 的 [`applyMiddleware()` 函数](http://redux.js.org/docs/api/applyMiddleware.html) 作用到了 Store 上。这样控制台就会有日志输出了。

在第 4 行，我们使用一个 [全局变量](http://www.w3schools.com/js/js_scope.asp) `__DEV__` 来控制是否开启一些调试功能。比如创建 Logger middleware，并开启[`predicate（断言）` 选项](https://github.com/fcomb/redux-logger#predicate--getstate-function-action-object--boolean)) 来记录日志；比如 17 行，将 Store 拷贝到 [Window object](http://www.w3schools.com/jsref/obj_window.asp)，这样就可以在控制台方便地直接查看 Store 对象了。


