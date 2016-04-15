---
pageid: 1-2-design
series: buildingf8app
partlabel: 第二章
title: 应用的多平台设计
layout: docs
permalink: /tutorials/building-the-f8-app/design/
intro: >
  我们将谈谈为什么 React Native 可以针对，或者甚至是应该针对，各个平台量身定制。而非各个平台上没有任何区分。
---

React Native 的一个亮点就是，他可以简单地创建可同时在 iOS 和 Android 平台上运行的应用，不必用这两个平台的原生语言重复大部分的业务逻辑。

尽管如此，React Native 的哲学其实是：“一次学习，随处可写”，而非：“一次编码，随处可运行”。这两者的细微差别在于，React Native 的 app 应该对各个平台做适配，而非让他们变得一样。

从 UI 的角度来看，各个平台有不同的视觉样式，UI 规范，或者从技术实现能力来看，应该从公共的 UI 基础入手，然后对两个平台做适配。

#### 开始之前

从这一章节开始，我们将会涉及到这个 app 的代码，所以你应该 [看看源码](https://github.com/fbsamples/f8app) 或者把源码 clone 到本地。 你也可以根据 [设置和运行 App 章节](tutorials/building-the-f8-app/local-setup/) 的内容在本地运行 App。不过这个章节中，你只需要看看源码即可。

### React Native 的思想

在你写任何 React Native 的代码之前，React Native 有一个非常重要的概念指导你如何看待 app 的每一个部分。那就是：“尽可能地复用代码”。

这也许看起来和 React Native 为各个平台做视觉适配冲突。视觉适配似乎是尝试为 iOS 和 Android 创建单独的组件。

其实不是的，这只要求用 React Native 写代码时，尽可能地共享代码。

当我们考量 React Native app 中的可视化组件时，关键在于使用平台抽象。工程师和设计师确定 app 中可复用的组件，比如：按钮，容器，列表的每一行，等等。并且，只在有需要的地方将这些组件做分化。

当然，有许多组件是比较复杂的，我们看看 F8 app 中的不同的组件。

### 多样化更小的组件

这是 F8 app 中的一个例子：

![iOS and Android Segmented Controls Comparison]({{ site.baseurl }}/static/images/iOS vs Android Segmented Controls@3x.png)

在 iOS 中，tab 控件是圆角；而在 Android 上，应该是下划线风格。在这两个平台上，控件的功能是一样的。

在视觉上，他们有一点不同。再次强调一下，我们需要“尽可能地复用代码”。

对于这样一个小组件来说，我们有大量的跨平台的逻辑是重合的：都是显示文本的按钮，都有 active，hover 状态，他们之间唯一的差别就在于视觉样式上的轻微的差别。所以最好的方式是使用同一个组件，然后在必要的地方，用控制语句来区别。

这个是以上控件的示例代码（来自于 `<F8SegmentedControl>`）：

```js
/* from js/common/F8SegmentedControl.js */
class Segment extends React.Component {
  props: {
    value: string;
    isSelected: boolean;
    selectionColor: string;
    onPress: () => void;
  };

render() {
    var selectedButtonStyle;
    if (this.props.isSelected) {
      selectedButtonStyle = { borderColor: this.props.selectionColor };
    }
    var deselectedLabelStyle;
    if (!this.props.isSelected && Platform.OS === 'android') {
      deselectedLabelStyle = styles.deselectedLabel;
    }
    var title = this.props.value && this.props.value.toUpperCase();

    var accessibilityTraits = ['button'];
    if (this.props.isSelected) {
      accessibilityTraits.push('selected');
    }

    return (
      <TouchableOpacity
        accessibilityTraits={accessibilityTraits}
        activeOpacity={0.8}
        onPress={this.props.onPress}
        style={[styles.button, selectedButtonStyle]}>
        <Text style={[styles.label, deselectedLabelStyle]}>
          {title}
        </Text>
      </TouchableOpacity>
    );
  }
}
```

Here we're simply applying different styles depending on which platform the code runs

在这里，我们使用 React Native 的 [Platform module](https://facebook.github.io/react-native/docs/platform-specific-code.html#platform-module)，根据代码在不同的平台上运行，应用不同的样式。在这两个平台上，tab 按钮先使用同样的样式，然后分化（代码来自 `<F8SegmentedControl>`）：

```js
/* from js/common/F8SegmentedControl.js */
var styles = F8StyleSheet.create({
  container: {
    flexDirection: 'row',
    backgroundColor: 'transparent',
    ios: {
      paddingBottom: 6,
      justifyContent: 'center',
      alignItems: 'center',
    },
    android: {
      paddingLeft: 60,
    },
  },
  button: {
    borderColor: 'transparent',
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: 'transparent',
    ios: {
      height: HEIGHT,
      paddingHorizontal: 20,
      borderRadius: HEIGHT / 2,
      borderWidth: 1,
    },
    android: {
      paddingBottom: 6,
      paddingHorizontal: 10,
      borderBottomWidth: 3,
      marginRight: 10,
    },
  },
  label: {
    letterSpacing: 1,
    fontSize: 12,
    color: 'white',
  },
  deselectedLabel: {
    color: 'rgba(255, 255, 255, 0.7)',
  },
});
```

这里，我们使用一个改造后的 [React Native `StyleSheet` API](https://facebook.github.io/react-native/docs/stylesheet.html)，可根据平台进行一下选择转换：

```js
export function create(styles: Object): {[name: string]: number} {
  const platformStyles = {};
  Object.keys(styles).forEach((name) => {
    let {ios, android, ...style} = {...styles[name]};
    if (ios && Platform.OS === 'ios') {
      style = {...style, ...ios};
    }
    if (android && Platform.OS === 'android') {
      style = {...style, ...android};
    }
    platformStyles[name] = style;
  });
  return StyleSheet.create(platformStyles);
}
```

在这 `F8StyleSheet` 函数中，我们解析一个给定的样式对象。如果我们遇有 ios 或者 android 这样的 key 并和当前运行的平台一致，我们就使用 key 对应的样式，否则忽略。这个例子也向我们说明了，React Native 中尽量复用代码以减少重复代码的思想。

现在，我们可以在 app 中复用这个组件了，并且，UI 样式是对 iOS 和 Android 适配的。

### 分离复杂的差异

当在两个平台实现一个不仅在 UI 上分化，同时也几乎没有共同业务逻辑的组件时，我们需要采用不同的方式。下面的例子是 app 中的顶级导航菜单：

![iOS and Android Main Navigation Comparison]({{ site.baseurl }}/static/images/iOS vs. Android@3x.png)

我们可以看到， iOS 版本使用固定的底部导航，而 Android 则使用侧滑菜单。他们之间在动画，样式，甚至菜单内容本身都存在巨大的差异。比如在 Android 上，有类似 logout 这样的菜单选项。

当然，你 *可以* 继续使用一个组件去实现，但是这个组件的逻辑会充斥着大量的条件控制语句，很快就会变得难以理解和维护。

正确的做法，我们应该使用 React Native 内置的 [特定平台下的扩展](http://facebook.github.io/react-native/docs/platform-specific-code.html#platform-specific-extensions)。这个特性允许我们创建两个不同的组件，其中一个叫 `FBTabsView.ios.js`，另外一个叫 `FBTabsView.android.js`。React Native 会根据运行的平台自动找到并加载他们。

#### 内置的 UI 组件

在每个 `FBTabsView` 组件中，我们也可以复用一些内置的 UI 元素。在 Android 版本上使用 [`DrawerLayoutAndroid`](http://facebook.github.io/react-native/docs/drawerlayoutandroid.html) （根据名字我们也可以知道，这个组件只在 Android 中才有）：


```js
/* from js/tabs/F8TabsView.android.js */
render() {
  return (
    <DrawerLayoutAndroid
      ref="drawer"
      drawerWidth={300}
      drawerPosition={DrawerLayoutAndroid.positions.Left}
      renderNavigationView={this.renderNavigationView}>
      <View style={styles.content} key={this.props.activeTab}>
        {this.renderContent()}
      </View>
    </DrawerLayoutAndroid>
  );
}
```

在第 8 行，我们指定绘制菜单内容的函数 `renderNavigationViewb()`，在这个函数里面，我们绘制一个 [`ScrollView`](http://facebook.github.io/react-native/docs/scrollview.html) 组件，在这个组件内，我们填充了很多 `MenuItem`（参见 [`MenuItem.js`](https://github.com/fbsamples/f8app/blob/master/js/tabs/MenuItem.js)）：

```js
/* from js/tabs/F8TabsView.android.js */
renderNavigationView() {
  ...
  return(
    <ScrollView style={styles.drawer}>
      <MenuItem
        title="Schedule"
        selected={this.props.activeTab === 'schedule'}
        onPress={this.onTabSelect.bind(this, 'schedule')}
        icon={scheduleIcon}
        selectedIcon={scheduleIconSelected}
      />
      <MenuItem
        title="My F8"
        selected={this.props.activeTab === 'my-schedule'}
        onPress={this.onTabSelect.bind(this, 'my-schedule')}
        icon={require('./schedule/img/my-schedule-icon.png')}
        selectedIcon={require('./schedule/img/my-schedule-icon-active.png')}
      />
      <MenuItem
        title="Map"
        selected={this.props.activeTab === 'map'}
        onPress={this.onTabSelect.bind(this, 'map')}
        icon={require('./maps/img/maps-icon.png')}
        selectedIcon={require('./maps/img/maps-icon-active.png')}
      />
      <MenuItem
        title="Notifications"
        selected={this.props.activeTab === 'notifications'}
        onPress={this.onTabSelect.bind(this, 'notifications')}
        badge={this.state.notificationsBadge}
        icon={require('./notifications/img/notifications-icon.png')}
        selectedIcon={require('./notifications/img/notifications-icon-active.png')}
      />
      <MenuItem
        title="Info"
        selected={this.props.activeTab === 'info'}
        onPress={this.onTabSelect.bind(this, 'info')}
        icon={require('./info/img/info-icon.png')}
        selectedIcon={require('./info/img/info-icon-active.png')}
      />
    </ScrollView>
  );
}
```

对比来看，iOS 版本在 `render()` 函数中使用了另外一个不同的内置组件，[`TabBarIOS`](http://facebook.github.io/react-native/docs/tabbarios.html)：


```js
/* from js/tabs/F8TabsView.ios.js */
render() {
  var scheduleIcon = this.props.day === 1
    ? require('./schedule/img/schedule-icon-1.png')
    : require('./schedule/img/schedule-icon-2.png');
  var scheduleIconSelected = this.props.day === 1
    ? require('./schedule/img/schedule-icon-1-active.png')
    : require('./schedule/img/schedule-icon-2-active.png');
  return (
    <TabBarIOS tintColor={F8Colors.darkText}>
      <TabBarItemIOS
        title="Schedule"
        selected={this.props.activeTab === 'schedule'}
        onPress={this.onTabSelect.bind(this, 'schedule')}
        icon={scheduleIcon}
        selectedIcon={scheduleIconSelected}>
        <GeneralScheduleView
          navigator={this.props.navigator}
          onDayChange={this.handleDayChange}
        />
      </TabBarItemIOS>
      <TabBarItemIOS
        title="My F8"
        selected={this.props.activeTab === 'my-schedule'}
        onPress={this.onTabSelect.bind(this, 'my-schedule')}
        icon={require('./schedule/img/my-schedule-icon.png')}
        selectedIcon={require('./schedule/img/my-schedule-icon-active.png')}>
        <MyScheduleView
          navigator={this.props.navigator}
          onJumpToSchedule={() => this.props.onTabSelect('schedule')}
        />
      </TabBarItemIOS>
      <TabBarItemIOS
        title="Map"
        selected={this.props.activeTab === 'map'}
        onPress={this.onTabSelect.bind(this, 'map')}
        icon={require('./maps/img/maps-icon.png')}
        selectedIcon={require('./maps/img/maps-icon-active.png')}>
        <F8MapView />
      </TabBarItemIOS>
      <TabBarItemIOS
        title="Notifications"
        selected={this.props.activeTab === 'notifications'}
        onPress={this.onTabSelect.bind(this, 'notifications')}
        badge={this.state.notificationsBadge}
        icon={require('./notifications/img/notifications-icon.png')}
        selectedIcon={require('./notifications/img/notifications-icon-active.png')}>
        <F8NotificationsView navigator={this.props.navigator} />
      </TabBarItemIOS>
      <TabBarItemIOS
        title="Info"
        selected={this.props.activeTab === 'info'}
        onPress={this.onTabSelect.bind(this, 'info')}
        icon={require('./info/img/info-icon.png')}
        selectedIcon={require('./info/img/info-icon-active.png')}>
        <F8InfoView navigator={this.props.navigator} />
      </TabBarItemIOS>
    </TabBarIOS>
  );
}
```

我们可以看到，虽然 iOS 的菜单用了近乎一样的数据，只有细微的数据结构的差别。和 Android 中有一个单独的函数创建菜单项不一样，菜单项 [`TabBarItemIOS`](http://facebook.github.io/react-native/docs/tabbarios-item.html#content) 被当作子组件插入到的父容器菜单中。

这些 `TabBarItem` 大体上相当于 Android 的 `MenuItem` 组件。区别只在于，在 Android 的组件中，我们定义了一个 [`View` component](http://facebook.github.io/react-native/docs/view.html#content):

```js
<View style={styles.content} key={this.props.activeTab}>
  {this.renderContent()}
</View>
```
当 tab 改变时，使用 `renderContent()` 改变内容。而 iOS 的组件却有多个不同的 `View` 组件，比如：

```js
<GeneralScheduleView
  navigator={this.props.navigator}
  onDayChange={this.handleDayChange}
/>
```

他们属于 `TabBarItem` 的一部分，点击之后，这部分会显示。

### 设计迭代周期

不管你在开发什么应用，快速调整 UI 总是非常痛苦的。如果工程师和设计师坐在一起调，整个处理过程会慢下来。

React Native 有一个 [live reload](http://facebook.github.io/react-native/docs/debugging.html#live-reload) 特性。这个特性使得 JavaScript 代码发生改变之后，界面会立刻自动刷新。这个特性可缩短设计迭代流程。

但如果一个组件在不同状态下表现得不一样时该怎么办？比如，有一个按钮，有默认样式，按下状态时的样式，正在加载时，加载完成时的状态。

为了避免每一次都在 app 中和控件交互，我们做了一个可视的调试控件 `Playground`：


```js
/* from js/setup.js */
class Playground extends React.Component {
  constructor(props) {
    super(props);
    const content = [];
    const define = (name: string, render: Function) => {
      content.push(<Example key={name} render={render} />);
    };

    var AddToScheduleButton = require('./tabs/schedule/AddToScheduleButton');
    AddToScheduleButton.__cards__(define);
    this.state = {content};
  }

  render() {
    return (
      <View style={{paddingTop: 20}}>
        {this.state.content}
      </View>
    );
  }
}
```

<!---
TOWATCH: Changes to the Playground component
-->

This simply creates an empty view that can be swapped out to load instead of the actual app. When we combine this with some example definitions in one of the UI components, in this case `AddToScheduleButton.js`:

这个控件简单地创建了一个空的 view，这个 view 可以被真正的控件置换。比如当我们在一个 UI 组件中，把这个和一些定义绑定，以 `AddToScheduleButton` 为例说明：

```js
/* from js/tabs/schedule/AddToScheduleButton.js */
module.exports.__cards__ = (define) => {
  let f;
  setInterval(() => f && f(), 1000);

  define('Inactive', (state = true, update) =>
    <AddToScheduleButton isAdded={state} onPress={() => update(!state)} />);

  define('Active', (state = false, update) =>
    <AddToScheduleButton isAdded={state} onPress={() => update(!state)} />);

  define('Animated', (state = false, update) => {
    f = () => update(!state);
    return <AddToScheduleButton isAdded={state} />;
  });
};
```

在 UI 预览工具中我们可以看到：

![UI preview playground in action with a button and three different states]({{ site.baseurl }}/static/images/button-playground.gif)

这个例子定义了按钮正常态和按压态。为了预览过渡动画，还定义了在两个状态间循环往复的动画状态。

这让工程师和设计师在一起调整基础组件的视觉风格时变得相当快。

`<Playground>` 可以在任何 React Native app 中复用，如果你想使用它，只需要在 `setup()` 函数中加载 `<Playground>` 即可：

```js
/* from js/setup.js */
render() {
  ...
  return (
    <Provider store={this.state.store}>
      <F8App />
    </Provider>
  );
}
```

变为：

```js
/* in js/setup.js */
render() {
  ...
  return (
    <Provider store={this.state.store}>
      <Playground />
    </Provider>
  );
}
```

可以修改 `<Playground>` 组件以加载其他不同的组件。
