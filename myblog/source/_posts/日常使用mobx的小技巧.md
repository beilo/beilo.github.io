---
title: 日常使用mobx的小技巧
date: 2019-06-08 22:06:40
tags:javascript
toc: true
---
### 日常使用mobx的小技巧

由于自己开发的项目都是中小型项目，所以在技术选型上使用了mobx。但是使用过程中发现关于mobx的技术文章并不多。于是萌发出写这篇文章的想法。请轻喷。

##### mobx一些有关资料

[中文文档](https://cn.mobx.js.org/)
[github仓库](https://github.com/mobxjs/mobx)
[npm地址](https://www.npmjs.com/package/mobx)

##### 添加mobx

```
npm i mobx
npm i mobx-react
```

mobx用来操作store（也就是数据操作层，model层），而mobx-react则是用来操作view（也就是视图层，Component层）。

###### mobx常用操作符

1. observable，将`JS`基本数据类型、引用类型、普通对象、类实例、数组和映射，转换为可观察数据。
2. action，用来修改`observable`的数据的动作，只有`action`和`runInAction`才能修改`observable`。
3. runInAction，用来在异步的时候执行修改`observable`的数据的动作。例如网络请求后修改数据。
4. computed，根据现有的`observable`的值或其它计算值衍生出的值。只有在`view`使用了`computed`的值，`computed`才会执行计算

###### mobx-react常用操作符

1. observer，将 `React组件` 转变成响应式组件。
2. inject，将组件连接到提供的`stores` 。一般是用来连接到上层组件提供的`store`或者`全局store`。
3. Provider，它是一个`react组件`，用来向下传递 `stores`。任意子组件可以使用`inject`来获取`Provider`的`store`。

##### 代码

下面会贴点自己的代码。希望能给大家带来一些帮助。

全局store

```javascript
// 文件 index.jsx
import { Provider } from "mobx-react";
import * as stores from "../../stores";
class MainView extends Component {
render() {
return (
<React.Fragment>
<Provider {...stores}>  // 这里是全局stores的配置
<Switch>
{routers.map((item, index) => {
return (
<Route
exact
key={item.path}
path={item.path}
component={item.component}
/>
);
})}
</Switch>
</Provider>
</React.Fragment>
);
}
}


// 文件 ../../stores/index.js
import aStore from "./aStore";
import bStore from "./bStore";
export { aStore, bStore};

// 文件 aStore.js
class AStore {
@observable info = {};
@observable line = {};
@action
onInfo = data => {
this.info = data;
};
@action
onLine = data => {
this.line = data;
};
}
const aStore = new AStore();
export default aStore;

// 组件使用aStore
@inject(all => ({
aStore: all.aStore // 连接到aStore
}))
@observer
class Detail extends Component {
updateInfo = () => {
const aStore = this.props.aStore;
aStore.onInfo(info) // 改变store属性
}
render() {
const aStore = this.props.aStore; // 获取到全局store
return (
<div>
{aStore.info.name} // 使用里面的属性
<Button onClick={this.updateInfo}>改变info</Button>
</div>
);
}
}
```

组件内可观察数据。

```javascript
@observer
class Detail extends Component {
@observable info = {};
@observable index = 1;

@computed get sum() {
return index * 4
}

@action
onInfo = data => {
this.info = data;
};
updateInfo = () => {
this.onInfo(info) // 改变store属性
}
render() {
return (
<div>
{this.info.name} // 使用里面的属性
<Button onClick={this.updateInfo}>改变info</Button>
</div>
);
}
}
```

最后一种是我自己项目使用的，个人觉得不错。可以分离逻辑层和视图层。如果想使用全局`stores`，直接用`inject`导入第一种方式注入的`stores`。能较好划分`全局stores`和`单业务store`的职责，而不是无脑的以树的方式全挂在`index.js`上面。`子组件`最好跟`父组件`用同一个`store`，方便沟通的同时层级不多也不会导致`store`太过复杂。

```javascript
// 逻辑层 用来处理业务逻辑
import { observable, runInAction } from "mobx";
import { aService, bService } from "../../../services/dispatch";
import { T } from "react-toast-mobile";
class ListStore {
@observable list = [];

serInitData = async () => {
try {
T.loading();
const data = await aService()
runInAction("serInitData", () => {
this.list = data || [];
});
} catch (error) {
T.notify(error.message);
console.error(error);
} finally {
T.loaded();
}
};

serBService = async vehicleNum => {
try {
T.loading();
await bService(vehicleNum);
runInAction("serBService", () => {
this.list = this.list.filter(params => {
return params.vehicleNum != vehicleNum;
});
});
} catch (error) {
T.notify(error.message);
// eslint-disable-next-line no-console
console.error(error);
} finally {
T.loaded();
}
};
}
export default ListStore;


// 视图层 纯粹的视图展示和操作触发
@observer
class Detail extends Component {
store = new ListStore();

componentDidMount() {
// 初始化数据
this.store.serInitData();
}

updateInfo = () => {
this.store.serBService(); // 调用bService
}
render() {
return (
<div>
{this.store.list.map((item)=>{
return 
<Provider myStore={this.store} >
<Item key={item.guid} />
</Provider>
})} // 轮询列表
<Button onClick={this.updateInfo}>改变info</Button>
</div>
);
}
}

const Item = inject(allStores => ({
aStore: all.aStore, // 连接到aStore
myStore: allStores.myStore, // 连接到父组件的myStore
}))(
observer(function(props) { // 使用function方式在发送action等操作时，会带上Item这个组件名字，能变相的看到发送的来源。使用箭头函数则不会。而且react推荐的无状态组件也是使用的function

return (
<div>
{this.props.aStore.xxxxx} // 使用全局store
{this.props.myStore.xxxxx} // 使用父组件的store
</div>
);
}));

```

##### 一些备注

1. 使用`@observer`后，无法通过组件的`refs`属性调用其对应的方法？

```javascript
this.myrefs.wrappedInstance.testFunc() //这样就可以调用了，参考链接
// https://stackoverflow.com/questions/43847401/reactnative-mobx-how-to-access-component-refs-from-mobx
```

2. `mobx`如何自动保存数据

```javascript
import { observable, action, autorun, toJS, set } from "mobx";

function autoSave(store, save) {
let firstRun = true;
autorun(() => {
// 此代码将在每次运行任何可观察属性时运行
// 对store进行更新。
const json = JSON.stringify(toJS(store));
if (!firstRun) {
save(json);
}
firstRun = false;
});
}
class RouteState {
@observable state = {};
constructor() {
this.load();
autoSave(this, this.save.bind(this));
}
load() {
const storeTemp = sessionStorage.getItem("route_state");
if (storeTemp) {
const data = JSON.parse(storeTemp);
set(this, data);
}
}
save(json) {
sessionStorage.setItem("route_state", json);
}
@action.bound
actionState(_state) {
this.state = _state;
}
}

// 参考链接
https://stackoverflow.com/questions/40292677/how-to-save-mobx-state-in-sessionstorage
```

3. `mobx`的使用非常灵活。可以多种使用方式在同一个项目使用。并不冲突。
4. 推荐所有的组件都`observer`化，这并不会造成性能损耗，反而会优化组件。
5. 想不到了。想起来再更新吧。
