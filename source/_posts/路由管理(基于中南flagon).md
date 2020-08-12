

路由配置表形如

```javascript
{
  name: '更新日志',
  path: '/purchaser/zoina/changelog',
  breadcrumb: [
    {
      title: '工作台',
    },
    {
      title: '更新日志',
    },
  ],
  component: () => import('./components/zoina/changelog'),
},
```



### 前置介绍

dva/dynamic

不需要在头部import组件

```jsx
const CompontentPage = dynamic({
  component: () => import('./routes/CompontentPage')
})
//Route组件使用还是一样
<Route path="/" exact component={IndexPage} />
```

### 实现

flagon是基于dva来启动整个项目的。

#### index.js

```javascript
// 此处export 项目中所用的库，如moment，request等
const {
      allowIE = true,
      routes,
      swEnable,
    } = options
// 此处做一些路由是否定义，是否允许IE之类的判断。
// 最后，生成一个app实例，并挂载到div上
const app = createInstance({ ...options, onStateChange })
app.start('#app')
```

options为外部传入参数，即flagon.start(options)

#### instance.js

createInstance实际上就是对dva框架进行二次封装

```jsx
//instance.js
import dva from 'dva'
import createHistory from 'history/createBrowserHistory'
import createLoading from 'dva-loading'
import { createLogger } from 'redux-logger'
import config from './config'
import page from './page-model'
import router from './router'

const history = createHistory()

const createInstance = (customOptions) => {
  const { allAuth, onStateChange } = customOptions
  // config.initOptions即为return { ...initalOptions, ...customOptions }
  const options = config.initOptions({ ...customOptions }) 
  const app = dva({
    history,
    onError: options.onError,
    onStateChange,
    ...process.env.production === 'production' ? {} : {
      onAction: createLogger(),
    },
  })

  app.options = options
  app.use(createLoading())
  app.model(page())
  app.router((props) => {
    // 调用router函数
    return router({
      // 这里传进去props我看了一下router代码,除去下面的allAuth,Options，app，只用到一个props.history,应该是dva提供的。
      ...props,
      allAuth,
      options, //此处包含initialOptions和customOptions
      /*
      	const initalOptions = {
        onError,
        router,// 即为router.js
        RootComp,// 根组件，传到下文LoadUserInfo组件中
      }
      */
      app,
    })
  })

  return app
}

export default createInstance
```



#### router.js

配置路由，绘制框架页面。

```jsx
import React from 'react'
import { Router, Route, Switch } from 'dva/router'
import dynamic from 'dva/dynamic'
import { LocaleProvider } from 'antd'
import Comp from './root-comp'
import LoadUserInfo from './load-user-info'
import { locales, locale } from './locale'

export default function (props) {
  const { app, allAuth, options } = props
  const { frontPages, RootComp = Comp, routes } = options
  const passProps = {
    app: props.app,
    routes,
    allAuth,
    options,
  }

  const front = routes.filter(route => route.front)

  return (
    <LocaleProvider locale={(locales[locale] || locales['zh-CN']).antdLocale}>
      <Router history={props.history}>
        <Switch>
          {
            front.map(item => (
              <Route exact
                key={item.path}
                path={item.path}
                component={(innerProps) => {
                  const DynamicEl = dynamic({
                    app,
                    component: item.component,
                  })

                  return <DynamicEl {...innerProps} />
                }}
              />))
          }
          {
            frontPages instanceof Function && frontPages(passProps)
          }
          {// 该组件绘制页面}
          <LoadUserInfo {...passProps} RootComp={RootComp} />
        </Switch>
      </Router>
    </LocaleProvider>
  )
}
```

可以发现router函数本身的参数是包含自身的，这样有什么意义目前还不知晓。

LoadUserInfo接收的props(主要为allAuth,router,app,options,RootComp)除去app都来自flagon.strat(options)中的options，其中RootComp组件有默认，也可在options中进行自定义设置。

#### load-user-info.js

```jsx
render () {
    const { RootComp } = this.props
    const { loggedIn, resources } = this.state
    return loggedIn && (<RootComp {...this.props} resources={resources}>
      <Route path="/" render={renderProps => <BasicLayout {...renderProps} {...this.props} />} />
    </RootComp>)
  }
```
到这里就进行具体页面绘制了，BasicLayout组件不多关注其他的，具体关注面包屑和菜单，权限控制本次也不涉及。

中南菜单是在后台配的，options有一个resourceAPI。RootComp中的resource调用该接口获取。



// 之前以为只有内容部分是根据路由动态渲染的，实际上包括面包屑，title都是。需要对比一下毕设项目结构与中南结构。主要里Route组件里到底需要包含什么。

// 给出一个总体的代码层次。