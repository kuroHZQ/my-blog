# single-spa

**微前端**

实现一套微前端架构，可以把其分成四部分（参考：https://alili.tech/archive/11052bf4/）

1. 加载器：也就是微前端架构的核心，主要用来调度子应用，决定何时展示哪个子应用（看了几篇文章都是通过System.js实现）

2. 包装器：有了加载器，可以把现有的应用包装，使得加载器可以使用它们，也就是single-spa充当的角色

3. 主应用：一般是包含所有子应用公共部分的项目

4. 子应用：众多展示在主应用内容区的应用


总的来说是这样一个流程：用户访问index.html后，浏览器运行加载器的js文件，加载器去配置文件，然后注册配置文件中配置的各个子应用后，首先加载主应用(菜单等)，**再通过路由判定，动态远程加载子应用（路由分发应用）**。
<!-- more -->
**single-spa能做什么：**

1. 同一页面使用多个框架而不需要刷新页面
2. 各模块独立部署
3. 使用新的框架编写代码，而不需要重写现有的应用程序

### single-spa 基本使用

准备一个single-spa-config.js，在顶层index.html引用。用single-spa-vue等框架改写各个子应用的打包入口文件。没错就是这么简单~

#### single-spa-config（全局配置文件）
用于在SPA中注册应用
注册方式：**registerApplication**函数
参数包含：

1. name
2. loadingFunction(加载的入口，( ) => <Function | Promise>)
3. activityFunction(何时加载，(location) => boolean)
4. customProps(可选，Object，在包装器生命周期函数中传递给子应用的props)

```javascript
// single-spa-config.js
import { registerApplication, start } from 'single-spa';
registerApplication("applicationName", loadingFunction, activityFunction);
start();
function loadingFunction() {
  return import("src/react/main.js");
}
function activityFunction(location) {
  return location.pathname.indexOf("/react/") === 0;
}
// 实际配置的话可以这么写
registerApplication(
  'vue', 
  () => System.import('./src/vue/vue.app.js'),
  () => location.pathname.startsWith("/react")
);
```
注：

1. start函数必须在子应用加载完后才能调用。在调用之前，子应用已经加载了只是未被渲染。

2. 由于System.import('xxx') 这种方式，获得是个promise对象，所以检测所需模块是否都引入完全，使用Promise.all方法

3. 当我们的浏览器url的前缀有/react的时候,程序就可以正常渲染这个应用，所以我们这个react应用的所有路由前缀都得有/react

*这里说一下第二个参数，也可以是一个包含了模块项目生命周期方法的对象。比较下边第三点模块项目入口文件，实际上需要的就是这些方法。可能没有对应包装库的时候才需要这么做。*

```javascript
const application = {
  bootstrap: () => Promise.resolve(), //bootstrap function
  mount: () => Promise.resolve(), //mount function
  unmount: () => Promise.resolve(), //unmount function
}
registerApplication('applicatonName', application, activityFunction)
```

#### 模块项目需要做的处理

子模块需要包装，single-spa提供了一些库，比如single-spa-vue是针对vue项目的包装器，类似还有single-spa-react等

还需安装systemjs-webpack-interop帮助您使webpack和systemjs一起正常工作

修改子模块入口文件如下（以vue和react项目为例，其余参考官方文档Ecosystem条目）

