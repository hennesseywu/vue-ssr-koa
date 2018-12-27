

## VUE-SSR 配合webpack自动构建


```shell
yarn  #安装依赖

yarn dev # 开发环境

yarn build # 线上构建
yarn start # 线上运行
```


# SPA（Single-Page Application - 单页应用程序）VS 服务器端渲染(SSR)：

### 优势：
### 1，更好的SEO：抓包工具不会等待异步javascript完成再抓取页面内容。
### 2，更快的内容到达时间(time-to-content)：无需等待javascript加载完成才进行渲染；

### 劣势：
### 1，某些特定代码只能在某些钩子函数中使用；
### 2，前端人员维护nodejs服务并考虑服务器负载问题和缓存问题；

# 服务器端渲染(SSR) VS 预渲染(Prerendering)
### 预渲染：只是用来改善某些特定营销页面(如：/，/index等)的SEO；
### webpack预渲染：https://github.com/chrisvfritz/prerender-spa-plugin



# 开始
## 核心依赖：vue-server-renderer + nodejsServer
## 通过 vue-server-renderer.createBundleRenderer指定渲染模板，再通过renderToString转成html字符串，整个过程通过上下文context作为容器
##（开发环境和正式环境配置不同，开发环境配置了热重载便于开发，正式环境之间读取build后的文件）

```server.js
const fs = require('fs')
const Koa = require('koa')
const path = require('path')
const chalk = require('chalk')
const LRU = require('lru-cache')
const send = require('koa-send')
const Router = require('koa-router')
const setupDevServer = require('../build/setup-dev-server')
const {
  createBundleRenderer
} = require('vue-server-renderer')


// 缓存
const microCache = LRU({
  max: 100,
  maxAge: 1000 * 60 // 重要提示：条目在 1 秒后过期。
})

const isCacheable = ctx => {
  // 实现逻辑为，检查请求是否是用户特定(user-specific)。
  // 只有非用户特定(non-user-specific)页面才会缓存
  console.log(ctx.url)
  if (ctx.url === '/b') {
    return true
  }
  return false
}

//  第 1 步：创建koa、koa-router 实例
const app = new Koa()
const router = new Router()

let renderer
const templatePath = path.resolve(__dirname, './index.template.html')

// 第 2步：根据环境变量生成不同BundleRenderer实例
if (process.env.NODE_ENV === 'production') {
  // 获取客户端、服务器端打包生成的json文件
  const serverBundle = require('../dist/vue-ssr-server-bundle.json')
  const clientManifest = require('../dist/vue-ssr-client-manifest.json')
  // 赋值
  renderer = createBundleRenderer(serverBundle, {
    runInNewContext: false,
    template: fs.readFileSync(templatePath, 'utf-8'),
    clientManifest
  })
  // 静态资源
  router.get('/static/*', async (ctx, next) => {
    await send(ctx, ctx.path, {
      root: __dirname + '/../dist'
    });
  })
} else {
  // 开发环境
  setupDevServer(app, templatePath, (bundle, options) => {
    console.log('rebundle..............')
    const option = Object.assign({
      runInNewContext: false
    }, options)
    renderer = createBundleRenderer(bundle, option)
  })
}

const render = async (ctx, next) => {
  ctx.set('Content-Type', 'text/html')
  const handleError = err => {
    if (err.code === 404) {
      ctx.status = 404
      ctx.body = '404 Page Not Found'
    } else {
      ctx.status = 500
      ctx.body = '500 Internal Server Error'
      console.error(`error during render : ${ctx.url}`)
      console.error(err.stack)
    }
  }
  const context = {
    url: ctx.url
  }
  // 判断是否可缓存，可缓存并且缓存中有则直接返回
  const cacheable = isCacheable(ctx)
  if (cacheable) {
    const hit = microCache.get(ctx.url)
    if (hit) {
      console.log('从缓存中取', hit)
      return ctx.body = hit
    }
  }
  try {
    const html = await renderer.renderToString(context)
    ctx.body = html
    if (cacheable) {
      console.log('设置缓存: ', ctx.url)
      microCache.set(ctx.url, html)
    }
  } catch (error) {
    handleError(error)
  }

}

router.get('*', render)
app
  .use(router.routes())
  .use(router.allowedMethods())



const port = process.env.PORT || 3000
app.listen(port, () => {
  console.log(chalk.green(`server started at localhost:${port}`))
})

```

