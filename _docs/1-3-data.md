---
pageid: 1-3-data
series: buildingf8app
partlabel: 第三章
title: 数据集成
layout: docs
permalink: /tutorials/building-the-f8-app/data/
intro: >
  我们将说明 React Native 中的数据流，Redux 在 F8 app 中是如何工作的，以及连接 Parse Server 的简单流程。
---

> *这一系列的教程写于 F8 2016 app 开发期间，为的是用简单的文字来介绍 React Native 及其开源生态。教程于 2016 年 4 月 13 号发布到 makeitopen.com，内容翔实很有指导意义，[我](https://github.com/liaohuqiu) 看了之后花了周六日两天时间，翻译成中文，方便大家学习。* 

使用 [React](http://facebook.github.io/react/) 或 [React Native](http://facebook.github.io/react-native/) 开发 app，我们不需要太关注数据来自于哪，我们可以把更多的精力放在业务逻辑和 UI 上。

在 [第一部分](tutorials/building-the-f8-app/planning/)，我们提到我们是如何调整 Parse Server 使之能够作为数据后端的。我们也提到，将会在 app 中使用 Redux 处理数据。在这一章节，我们会阐释 Redux 在 React Native app 中是如何工作的，以及连接到 Parse Server 的简单流程。


在我们开始讨论 Redux 之前，让我们看看 React 的数据整合是如何进化到创建 Redux 本身的。

### 首先，React app 如何和数据交互？

React 经常被做为 MVC 中的 `View` 提到。但 React 比那个微妙多了，它把 MVC 看待成了其他东西。 

让我们看看 MVC 的架构：

* Model 是数据
* View 是数据在 app 中的表现
* Controller 处理数据交互逻辑

在 React 中，我们使用多个组件组成一个 View，同时每个组件又可以处理 Controller 才具有的处理逻辑的能力。比如：

```js
class Example extends React.Component {
    render() {
        // 使用现有数据呈现 View，有可能是一个窗体
        // 窗体将数据提交给 handleSubmit 函数
    }

    handleSubmit(e) {
        // 处理数据，controller 的逻辑
    }
};
```

每个 React 组件都有两种类型的数据，他们拥有不一样的角色：

* `props` 是在组件创建时就传入的数据，是不可变的。
* `state` 是组件可变的数据。

为了减少重复，建议 app 组件层级关系中最高层级的父组件（有时称为容器组件）拥有 `state` ，然后使用 `props` 向下传递给子组件。这样数据单向地从上游往下游流动，使得整个流程很快，呈现模块化。

如果你想了解更多关于这个建议背后的考虑和原因，你可以阅读 React 网站上的 [Thinking in React](https://facebook.github.io/react/docs/thinking-in-react.html) 章节相关的内容。

### Store 和 State

为了进一步阐述 React 中数据使用技术，Facebook 引入了 [Flux 架构](https://facebook.github.io/flux/docs/overview.html)。这个架构更像是一个你应用中需要实现的模式，而非一个真正能用的框架。

我们没在我们的应用中使用 [Flux](https://github.com/facebook/flux)，我们所使用的框架： Redux，是从 Flux 架构分化而来的。

Flux 扩充了 React 中的数据流程。它引入了 Store 这个概念：Store 是包含 app 的 `state` 的对象。 Flux 还引入实时修改 `state` 的新的流程。

* 每个 **Store** 都有一个回调函数，这个回调注册到 Dispatcher。
* **View**（就是 React 中的组件）可以触发 **Action**，它是一个包含一系列数据的一个对象，比如，是一个 form 输入的数据。同时，它还包含有一个 **action type** ，这是一个常量，用来描述正在执行的 Action。
* Action 被发送到 **Dispatcher**。
* Dispatcher 将这个 Action 传播到所有注册的 Store 的回调函数上。
* 如果 Store 对 Action 感兴趣（根据 Action 的类型判断），Store 会更新自身，以及其所包含的 `state`，更新之后，发送一个事件。
* 我们前面提到的容器组件，称为 **Controller View** 会监听这些事件，当收到事件时，从 Store 获取新数据。
* 取到数据之后通过调用 [`setState()`](https://facebook.github.io/react/docs/component-api.html#setstate) 使得其所包含的所有子组件重新 `render()`。


你可以看到 Flux 是如何在 React 中强制数据单向流动的，这使得 React 的数据部分更加优雅和结构化。

不过我们没有使用 Flux，如果你想了解更多，你可以访问 [Flux 教程](https://facebook.github.io/flux/docs/todo-list.html)。

那么我们真正使用的框架，Redux 和 Flux 又是怎样一个关系呢？

<h3 id='flux-to-redux'>从 Flux 到 Redux</h3>

Redux 是一个实现了 Flux 架构，但又从 Flux 中剥离的框架。[react-redux 包提供的官方的数据绑定实现](https://github.com/reactjs/react-redux) 使得和 React 的集成变得很简单。

Redux 中没有 Dispatcher，并且对于整个 app 的 `state`，只有一个 Store。

那么 Redux 中的数据流到底是怎样的？详细的部分我们下面后阐述，现在让我们看看大致的流程：

* React 可以触发 Action，比如按钮点击按钮。
* Action 是对象，包含一个类型以及相关的数据，通过 Store 的 `dispatch()` 函数发送到 Store。
* Store 接收 Action 的数据并将其连同当前的 `state` 树（[`state` 树](https://egghead.io/lessons/javascript-redux-the-single-immutable-state-tree) 是包含所有 `state` 的一种特殊的数据结构，是一个单一的对象）发给 **Reducer**。
* **Reducer** 是一个 [pure function](http://redux.js.org/docs/basics/Reducers.html#handling-actions)，它接收一个之前的 `state` 和一个 Action；并基于此 Action 将会产生的影响，返回一个新的 `state`。一个 app 可以包含一个 Reducer，但大部分的 app 最后会包含多个，每个处理 `state` 中不同的部分，[下文](#reducers) 会提到。
* Store 接收到新的 `state`，并替换当前的。
* 当 `state` 变化时，Store 触发 [事件](http://redux.js.org/docs/api/Store.html#subscribe)。
* 任何 [订阅了事件](http://redux.js.org/docs/api/Store.html#subscribe) 的组件 [从 Store 中提取新的 `state`](http://redux.js.org/docs/api/Store.html#getState)。
* 组件使用新的 `state` 进行更新。

简单起见，这个流程可用下图表示：

![Redux data flow as described above]({{ site.baseurl }}/static/images/redux_flowchart.png)

你可以看到数据随着一个很清晰的单项路径流动，没有重叠，没有反方向的数据流。这图也展示了 app 的每一部分可以多么清晰地分开：

* Store 只关心所只有的 `state`；
* View 中的组件，只关心显示数据和触发 Action；
* Action 只关注 `state` 中的某些数据发生变化了，并包含了这些数据；
* Reducer 只关注旧的状态并将 Action 放入到 `state` 中。

一切都是模块化的，非常优雅。当阅读这样的代码的时候，表意非常明显，很容易理解。

和 Flux 相比，这有另外的一些好处：

* Action 是唯一一个可以导致 `state` 变化的途径，这也将这流程从 UI 组件中归集起来。另外，因为 Action 都是由 Reducer 妥当排序的，这也防止了条件竞争。
* `state` 大体上是不可变的。创建一系列的 `state` ，每个 `state` 代表每个个体的变化。这给我们了一个 app 中清晰并且易于追溯的 `state` 历史。

### 整体

我们已经抽象地谈到了数据流，现在让我们看看我们的 app 中是如何运用的，以及我们从中学到的什么。

#### Store

[Redux 的文档](http://redux.js.org/docs/basics/Store.html) 很好地解释了如何创建一个简单的 Store，我们假设对文档所说的这些基础部分，你已经游刃有余。

##### Store 的离线同步

前面我们提到，为了我们的 app 能在没有网络或者网络条件不好的情况下工作，我们需要离线的本地存储。幸好，我们使用 Redux，有一个非常简单 module 可以用在我们的 app 中：[Redux 的持久化](https://www.npmjs.com/package/redux-persist)。

配合我们的 Store，我们也用到了 [**Middleware**](http://redux.js.org/docs/advanced/Middleware.html)，这点我们在后面的 [测试相关的章节][testing] 会更具体地提到。通过 middleware，我们可以在 Action 被分发到 Reducer 之前，加入一些处理逻辑，比如：日志，crash 报告，调用异步 API，等等。

我们实际的代码如下；

```js
/* js/store/configureStore.js */
var createF8Store = applyMiddleware(...)(createStore);

function configureStore(onComplete: ?() => void) {
  const store = autoRehydrate()(createF8Store)(reducers);
  persistStore(store, {storage: AsyncStorage}, onComplete);
  ...
  return store;
}
```

上面的代码也许有些难懂，我们展开看看：

```js
/* js/store/configureStore.js */

var middlewareWrapper = applyMiddleware(...);
var createF8Store = middlewareWrapper(createStore(reducers));

function configureStore(onComplete: ?() => void) {
  const rehydrator = autoRehydrate();
  const store = rehydrator(createF8Store);
  persistStore(store, {storage: AsyncStorage}, onComplete);
  ...
  return store;
}
```

1. 第 3 行，`applyMiddleware()` 返回一个函数，这个函数我们将会作用于我们的 Store 对象。关于 `applyMiddleware()`，你可以通过
[Redux `applyMiddleware` 相关的文档](http://redux.js.org/docs/api/applyMiddleware.html)) 了解更多的信息。

2. 第 4 行，`reducers` 是 app 中的所有 Reducer，[`createStore()`](http://redux.js.org/docs/api/createStore.html) 接收这个参数进行处理，返回一个 Store 对象，这个对象交给 Middleware 处理，返回值我们记为 `createF8Store`。

3. 和 `applyMiddleware()` 相似， [Persist 的 `autoRehydrate()`](https://github.com/rt2zz/redux-persist#autorehydrate) 它将返回一个可作用于 Store 的函数，我们记为：`rehydrator`。`rehydrator` 作用于我们的 `Store`，可将之前存储于本地的 Store 对象用当前的 Store 的 `state` 进行自动更新。

4. 第 9 行，[Persist 包中的 `persistStore()` 函数](https://github.com/rt2zz/redux-persist#persiststorestore-config-callback) 处理 Store 保存到本地存储相关的逻辑。这其中，我们配置使用了 [React Native 内置的异步存储系统](https://facebook.github.io/react-native/docs/asyncstorage.html))。`autoRehydrate()` 和 `persistStore()` 实际就是我们用来实现离线数据同步的所有代码了。

现在，没有网络连接的时候，之前的 Store 对象还在本地存储中，从用户的角度来看，app 还是可用的。

目前位置，我们的 Store 基本上算处理完了。如果你想了解更多，你可以看看： [Redux 持久化实现的技术细节](https://www.npmjs.com/package/redux-persist#basic-usage)。

### Reducer

在 [前面关于 Redux 的阐释中](#flux-to-redux)，我们提到 Redux 引入了一个 Reducer 对象。一个 app 可以有多个 Reducer，每个关注 `state` 的不同部分。比如：在一个带评论的 app 中，有一个 Reducer 处理登录相关的状态，另外一个 Reducer 处理评论数据。

在 F8 app 中，所有的 Reducer 的源码都在 `js/reducers/` 目录下，这里我们简要地看看 `user.js`：

```js
/* js/reducers/user.js */
import type {Action} from '../actions/types';

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
  if (action.type === 'LOGGED_IN') {
    let {id, name, sharedSchedule} = action.data;
    if (sharedSchedule === undefined) {
      sharedSchedule = null;
    }
    return {
      isLoggedIn: true,
      hasSkippedLogin: false,
      sharedSchedule,
      id,
      name,
    };
  }
  if (action.type === 'SKIPPED_LOGIN') {
    return {
      isLoggedIn: false,
      hasSkippedLogin: true,
      sharedSchedule: null,
      id: null,
      name: null,
    };
  }
  if (action.type === 'LOGGED_OUT') {
    return initialState;
  }
  if (action.type === 'SET_SHARING') {
    return {
      ...state,
      sharedSchedule: action.enabled,
    };
  }
  if (action.type === 'RESET_NUXES') {
    return {...state, sharedSchedule: null};
  }
  return state;
}

module.exports = user;
```

正如你所见，这个 Reducer 处理 login/logout 以及用户具体选项改变相关的逻辑。下面我们一部分一部分地来看。

> 注意， 22 行我们使用了 ES2015  [destructuring assignment](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment) 把 `action.data` 中对应 key 为左边变量名的值，赋值给了左边的变量。

##### 1. 初始状态

最开始，我们在第 12 行，依照 [Flow 的类型别名规范](http://flowtype.org/docs/type-aliases.html) （[测试相关的章节][testing] 详谈），我们定义了初始状态。 `initialState` 定义的值，会在 app 一启动就加载。

##### 2. Reducer 函数

从 20 到 56 行是 Reducer 的具体实现部分，实际也是相对简单的。函数接受 `state` 和 Action 作为参数，`state` 的默认值是 `initialState`。根据 Action 具体的 `type`，改变 `state`，返回一个新的 `state`。

以 43 行的 `LOGGED_OUT` 这个 Action 为例，我们把 `state` 重置为 `initialState`；而在 21 行，对于 `LOGGED_IN`，使用了 `action.data` 中 `id, name, sharedSchedule` 等用户输入的值，构造了一个新的 `state` 并返回。

现在让我们看看 46 行的 `SET_SHARING`，这里有一个 `...state`。  这个语法糖其实就是 [`Object.assign()`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Object/assign) ，但更紧凑和可读一些。这是 [React 所包含的 Javascript 语法转换](https://facebook.github.io/react-native/docs/javascript-environment.html#javascript-syntax-transformers) 的一部分，称为 [Object Spread Operator（对象扩散操作符）](http://redux.js.org/docs/recipes/UsingObjectSpreadOperator.html))。这里将 `state` 做了一个复制，然后更新 `sharedSchedule` 的值并返回。

你可以看到 Reducer 的结构是多么简单，可读性很强。定义个 `initialState`，提供一个函数，处理 `state` 和 Action，返回一个新的 `state`，就是这样。

在这个函数中，我们不做其他的任何事情，Reducer 的一个至为重要的规则就是，函数应该是 `pure function`，即对于一样的参数，一定会有同样的返回值。

[Redux 的文档](http://redux.js.org/docs/basics/Reducers.html#handling-actions) 的原话是：

>"Remember that the reducer must be pure. Given the same arguments, it should calculate the next state and return it. No surprises. No side effects. No API calls. No mutations. Just a calculation."

另外，我们再看看 `js/reducers/notifications.js`，这里也有对 `LOGGED_OUT` 这个 Action 进行的处理。每个 Reducer 在 Action 被派发后都会被调用，每个 Reducer 只处理自己关心的 `state` 树种的那部分 `state`。

### Actions

现在让我们看看 login 相关的 Action：

```js
/* from js/actions/login.js */
function skipLogin(): Action {
  return {
    type: 'SKIPPED_LOGIN',
  };
}
```

这个是一个非常简单的 **Action creator**，该函数返回的结果是一个真正的 Action。每个 Action 可以只包含一个 `type`，Reducer 根据这个 `type` 更新 `state`。

和 `type` 一起，Action 还可以携带一些数据，如：

```js
/* from js/actions/filter.js */
function applyTopicsFilter(topics): Action {
  return {
    type: 'APPLY_TOPICS_FILTER',
    topics: topics,
  };
}
```

这里 **Action creator**，接收一个参数并将其插入到 Action 中。

有些 Action creator 会先执行一些逻辑，他们会返回一个函数而不是一个 Action：

```js
/* from js/actions/login.js */
function logOut(): ThunkAction {
  return (dispatch) => {
    Parse.User.logOut();
    FacebookSDK.logout();
    ...

    return dispatch({
      type: 'LOGGED_OUT',
    });
  };
}
```

在这个例子中，我们使用了一个自定义的 ThunkAction（[Redux 建议这样做以减少重复定义](http://redux.js.org/docs/recipes/ReducingBoilerplate.html)），这个 Action creator 返回的函数，执行了 logout 相关的逻辑，然后分发了一个 Action。

注意：这里我们使用了 `=>` [Arrow function syntax（箭头函数语法）](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Functions/Arrow_functions)。

##### 异步 Action

如果我们和 API 交互，我们需要异步的 Action creators。. Redux 本身有一个 [相当复杂的异步方式](http://redux.js.org/docs/advanced/AsyncActions.html)。不过我们使用的是 React Native，我们可以使用 [ES7 的 `await`](https://facebook.github.io/react-native/docs/javascript-environment.html#javascript-syntax-transformers)，这会使得整个流程大大简化：

```js
/* from js/actions/config.js */
async function loadConfig(): Promise<Action> {
  const config = await Parse.Config.get();
  return {
    type: 'LOADED_CONFIG',
    config,
  };
}
```

这里，我们调用了一个 API，获取一些配置数据。任何网络 API 的调用都是一个耗时的操作。Action creator 等待结果返回，然后再返回一个 Action。这过程没有阻塞 JavaScript 线程。

这样的异步调用的好处就是，我们在等待 `Parse.Config` 返回的时候，其他异步的操作也可以同时进行。我们可以执行一系列的并发操作，而不必相互等待，以提高效率。

### 组件绑定

现在我们在我们 app 的 `setup()` 函数中把 Redux 的逻辑和 React 连接起来：

```js
/* from js/setup.js */
function setup(): React.Component {
  // ... 其他的设置逻辑

  class Root extends React.Component {
    constructor() {
      super();
      this.state = {
        store: configureStore(),
      };
    }
    render() {
      return (
        <Provider store={this.state.store}>
          <F8App />
        </Provider>
      );
    }
  }

  return Root;
}
```

我们使用官方的 [React 和 Redux 之间的绑定](https://github.com/reactjs/react-redux)，所以在第 18 行，我们可以使用内置的 [`<Provider>` 组件](http://redux.js.org/docs/basics/UsageWithReact.html#passing-the-store)。 Provider 可以把我们创建的 Store 和任意组件连接。

```js
/* from js/F8App.js */
var F8App = React.createClass({
  // 省略部分代码
})

function select(state) {
  return {
    notifications: state.notifications,
    isLoggedIn: state.user.isLoggedIn || state.user.hasSkippedLogin,
  };
}

module.exports = connect(select)(F8App);
```

上面的代码就是我们 app 的根容器组件 `<F8App>`。

在 13 行，我们使用了 React-Redux [`connect()` 函数](https://github.com/reactjs/react-redux/blob/master/docs/api.md#connectmapstatetoprops-mapdispatchtoprops-mergeprops-options)。`connect()` 函数的参数列表中的第一个参数名为 [`mapStateToProps`](https://github.com/reactjs/react-redux/blob/master/docs/api.md#arguments) 这个参数是一个函数，当 Store 更新时，这个函数就会被调用。

这里，我们传入的是第 6 行的 `select` 函数。这个函数传入的参数是 Store 中的 `state`，我们从这个 `state` 中提取数据，返回的数据会传入大到 `<F8App>` 中去。在这里，我们提取了通知消息相关的数据和登录状态相关的数据。

当 Store 中的数据变化时，`connect()` 函数被调用，我们提取的数据会被传入到 `<F8App>` 组件中。

```js
/* from js/F8App.js */
var F8App = React.createClass({
  ...
  componentDidMount: function() {
    // 省略代码
    if (this.props.notifications.enabled && !this.props.notifications.registered) {
        // 省略代码
    }
    // 省略代码
  },
  // 省略代码
})
```

现在，`<F8App>` 中第 6 行，我们可以得到最新的数据了，另外数据更新时，`render()` 方法也会被重新调用。下面我们看看如何在组件内触发一个 Action。

##### 组件内触发 Action

为了看清 Action 和组件是如何连接起来的，我们可以看看 `<GeneralScheduleView>` ，组件看起来像这样，可以切换 DAY 1， DAY 2，DAY 3。

![Screenshot of segmented controls]({{ site.baseurl }}/static/images/iOS vs Android Segmented Controls@3x.png)

主要代码如下：

```js
/* from js/tabs/schedule/GeneralScheduleView.js */
var {switchDay} = require('../../actions');

class GeneralScheduleView extends React.Component {
  props: Props;

  constructor(props) {
    super(props);
    this.renderStickyHeader = this.renderStickyHeader.bind(this);
    ...
    this.switchDay = this.switchDay.bind(this);
  }

  render() {
    return (
      <ListContainer
        title="Schedule"
        backgroundImage={require('./img/schedule-background.png')}
        backgroundShift={this.props.day - 1}
        backgroundColor={'#5597B8'}
        data={this.props.data}
        renderStickyHeader={this.renderStickyHeader}
        ...
      />
    );
  }

  ...

  renderStickyHeader() {
    return (
      <View>
        <F8SegmentedControl
          values={['Day 1', 'Day 2']}
          selectedIndex={this.props.day - 1}
          selectionColor="#51CDDA"
          onChange={this.switchDay}
        />
        ...
      </View>
    );
  }

  ...

  switchDay(page) {
    this.props.switchDay(page + 1);
  }
}

function select(state) {
  return {
    day: state.navigation.day,
    filter: state.filter,
    sessions: data(state),
  };
}

function mapDispatchToProps(dispatch) {
  return {
    switchDay: (day) => dispatch(switchDay(day)),
  };
}

module.exports = connect(select, mapDispatchToProps)(GeneralScheduleView);
```

为了说明问题，我们省略了部分细节。

在第 2 行，我们引入了 `switchDay()` 函数，这个函数实际在 `js/actions/navigation.js` 中：

```js
/* from js/actions/navigation.js */
  switchDay: (day): Action => ({
    type: 'SWITCH_DAY',
    day,
  })
```

这个 Action creator 简单返回一个 Action 对象，带上了 `day` 这个变量。

在 65 行，我们同样使用了 `connect()`，不过稍有不同的是，第二个参数，我们传入了一个函数 [`mapDispatchToProps`](https://github.com/reactjs/react-redux/blob/master/docs/api.md#arguments)。

当点击 "DAY 1" 的时候，`renderStickyHeader()` 中的 `onChange()` 会被触发，第 46 行的 `switchDay` 被调用，`props` 的 `switchDay` 被 `mapDispatchToProps` 映射到了 61 行的函数上。这个函数分发第 2 行引入的 `switchDay()` 返回的 Action。

在 Reducer 中，将修改过后的时间值 `day` 更新到 `state` 中：

```js
/* from js/reducers/navigation.js */
  if (action.type === 'SWITCH_DAY') {
    return {...state, day: action.day};
  }
```

因为 `<GeneralScheduleView>` 订阅了 `state`，一旦 `state` 发生变化，组件就更新了。

### Parse Server

教程直接到目前为止，相信你已经得到不少新的信息了。现在，再让我们来看看 React Native 和 Parse Server 是如何连接的：

```js
  Parse.initialize(
    'PARSE_APP_ID',
  );
  Parse.serverURL = 'http://exampleparseserver.com:1337/parse'
```

是的，由于我们使用了 [Parse + React](https://github.com/ParsePlatform/ParseReact) SDK （在 `parse/react-native` 包中），整个过程就是这么简单。

##### Parse 和 Action

我们在各个 Action 中要做很多查询，这些 Action creator 几乎是一样的，为了减少重复代码，我们创建了一个 base Action creator：

```js
/* from js/actions/parse.js */
function loadParseQuery(type: string, query: Parse.Query): ThunkAction {
  return (dispatch) => {
    return query.find({
      success: (list) => dispatch(({type, list}: any)),
      error: logError,
    });
  };
}
```

基于这个 base Action creator，我们创建具体的 Action creator：

```js
  loadMaps: (): ThunkAction =>
    loadParseQuery('LOADED_MAPS', new Parse.Query(Maps)),
```

`loadMaps()` 成了一个 Action creator，这个 creator 简单地运行一个 Parse Query，加载完所有的地图数据之后将数据分发。

在 `js/F8App.js` 中的 `componentDidMount()` 中你可以看到 app 中的所有 Parse 和 Action。这也就是说，在应用第一次运行的时候所有 Parse 的数据就加载进来了。

##### Parse 和 Reducer

我们已经缩减了 Action 的代码冗余，现在我们也想缩减 Reducer 的代码冗余。这些 Reducer 会接收一系列来自于 Parse API 的数据，并把这些数据更新到 `state` 树中。我们也创建一个 base Reducer：

```js
/* from js/reducers/createParseReducer.js */
function createParseReducer<T>(
  type: string,
  convert: Convert<T>
): Reducer<T> {
  return function(state: ?Array<T>, action: Action): Array<T> {
    if (action.type === type) {
      // Flow can't guarantee {type, list} is a valid action
      return (action: any).list.map(convert);
    }
    return state || [];
  };
}
```

这是一个很简单的 Reducer，当然，其中糅杂了很多 Flow 的类型注解，我们看看基于这个 base Reducer 派生出来的具体的 Reducer 是什么样的：

```js
/* from js/reducers/faqs.js */
const createParseReducer = require('./createParseReducer');

export type FAQ = {
  id: string;
  question: string;
  answer: string;
};

function fromParseObject(map: Object): FAQ {
  return {
    id: map.id,
    question: map.get('question'),
    answer: map.get('answer'),
  };
}

module.exports = createParseReducer('LOADED_FAQS', fromParseObject);
```

我们没有简单地重复 `createParseReducer` 中的代码，我们把一个映射 API 数据到 `state` 的函数传入到了其中，达到了缩减代码的目的。

好了，现在我们有一个结构良好，数据流简单易懂，和 Parse Server 串联起来了的 app 了。并且，我们还有了离线数据存储的支持。

[testing]:          {{ site.baseurl }}/tutorials/building-the-f8-app/testing/