```javascript
//vue项目
import Vue from 'vue';
import App from './App.vue';
import router from './router';
import singleSpaVue from 'single-spa-vue';
const vueLifecycles = singleSpaVue({
  Vue,
  appOptions: {
    el: '#app', //建议写一个函数确保有这个div，否则自动创建一个div并作为默认容器
    render: h => h(App),
    router,
  },
});
export const bootstrap = vueLifecycles.bootstrap;
export function mount(props) {
    createDomElement(); //确保有容器div，与react中domElementGetter作用一致
    return vueLifecycles.mount(props);
}
export const unmount = vueLifecycles.unmount;

//react项目
import React from 'react';
import ReactDOM from 'react-dom';
import rootComponent from './path-to-root-component.js';
// Note that SingleSpaContext is a react@16.3 (if available) context that provides the singleSpa props
import singleSpaReact, {SingleSpaContext} from 'single-spa-react';
const reactLifecycles = singleSpaReact({
  React,
  ReactDOM,
  rootComponent, // 根组件
  domElementGetter,// 对应挂载div
});
function domElementGetter() {
	// Make sure there is a div for us to render into
	let el = document.getElementById('app1');
	if (!el) {
		el = document.createElement('div');
		el.id = 'app1';
		document.body.appendChild(el);
	}
	return el;
}
export const bootstrap = reactLifecycles.bootstrap;
export const mount = reactLifecycles.mount;
export const unmount = reactLifecycles.unmount;
```

### demo：

基于react和vue的微前端实现，子项目在不同端口独立部署

项目结构示例：

```
├── projectMain                // 公共模块项目，
    ├── index.html            //  页面的总入口
    ├── main.js
    ├── single-spa-config.js   // 配置文件
    ├── webpack.config.js
├── projectA                // 子项目
    ├── app.js                 //子项目的页面入口
    ├── single-spa-entry.js   // 子项目打包入口
    ├── webpack.config.js.
    
```



主模块single-spa-config.js

```javascript
import * as singleSpa from 'single-spa';

async function loadApp(name, prefix, appURL, customProps) {
  // register the app with singleSPA and pass a reference to the store of the app as well as a reference to the globalEventDistributor
  singleSpa.registerApplication(name, () => SystemJS.import(appURL), () => location.hash.startsWith(`#${prefix}`), customProps);
}

async function init() {
  const loadingPromises = [];

  // app1: The URL "/react/..." is being redirected to "http://localhost:8001/..." this is done by the webpack proxy (webpack.config.js)
  loadingPromises.push(loadApp('react-app', '/react', '/react/singleSpaEntry.js'));

  // app2: The URL "/vue/..." is being redirected to "http://localhost:8002/..." this is done by the webpack proxy (webpack.config.js)
  loadingPromises.push(loadApp('vue-app', '/vue', '/vue/singleSpaEntry.js'));

  // wait until all stores are loaded and all apps are registered with singleSpa
  await Promise.all(loadingPromises);
  singleSpa.start();
}

init();
```

这个例子加载子模块用的是web-dev-sever

主模块 Webpack.config.js修改：

```javascript
// Proxy config for development purposes. In production, you would configure you webserver to do something similar.
proxy: {
	"/react": {
	 	target: "http://localhost:8001",
		pathRewrite: {"^/react" : ""}
	},
	"/vue": {
		target: "http://localhost:8002",
		pathRewrite: {"^/vue" : ""}
	},
}
```

子模块Webpack.config.js：

主要是改写入口，还有一些路径

```javascript
entry: {
	singleSpaEntry: './src/singleSpaEntry.js',
},
```

这样就可以实现部署在本地不同端口的项目在一个页面切换了。

### single-spa-vue做了什么？

很轻量的一个文件，贴一下部分源码：仅仅看一下mount

```javascript
const defaultOpts = {
  // required opts
  Vue: null,
  appOptions: null,
  template: null,
}

export default function singleSpaVue(userOpts) {
  const opts = {
    ...defaultOpts,
    ...userOpts,
  };
  let mountedInstances = {};
  return {
    bootstrap: bootstrap.bind(null, opts, mountedInstances),
    mount: mount.bind(null, opts, mountedInstances),
    unmount: unmount.bind(null, opts, mountedInstances),
    update: update.bind(null, opts, mountedInstances),
  };
}

