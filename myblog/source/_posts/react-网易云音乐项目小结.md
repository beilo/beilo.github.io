---
title: react 网易云音乐项目小结
date: 2018-10-25 10:56:34
tags: javascript
toc: true
---


# 学习搭建一个网易云音乐的项目

## 采用的技术框架
- react 视图层框架
- bable js转译
- webpack 打包
- axios 网络通讯
- dvaJs model层框架
- [网易云音乐api](https://github.com/Binaryify/NeteaseCloudMusicApi)


## 项目步骤

#### babel 配置
项目中使用到了最新的特性,所以针对性的配置了 `babel`
``` javascript
package.json

"devDependencies": {
    "@babel/core": "^7.1.2",
    "@babel/plugin-proposal-class-properties": "^7.1.0", // 支持静态属性
    "@babel/plugin-proposal-decorators": "^7.1.2", //支持装饰器
    "@babel/plugin-transform-runtime": "^7.1.0",  //供编译模块复用工具函数
    "@babel/preset-env": "^7.1.0", //转译所有
    "@babel/preset-react": "^7.0.0", //转译react代码
    "babel-loader": "^8.0.4",
    "babel-plugin-import": "^1.9.1", //import 按需加载
},
"dependencies": {
    "@babel/runtime": "^7.1.2",
}
```

``` Json
.babelrc

{
    "presets": ["@babel/preset-env", "@babel/preset-react"],
    "plugins": [
        ["@babel/plugin-proposal-decorators", {
            "legacy": true
        }],
        "@babel/plugin-proposal-class-properties",
        [
            "@babel/plugin-transform-runtime",
            {
                "corejs": false,
                "helpers": true,
                "regenerator": true,
                "useESModules": false
            }
        ]
    ]
}
```

#### webpack 配置
``` javascript
webpack.config.js

const HtmlWebPackPlugin = require("html-webpack-plugin"); //复制html并且添加js链接
const path = require('path'); //管理路径
const webpack = require('webpack');



module.exports = {
    entry: path.join(__dirname, "src", "index.jsx"),
    output: {
        path: path.resolve(__dirname, 'dist'),
        filename: 'bundle.js'
    },
    devtool: 'source-map',
    resolve: {
        extensions: ['.js', '.jsx']
    },
    module: {
        rules: [{
                test: /\.js$/,
                exclude: /node_modules/,
                use: {
                    loader: "babel-loader"
                },
                exclude: [
                    path.join(__dirname, '../node_modules') // 由于node_modules都是编译过的文件，这里我们不让babel去处理其下面的js文件
                ]
            }, {
                test: /.jsx$/, //使用loader的目标文件。这里是.jsx
                loader: 'babel-loader'
            },
            {
                test: /\.html$/,
                use: [{
                    loader: "html-loader"
                }]
            },
            {
                test: /\.css$/,
                use: ['style-loader', {
                    loader: 'css-loader',
                    // options: {
                    //     modules: true,
                    //     localIdentName: '[path][name]__[local]--[hash:base64:5]'
                    // }
                }]
            },
            {
                test: /\.less$/,
                use: [{
                    loader: "style-loader" // creates style nodes from JS strings
                }, {
                    loader: 'css-loader',
                    // options: {
                    //     modules: true,
                    //     localIdentName: '[path][name]__[local]--[hash:base64:5]'
                    // } // translates CSS into CommonJS
                }, {
                    loader: "less-loader" // compiles Less to CSS
                }],
                include: [
                    path.resolve(__dirname, "node_modules", "antd"),
                ]
            },
            {
                test: /\.(png|jpg|gif|woff|woff2|eot|ttf|svg)$/i,
                use: [{
                    loader: 'url-loader',
                    options: {
                        limit: 100000
                    }
                }]
            }
        ]
    },
    plugins: [
        new HtmlWebPackPlugin({
            template: path.join(__dirname, "src", "index.html"),
            filename: path.join(__dirname, "dist", "index.html")
        }),
        new webpack.HotModuleReplacementPlugin()
    ],
    devServer: {
        port: 4954,
        contentBase: "./dist",
        historyApiFallback: true,
        hot: true
    }
}
```

#### index.jsx 入口文件

``` javascript
index.jsx

import dva from "dva";
import routers from './config/Routers'
import history from './config/History'

// model
import BaseModel from './model/BaseModel';
import LoginModel from './model/LoginModel'
import SongListModel from './model/SongListModel';
import PlayListDetailModel from './model/PlayListDetailModel';

const app = dva({
    history: history, //自定义history,可以去除url上的_k和#之类的符号
    onError(error) {
        console.error(error.stack); //同意的error处理
    },
});

// 引入model层
app.model(LoginModel)
app.model(SongListModel)
app.model(PlayListDetailModel)
app.model(BaseModel)

// 引入router
app.router(routers)

// 定义些全局变量
window.App_ = {
    history: history,
    dva: app
}

// 启动,在class="root"
app.start("#root")

// webpack service配置
module.hot.accept();
```

#### History.js
[History和Routers请参考](https://github.com/brickspert/blog/issues/3)

``` javascript
History.js

import createHistory from 'history/createBrowserHistory';
export default createHistory();
```


#### Routers.jsx
由于`router-v4`把`<Route/>`组件化,所以`router-v3`那种配置便不可用了.在`v4`中`<Router/>`可以当做子组件使用,具体参考`BaseLayout.jsx`

``` javascript
Routers.jsx

import React from 'react'
import { Route, Router, Switch } from 'react-router-dom'

import BaseLayout from '../layout/BaseLayout'


import history from './History';
const routers = () => {
    return (
        <Router history={history}>
            <Route path="/" component={BaseLayout}>
            </Route>
        </Router>
    )
}

export default routers
```

#### BaseLayout.jsx
``` javascript
class BasicLayout extends React.Component {

    push(){
        // 这是跳转页面的方法,具体使用参考history库
        // App_.history是在index.jsx中定义的全局变量
        App_.history.push("/songList")
    }

    render() {
        return (
            <Layout>
                <Header>
                    Header
                </Header>
                <Content>
                    <div>
                        <Switch>
                            <Route path="/login" component={Login}></Route>
                            <Route path="/songList" component={SongList}></Route>
                            <Route path="/playListDetail" component={PlayListDetail}>
                            </Route>
                        </Switch>
                    </div>
                </Content>
                <Footer style={{ textAlign: 'center', height: "150px" }}>
                    Footer
                </Footer>
            </Layout>
        )
    }
}
```

#### model层案例

`namespace`是一个作用域,用于你查找对应`model`的数据:  
使用`dvaJS`管理`state`后,会有一个`this.state`对象,所有的`state`都以`namespace`为`key`,整合到了`this.state`中.那么如何获取我对应`model`的`state`呢?  
这个时候只要通过`this.state.[namespace]`就能获取到对应的`state`了,比如`this.state.songListNamespace`就能获取到下面`state`了.

``` javascript
export default {
    namespace: "songListNamespace", 
    state: {
        playlist: [],
        loading: false,
    },
    effects: {
        * getDataList(_, {
            call,
            put,
            select
        }) {
            // call暂时不知道干嘛的,还没用过

            // select 用于获取state 例如:
            // const loading = yield select(state => state.songListNamespace.loading) 就是获取loading状态.PS:state是全局的,所以要加上namespace

            // 调用接口
            const response = yield RecommendService.songList()
            const data = response.result
            // put 相当于一个动作,目的是调用下面reducers中的addData方法
            yield put({
                type: "addData",
                payload: {
                    data: data
                }
            })
        },

        // 展示下怎么传递参数到effects
        * pushPlayListDetail({
            payload //传递的参数,里面也可以放回调函数,回调到view层
        }, {
            put
        }) {
            // push带参数
            App_.history.push({
                pathname: '/playListDetail',
                state: {
                    id: payload.id
                }
            })
        }
    },
    reducers: {
        addData(state, action) {
            return {
                ...state,
                playlist: action.payload.data
            }
        }
    }
}
```

#### 上面model对应的view

``` javascript
import React from 'react'
import { List, Avatar, Spin } from 'antd';
import { connect } from "dva";

// 作用域
const namespace = "songListNamespace"
// 将state 转化为 this.props.songListData PS:只是使用方式上的转化
const mapStateToProps = (state) => {
    const songListData = state[namespace]
    return {
        songListData
    }
}
// 定义的action,用于调用model中的effects中的方法.PS:需要指明作用域,可以跨域调用,state同理
const mapDispatchToProps = (dispatch) => {
    return {
        getDataList: () => {
            dispatch({
                type: `${namespace}/getDataList`
            })
        },
        onPushSongList: (id) => {
            dispatch({
                type: `${namespace}/pushPlayListDetail`,
                payload: {
                    id, id
                }
            })
        }
    }
}

class SongList extends React.Component {
    state = {
        playlist: [],
        loading: false
    }

    componentDidMount() {
        // 调用model中的getDataList方法,调用流程如下
        // 当前文件 mapDispatchToProps -> getDataList -> model -> effects -> getDataList
        this.props.getDataList()
    }

    render() {
        return (
            <Spin tip="加载数据,请稍等..." spinning={this.props.songListData.loading} delay="500">
                <List
                    itemLayout="horizontal"
                    dataSource={this.props.songListData.playlist}
                    renderItem={(item) => (
                        <List.Item>
                            <List.Item.Meta
                                avatar={<Avatar src={item.picUrl} />}
                                title={<span
                                    onClick={() => {
                                        this.props.onPushSongList(item.id)
                                    }}
>{item.name}</span>}
                                description={<span>{item.copywriter}</span>}
                            >
                            </List.Item.Meta>
                        </List.Item>
                    )}
                />
            </Spin>
        )
    }
}
// 将mapStateToProps,mapDispatchToProps,SongList整合为一个整体
export default connect(mapStateToProps, mapDispatchToProps)(SongList);
```

const songListData = state[namespace]
    return {
        songListData
    }

#### connect的方便写法
下面这种写法要通过`ES7`的装饰器才可以使用,上面`babel`中有介绍
``` javascript
@connect(
    _state => ({
        songListData: _state[namespace], // 总的
        playlist:_state[namespace].playlist, //分开的
        loading:_state[namespace].loading, //分开的
    }),
    _dispatch => ({
        getDataList: () => {
            _dispatch({
                type: `${namespace}/getDataList`
            })
        },
        onPushSongList: () => {
            _dispatch({
                type: `${namespace}/pushPlayListDetail`,
                payload: {
                    id, id
                }
            })
        }
    }))
class SongList extends React.Component {
    //使用
    // this.props.songListData 
    // this.props.playlist 
    // this.props.loading
}
```

## 总结
前端的技术栈有点长,要慢慢的整理.现在我也只是简单使用,`dvaJs`更是入门.  
接下来还有`react`的`PureComponent`,`无状态组件`,`生命周期`等等.  
`dva`的话还要了解`react-router/react-saga/react-redux`,等等.学习的路还很长啊.