## 钩子函数：ssr情况下，仅beforeCreated和created会在服务端渲染调用
## 创建工厂实例函数：
``` server.js
const createApp = require('./app')

server.get('*', (req, res) => {
  const context = { url: req.url }
  const app = createApp(context)

  renderer.renderToString(app, (err, html) => {
    // 处理错误……
    res.end(html)
  })
})
```

## 异步路由配置，按需加载：
```router.js
import Vue from 'vue'
import Router from 'vue-router'

Vue.use(Router)

export function createRouter () {
  return new Router({
    mode: 'history',
    routes: [
      { path: '/', component: () => import('./components/Home.vue') },
      { path: '/item/:id', component: () => import('./components/Item.vue') }
    ]
  })
}

```


# vuex：数据预取存储容器

### 1,页面路由组件暴露自定义静态函数： asyncData，实例化之前无法调用this只能传入store
```index.vue
<template>
  <div>{{ item.title }}</div>
</template>

<script>
export default {
  asyncData ({ store, route }) {
    // 触发 action 后，会返回 Promise
    return store.dispatch('fetchItem', route.params.id)
  },
  computed: {
    // 从 store 的 state 对象中的获取 item。
    item () {
      return this.$store.state.items[this.$route.params.id]
    }
  }
}
</script>
```

## 2，服务端数据预取：
### 1，router.getMathedComponent获取到匹配组件；
### 2，判断是否有暴露asyncData方法；
### 3，将asyncData方法执行生成的store附加到上下文中；
### 4，使用template时，context.state将作为window.__INITIAL_STATE__状态，自动嵌入最终THML中，client模式下挂载之前store已经可以获取到状态。

```entry-server.js
import { createApp } from './app'

export default context => {
  return new Promise((resolve, reject) => {
    const { app, router, store } = createApp()

    router.push(context.url)

    router.onReady(() => {
      const matchedComponents = router.getMatchedComponents()
      if (!matchedComponents.length) {
        return reject({ code: 404 })
      }

      // 对所有匹配的路由组件调用 `asyncData()`
      Promise.all(matchedComponents.map(Component => {
        if (Component.asyncData) {
          return Component.asyncData({
            store,
            route: router.currentRoute
          })
        }
      })).then(() => {
        // 在所有预取钩子(preFetch hook) resolve 后，
        // 我们的 store 现在已经填充入渲染应用程序所需的状态。
        // 当我们将状态附加到上下文，
        // 并且 `template` 选项用于 renderer 时，
        // 状态将自动序列化为 `window.__INITIAL_STATE__`，并注入 HTML。
        context.state = store.state

        resolve(app)
      }).catch(reject)
    }, reject)
  })
}
```
## 3，客户端预取数据

### 1，在路由导航之前解析数据：等待视图所需数据全部解析才进入页面,判断路由组件差异，触发asyncData方法，同步数据到store

### 2，匹配渲染的视图，再获取数据：通过全局mixin beforeMount方法调用asyncData方法，

```entry-client.js
Vue.mixin({
  beforeRouteUpdate (to, from, next) {
    const { asyncData } = this.$options
    if (asyncData) {
      asyncData({
        store: this.$store,
        route: to
      }).then(next).catch(next)
    } else {
      next()
    }
  }
})
```
### 拆分路由的情况：使用 store.registerModule 惰性注册(lazy-register)这个模块

# 客户端激活

### vue浏览器端接管服务端发送的HTML，data-server-rendered="true"
### 强制使用应用程序的激活模式 app.$mount('#app', true)