function mount(opts, mountedInstances, props) {
  return Promise
    .resolve()
    .then(() => {
      const appOptions = {...opts.appOptions}
      if (props.domElement && !appOptions.el) {
        appOptions.el = props.domElement;
      }

      if (!appOptions.el) {
        // 为了便于阅读删掉了代码，这里就是如果没有容器就创建一个容器。
        mountedInstances.domEl = domEl
      }
			//  容错处理
      if (!appOptions.render && !appOptions.template && opts.rootComponent) {
        appOptions.render = (h) => h(opts.rootComponent)
      }  
      if (!appOptions.data) {
        appOptions.data = {}
      }

      appOptions.data = {...appOptions.data, ...props}

      mountedInstances.instance = new opts.Vue(appOptions);  // 实例化vue对象
      if (mountedInstances.instance.bind) {
        mountedInstances.instance = mountedInstances.instance.bind(mountedInstances.instance);
      }
    })
}

```
个人理解就是传出根据入参形成的四个生命周期方法，也就是"包装"你传入的框架和配置。这样就抹平了各种框架之间的差异。

single-spa框架将会接收这些生命周期方法，在需要的时候调用，根据路由变化来进行挂载，卸载等操作。

### single-spa做了什么

跟着源代码走一遍。下文都是列举部分代码

**首先看registerApplication内部**：将所有app信息都存在一个数组里

```javascript
  apps.push({
    loadErrorTime: null,
    name: appName,
    loadImpl,
    activeWhen: activityFn,
    status: NOT_LOADED,// app状态，用来筛选需要操作的app
    parcels: {},
    devtools: {
      overlays: {
        options: {},
        selectors: [],
      }
    },
    customProps
  });
	ensureJQuerySupport();
  reroute();  // 这一步的reroute是做什么的？

