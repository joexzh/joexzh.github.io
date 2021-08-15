---
layout:     post
title:      "写 typescript-backbone-router-demo 时遇到的坑"
subtitle:   "traps-when-building-the-typescript-backbone-router-demo"
date:       2017-07-16 19:00:00
author:     "Joe"
tags:
    - typescript
    - webpack
    - js
categories:
    - [coding]
---

项目地址: [typescript-backbone-router-demo](https://github.com/joexzh/typescript-backbone-router-demo)

## 1. 使用 dynamic import: typescript 和 webpack

1. tsconfig.json 中 compilerOptions.module 要选 esnext, 否则 build 出来的 import() 的部分, webpack 无法识别, 无法正确进行 code splitting.

2. import() 中的模块名称至少要让 webpack 识别出文件夹, 否则无法打包进去, 例如

```typescript
import(controllerPath)  // 使用变量 webpack 无法识别模块的引用, 因此无法打包, 加载时就会报错
import('controllers/' + controllerName + 'Controller') // 会把 ./controller 内所有模块打包并 code splitting
```

3. `CommonChunkPlugin` 目前还无法对 dynamic import 和 正常 import 的模块共同进行 code split, 目前已经有人提交 PR, 相信很快就会修复. 参考 [https://github.com/webpack/webpack/issues/4392](https://github.com/webpack/webpack/issues/4392).

## 2. backbone 里面 this.routes 的坑

extends 的 Backbone.Router 直接写 `routes: {}` 就完事了.

typescript 初始化顺序是这样的

1. The base class initialized properties are initialized
2. The base class constructor runs
3. The derived class initialized properties are initialized
4. The derived class constructor runs

父类初始化的时候获取不到子类的 routes, 因此要 hack 进 backbone 的 `private code`, 如下面, 真是累觉不爱

```typescript
export default class AppRouter extends Backbone.Router {

    constructor(defaultControllerName: string, options?: Backbone.RouterOptions) {
        super(options);
        
        this.routes = <any>{
            '': 'dispatchController',
            ':controller': 'dispatchController',
            ':controller/:action': 'dispatchController',
            ':controller/:action/*path': 'dispatchController'
        };
        (<any>this)._bindRoutes();
    }

    dispatchController(controllerName?: string, actionName?: string, params?: string) {
        ...
    }
}
```

Backbone.View 提供了 event(), 因此不用担心这个问题.

## 3. 使用 history 要服务器支持

如果路由使用 history api， 必须要服务器支持.

例如程序起点是 `www.domain.com`, 访问 `www.domain.com/home` 期望路由到 home controller default action. 如果没有服务器支持就会报错 404.

所谓的服务器支持, 就是要服务器支持重写 url. 设定一个规则, 访问 `www.domain.com` 无论 com 后面跟了什么内容, 服务器都会当成访问 `www.domain.com`, 当然 `js` `png` 这些资源文件除外.

因此, 我哭着再一次翻开 <精通正则表达式> 这本书.

下一篇会介绍怎么配置 IIS 的 URL rewrite 去支持 history api.