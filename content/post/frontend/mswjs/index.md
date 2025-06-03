---
title: mock - msw.js
description: 多人合作时后端拖着不给接口时不得不学的一环
date: 2025-06-03 02:00:00+0000
categories:
    - frontend
tags:
    - vue
---

## 背景

作为一个全栈开发，但通常来说以后端居多，平常写前端也基本上就是自己把接口写完了再去对接，因此接触到mock比较少（应该来说对前端仅限于能跑就行的程度），由于近期比赛与其他同学合作，处于引导后端同学自主开发的想法，我主动承接了前端的工作，并表示不会参与后端的任何工作，但是！由于某同学迟迟部署不成功，拿不到上线的接口的我只能开始学习mock（）

## 什么是mock

Mock（模拟）是软件开发中一种重要的技术，特别是在前端开发中，它允许开发者在没有真实后端服务的情况下进行开发和测试。

Mock 是指创建模拟对象或数据来替代真实依赖项的技术。在前端开发中，Mock 主要用于：

- 模拟后端 API 响应
- 模拟浏览器 API 或环境
- 模拟第三方服务
- 在测试中隔离被测代码

Mock Service Worker（MSW）是一个用于浏览器和 Node.js 的 API 模拟库。使用 MSW，您可以拦截发出的请求，观察它们，并使用模拟响应来回答它们。

下文中就介绍如何利用msw.js来进行mock操作，链接：[https://mswjs.io/](https://mswjs.io/)

## 安装

``` shell
npm i msw --save-dev
```

msw.js依赖一个worker工作，在正式使用之前，我们需要把工作脚本导入public目录

``` shell
npx msw init <PUBLIC_DIR> --save
```

## 模拟接口

首先，我们需要用一个数组来定义我们要模拟的接口列表，例如：

``` javascript
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.get('/user', () => {
    return HttpResponse.json({
      firstName: 'John',
      lastName: 'Maverick',
    })
  }),
]
```

该例子就是模拟了'/user'这个get接口，可以看到，模拟返回值是一个json

## 初始化worker并启动

在`main.js`中使用如下代码：

``` javascript
import { setupWorker } from 'msw/browser'
import { handlers } from './handlers'

export const worker = setupWorker(...handlers)
worker.start()
```

即可启动mock，在控制台看到：`[MSW] Mocking enabled.`即为启动成功

在实际使用中，我们可以通过`import.meta.MODE`区分生产环境和开发环境来决定是否启动mock（vite下）

## 会遇到的问题

worker必须要在https和受信任的证书下才能工作，所以我们需要借助[mkcert：https://github.com/FiloSottile/mkcert](https://github.com/FiloSottile/mkcert)生成证书

### mkcert安装

以brew为例：

``` shell
brew install mkcert
```

如果你想在firefox上开发，你还需要安装nss

``` shell
brew install nss
```

### 安装mkcert为受信任的证书机构

``` shell
mkcert -install
```

安装后，生成的证书可以被浏览器信任，而不是显示为不安全的证书

### 生成证书

``` shell
mkcert -key-file key.pem -cert-file cert.pem localhost
```

生成了key.pem和cert.pem

### node中启用https

以vite为例：vite.config.js：

``` javascript
import {defineConfig} from 'vite'
import vue from '@vitejs/plugin-vue'
import fs from "fs";

// https://vite.dev/config/
export default defineConfig(({command, mode}) => {
    if (mode === 'development') {
        // development 模式下启用https
        return {
            plugins: [vue()],
            server: {
                https: {
                    key: fs.readFileSync('./key.pem'),
                    cert: fs.readFileSync('./cert.pem')
                },
            }
        }
    }
    return {
        plugins: [vue()],
    }
})

```

## 后记

我们大前端终于独立了口牙