```

注册之后，我们是用**singleSpa.start()**来启动的，这个函数很简单，就是调用一下reroute函数

```javascript
let started = false;
export function start() {
  started = true;
  reroute();
}
```

**再来看reroute**

```javascript
let appChangeUnderway = false, peopleWaitingOnAppChange = [];
export function reroute(pendingPromises = [], eventArguments) {
  // appChangeUnderway默认为false，下一步置为true，等到change完毕了，重新置为false
  // 如果目前仍在进行appchange，就加入等待队列
  if (appChangeUnderway) {
    return new Promise((resolve, reject) => {
      peopleWaitingOnAppChange.push({
        resolve,
        reject,
        eventArguments,// 地址栏参数
      });
    });
  }

  appChangeUnderway = true;
  let wasNoOp = true;

  if (isStarted()) {
    return performAppChanges();
  } else {
    return loadApps();
  }
```

**loadApps()** 

```javascript
function loadApps() {
    return Promise.resolve().then(() => {
      const loadPromises = getAppsToLoad().map(toLoadPromise);

      if (loadPromises.length > 0) {
        wasNoOp = false;
      }

      return Promise
        .all(loadPromises)
        .then(finishUpAndReturn)
        .catch(err => {
          callAllEventListeners();
          throw err;
        })
    })
  }
// toLoadPromise里的主要操作，获取子项目的钩子
app.status = NOT_BOOTSTRAPPED;
app.bootstrap = flattenFnArray(appOpts.bootstrap, `App '${app.name}' bootstrap function`);
app.mount = flattenFnArray(appOpts.mount, `App '${app.name}' mount function`);
app.unmount = flattenFnArray(appOpts.unmount, `App '${app.name}' unmount function`);
// unload可选
app.unload = flattenFnArray(appOpts.unload || [], `App '${app.name}' unload function`);
app.timeouts = ensureValidAppTimeouts(appOpts.timeouts)
```

**performAppChanges()**

路由分发应用关键操作就在这一步进行

```javascript
  function performAppChanges() {
    return Promise.resolve().then(() => {
      window.dispatchEvent(new CustomEvent("single-spa:before-routing-event", getCustomEventDetail()));
      // 首先需要unload,unmount之前的app
      const unloadPromises = getAppsToUnload().map(toUnloadPromise);

      const unmountUnloadPromises = getAppsToUnmount()
        .map(toUnmountPromise)
        .map(unmountPromise => unmountPromise.then(toUnloadPromise)); // 将

      const allUnmountPromises = unmountUnloadPromises.concat(unloadPromises);
      if (allUnmountPromises.length > 0) {
        wasNoOp = false;
      }

      const unmountAllPromise = Promise.all(allUnmountPromises);

      const appsToLoad = getAppsToLoad();

      /* 在其他app还在unmount的时候可以load和bootstrap现在的app，但是只有等其他app unmount完毕
       * 之后才会mount现在的app 
       */
      const loadThenMountPromises = appsToLoad.map(app => {
        return toLoadPromise(app)
          .then(toBootstrapPromise)
          .then(app => {
            return unmountAllPromise
              .then(() => toMountPromise(app))
          })
      })
      if (loadThenMountPromises.length > 0) {
        wasNoOp = false;
      }

      /* These are the apps that are already bootstrapped and just need
       * to be mounted. They each wait for all unmounting apps to finish up
       * before they mount.
       */
      const mountPromises = getAppsToMount()
        .filter(appToMount => appsToLoad.indexOf(appToMount) < 0)
        .map(appToMount => {
          return toBootstrapPromise(appToMount)
            .then(() => unmountAllPromise)
            .then(() => toMountPromise(appToMount))
        })
      if (mountPromises.length > 0) {
        wasNoOp = false;
      }
      return unmountAllPromise
        .catch(err => {
          callAllEventListeners();
          throw err;
        })
        .then(() => {
          /* 其他的app在被unmount之后，他们的事件(比如hashchange or popstate)也应该已经被清除. 
           * 这对其余捕获的事件侦听器处理DOM事件是安全的
           */
          callAllEventListeners(); // 唤醒

          return Promise
            .all(loadThenMountPromises.concat(mountPromises))
            .catch(err => {
              pendingPromises.forEach(promise => promise.reject(err));
              throw err;
            })
            .then(() => finishUpAndReturn(false))
        })

    })
  }
```

之后每次监听到hashchange和popstate事件,都会调用一遍reroute。

最后finishUpAndReturn：

结束当前app的操作并检查等待队列

```javascript
  function finishUpAndReturn(callEventListeners=true) {
    const returnValue = getMountedApps();

    if (callEventListeners) {
      callAllEventListeners();
    }
    pendingPromises.forEach(promise => promise.resolve(returnValue));

    try {
      const appChangeEventName = wasNoOp ? "single-spa:no-app-change": "single-spa:app-change";
      window.dispatchEvent(new CustomEvent(appChangeEventName, getCustomEventDetail()));
      window.dispatchEvent(new CustomEvent("single-spa:routing-event", getCustomEventDetail()));
    } catch (err) {
      /* We use a setTimeout because if someone else's event handler throws an error, single-spa
       * needs to carry on. If a listener to the event throws an error, it's their own fault, not
       * single-spa's.
       */
      setTimeout(() => {
        throw err;
      });
    }

    /* Setting this allows for subsequent calls to reroute() to actually perform
     * a reroute instead of just getting queued behind the current reroute call.
     * We want to do this after the mounting/unmounting is done but before we
     * resolve the promise for the `reroute` function.
     */
    appChangeUnderway = false;

    if (peopleWaitingOnAppChange.length > 0) {
      // 将等待队列中的放入待执行，然后
      const nextPendingPromises = peopleWaitingOnAppChange;
      // 等待队列清空
      peopleWaitingOnAppChange = [];
      reroute(nextPendingPromises);
    }

    return returnValue;
  }
```

总结一下single-spa的工作流程：

首先registerApplication函数将所有app注册到一个数组中，之后从这个数组中筛选出符合操作条件的app(Array.filter)，然后通过toMountPromise等函数转换成成相应的promise（调用子项目调用的mount，unmount等钩子）。

参考：

https://single-spa.js.org/ 官网

https://alili.tech/archive/11052bf4/ 

https://www.jianshu.com/p/54904acb5896

https://github.com/me-12/single-spa-portal-example  相对完整的多技术栈应用demo

