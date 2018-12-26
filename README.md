

## VUE-SSR 配合webpack自动构建


```shell
yarn  #安装依赖

yarn dev # 开发环境

yarn build # 线上构建
yarn start # 线上运行
```


# SPA（Single-Page Application - 单页应用程序）VS 服务器端渲染(SSR)：
## 优势：
### 1，更好的SEO：抓包工具不会等待异步javascript完成再抓取页面内容。
### 2，更快的内容到达时间(time-to-content)：无需等待javascript加载完成才进行渲染；
## 劣势：
### 1，某些特定代码只能在某些钩子函数中使用；
### 2，前端人员维护nodejs服务并考虑服务器负载问题和缓存问题；

 # 服务器端渲染(SSR) VS 预渲染(Prerendering)
### 预渲染：只是用来改善某些特定营销页面(如：/，/index等)的SEO；
### webpack预渲染：https://github.com/chrisvfritz/prerender-spa-plugin



# 开始
## 核心依赖：vue-server-renderer + nodejsServer
## 钩子函数：仅beforeCreated和created会在服务端渲染调用
## 创建工厂实例函数：
```
// server.js
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
# 构建步骤

## 异步路由配置，按需加载：
```
// router.js
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

### 1,页面路由组件暴露自定义静态函数： asyncData，实例化之前无法调用this
```
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

```
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

### 1，在路由导航之前解析数据：等待视图所需数据全部解析才进入页面
   ### 判断路由组件差异，触发asyncData方法，同步数据到store

### 2，匹配渲染的视图，再获取数据：通过全局mixin beforeMount方法调用asyncData方法，

```
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