![vue](https://upload-images.jianshu.io/upload_images/12890819-7820bc20092c4c40.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**前言**

本文内容讲解的内容：**一张思维导图辅助你深入了解 Vue | Vue-Router | Vuex 源码架构**。

项目地址：[https://github.com/biaochenxuying/vue-family-mindmap](https://github.com/biaochenxuying/vue-family-mindmap)

[markdown 文字版](https://github.com/biaochenxuying/vue-family-mindmap/blob/master/Vue-family.md)

[pdf 版](https://github.com/biaochenxuying/vue-family-mindmap/blob/master/Vue-family.pdf)

先来张 Vue 全家桶 总图：

![](https://upload-images.jianshu.io/upload_images/12890819-f7a2788fbc61e68a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 1. 项目目录 

![](https://upload-images.jianshu.io/upload_images/12890819-41efc5a9eec040f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

scripts: 构建相关的文件，一般情况下我们不需要动。

- git-hooks：存放git钩子的目录
- alias.js：别名配置
- config.js：生成rollup配置的文件
- build.js：对 config.js 中所有的rollup配置进行构建
- ci.sh：持续集成运行的脚本
- release.sh： 用于自动发布新版本的脚本

dist: 构建后文件的输出目录

examples: 存放一些使用Vue开发的应用案例

flow: 类型声明，使用开源项目 [Flow]

packages: 存放独立发布的包的目录

test: 包含所有测试文件

src: 源码,重点

- compiler: 编译器代码的存放目录，将 template 编译为 render 函数
- core: 核心代码 ，与平台无关的代码

	- observer： 响应系统，包含数据观测的核心代码
	- vdom：包含虚拟DOM创建(creation)和打补丁(patching)的代码
	- instance：包含Vue构造函数设计相关的代码
	- global-api：包含给Vue构造函数挂载全局方法(静态方法)或属性的代码
	- components：包含抽象出来的通用组件

- platforms: 不同平台的支持，包含平台特有的相关代码，不同平台的不同构建的入口文件也在这里

	- web：web平台

		- entry-runtime.js：运行时构建的入口，不包含模板(template)到render函数的编译器，所以不支持 `template` 选项，我们使用vue默认导出的就是这个运行时的版本。大家使用的时候要注意
		- entry-runtime-with-compiler.js：独立构建版本的入口，它在 entry-runtime 的基础上添加了模板(template)到render函数的编译器
		- entry-compiler.js：vue-template-compiler 包的入口文件
		- entry-server-renderer.js：vue-server-renderer 包的入口文件
		- entry-server-basic-renderer.js：输出 packages/vue-server-renderer/basic.js 文件

	- weex：混合应用

- serve: 服务端渲染，包含(server-side rendering)的相关代码
- sfc: 包含单文件组件( .vue 文件)的解析逻辑，用于vue-template-compiler包
- shared: 共享代码，包含整个代码库通用的代码

package.json：对项目的描述文件，包含了依赖包等信息

yarn.lock ：yarn 锁定文件

.editorconfig：针对编辑器的编码风格配置文件

.flowconfig：flow 的配置文件

.babelrc：babel 配置文件

.eslintrc：eslint 配置文件

.eslintignore：eslint 忽略配置

.gitignore：git 忽略配置


## 2. 源码构建，基于 Rollup 

![](https://upload-images.jianshu.io/upload_images/12890819-dfc52768e1942719.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**1. 根据 format 构建格式可分为三个版（再根据有无 compiler ，每个版本中又可以再分出二个版本）**

- cjs：表示构建出来的文件遵循 CommonJS 规范

	- Runtime Only 
	- Runtime + Compiler

- es：构建出来的文件遵循 ES Module 规范

	- Runtime Only 
	- Runtime + Compiler

- umd：构建出来的文件遵循 UMD 规范

	- Runtime Only 
	- Runtime + Compiler

**2. 总结**

- Runtime Only：通常需要借助如 webpack 的 vue-loader 工具把 .vue 文件编译成JavaScript，因为是在编译阶段做的，所以它只包含运行时的 Vue.js 代码，因此代码体积也会更轻量。             
Runtime + Compiler：我们如果没有对代码做预编译，但又使用了 Vue 的 template 属性并传入一个字符串，则需要在客户端编译模板。Vue.js 2.0 中，最终渲染都是通过 render 函数，如果写 template 属性，则需要编译成 render 函数，那么这个编译过程会发生运行时，所以需要带有编译器的版本。

## 3. vue 本质：构造函数

![](https://upload-images.jianshu.io/upload_images/12890819-712e7ffb25339677.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
function Vue (options) {
  if (process.env.NODE_ENV !== production' && !(this instanceof Vue) ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
```
**总结**

- vue 本质上就是一个用 Function 实现的 Class，然后在它的原型 prototype 以及它本身上扩展了一系列的方法和属性。                                      
- Vue 不用 ES6 的 Class 去实现的原因：按功能区分，把功能扩展分散到多个模块中去实现，然后挂载中 vue 的原型 prototype 上，也有在 Vue 这个对象本身上。                                                       
- 而不是在一个模块里实现所有，这种方式是用 Class 难以实现的。这么做的好处是非常方便代码的维护和管理。

## 4. 数据驱动

![](https://upload-images.jianshu.io/upload_images/12890819-d76f2eaddae2f07a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**1.  new Vue**

```
var app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!'
  }
})
```

**2. init**

- 调用 this._init(options) 进行初始化

	- mergeOptions 合并配置
	- initLifecycle(vm) 初始化生命周期，调用生命周期钩子函数 callHook(vm, 'beforeCreate')
	- initEvents(vm) 初始化事件中心
	- initRender(vm) 初始化渲染
	- 初始化 data、props、computed、watcher 等等

**3. Vue 实例挂载 $mount**

- $mount 这个方法的实现是和平台、构建方式都相关的。我们分析带 compiler 版本的 $mount 实现。在 Vue 2.0 版本中，所有 Vue 的组件最终都会转换成 render 方法。

	- 1. 它对 el 做了限制，Vue 不能挂载在 body、html 这样的根节点上。
	- 2. 如果没有定义 render 方法，则会调用 compileToFunctions 方法把 el 或者 template 字符串转换成 render 方法。
	- 3. mountComponent：核心就是先实例化一个渲染Watcher，在它的回调函数中会调用 updateComponent 方法，在此方法中调用 vm._render 方法先生成虚拟 Node，最终调用 vm._update 更新 DOM。
	- 4. 将 vm._isMounted 设置为 true，表示已经挂载
	- 5. 执行 mounted 钩子函数：callHook(vm, 'mounted')

**4. compile**

- 在 Vue 2.0 版本中，所有 Vue 的组件的渲染最终都需要 render 方法，无论我们是用单文件 .vue 方式开发组件，还是写了 el 或者 template 属性，最终都会转换成 render 方法，那么这个过程是 Vue 的一个“在线编译”的过程，它是调用 compileToFunctions 方法实现的。

**5. render: Vue 的 _render 方法是实例的一个私有方法，最终会把实例渲染成一个虚拟 Node。**

- vm._render 最终是通过执行 createElement 方法并返回的是 vnode，它是一个虚拟 Node

**6. Virtual DOM（虚拟 dom）:  本质上是一个原生的 JS 对象，用 class 来定义。**

- 1. 核心定义：几个关键属性，标签名、数据、子节点、键值等，其它属性都是都是用来扩展 VNode 的灵活性以及实现一些特殊 feature 的。
- 2. 映射到真实的 DOM ，实际上要经历 VNode 的 create、diff、patch 等过程。
- 3. createElement： 创建 VNode

	- 1. children 的规范化：由于 Virtual DOM 实际上是一个树状结构，每一个 VNode 可能会有若干个子节点，这些子节点应该也是 VNode 的类型。因为子节点 children 是任意类型的，因此需要把它们规范成 VNode 类型。

		- 1. simpleNormalizeChildren：调用场景是 render 函数是编译生成的。
		- 2. normalizeChildren

			- 1. 一个场景是 render 函数是用户手写的，当 children 只有一个节点的时候，Vue.js 从接口层面允许用户把 children 写成基础类型用来创建单个简单的文本节点，这种情况会调用 createTextVNode 创建一个文本节点的 VNode。
			- 2. 另一个场景是当编译 slot、v-for 的时候会产生嵌套数组的情况，会调用 normalizeArrayChildren 方法，遍历 children (可能会递归调用 normalizeArrayChildren )。

		- 3. 总结

			- 经过对 children 的规范化，children 变成了一个类型为 VNode 的 Array

	- 2. VNode 的创建

		- 规范化 children 后，会去创建一个 VNode 的实例。

			- 1. 直接创建一个普通 VNode。
			- 2. 或者通过 createComponent 创建一个组件类型的 VNode，本质上它还是返回了一个 VNode。
			- 3. 总结

				- 每个 VNode 有 children，children 每个元素也是一个 VNode，这样就形成了一个 VNode Tree，它很好的描述了我们的 DOM Tree。

	- 3. update：通过 Vue 的 _update 方法，_update 方法的作用是把 VNode 渲染成真实的 DOM。_update 的核心就是调用 vm.__patch__ 方法，__patch__在不同的平台，比如 web 和 weex 上的定义是不一样的。

**7. update 的核心：调用 vm.__patch__ 方法**

- update：通过 Vue 的 _update 方法，_update 方法的作用是把 VNode 渲染成真实的 DOM。_update 的核心就是调用 vm.__patch__ 方法，__patch__在不同的平台，比如 web 和 weex 上的定义是不一样的。

	- 1. 首次渲染

		- 1. 通过 createElm 方法，把虚拟节点创建真实的 DOM 并插入到它的父节点中。
		- 2. 然后调用 createChildren 方法去创建子元素，实际上是遍历子虚拟节点，递归调用 createElm。
		- 3. 接着再调用 invokeCreateHooks 方法执行所有的 create 的钩子并把 vnode push 到 insertedVnodeQueue
		- 4. 最后调用 insert 方法把 DOM 插入到父节点中，因为是递归调用，子元素会优先调用 insert，所以整个 vnode 树节点的插入顺序是先子后父。
		- 5. 总结

			- 其实就是调用原生 DOM 的 API 进行 DOM 操作，Vue 就是这样动态创建的 DOM。

	- 2. 数据更新

**8. DOM：Vue 最终创建的 DOM。**

**9. 总结**

- 初始化 Vue 到最终渲染的整个过程：

**new Vue => init => $mounted => compile => render => vnode => patch => DOM**

## 5. 组件化

![](https://upload-images.jianshu.io/upload_images/12890819-1b0a5e99649978c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**1. introduction**

- 组件化，就是把页面拆分成多个组件 (component)，每个组件依赖的 CSS、JavaScript、模板、图片等资源放在一起开发和维护。组件是资源独立的，组件在系统内部可复用，组件和组件之间可以嵌套。

**2. createComponent**

- 在 createElement 的实现的时候，如果不是一个普通的 html 标签，就是通过 createComponent 方法创建一个组件 VNode。

	- 1. 构造子类构造函数

		- Vue.extend 函数

			- Vue.extend 的作用就是构造一个 Vue 的子类，它使用一种非常经典的原型继承的方式把一个纯对象转换一个继承于 Vue 的构造器 Sub 并返回，然后对 Sub 这个对象本身扩展了一些属性，如扩展 options、添加全局 API 等；并且对配置中的 props 和 computed 做了初始化工作；最后对于这个 Sub 构造函数做了缓存，避免多次执行 Vue.extend 的时候对同一个子组件重复构造。

				- 当我们去实例化 Sub 的时候，就会执行 this._init 逻辑再次走到了 Vue 实例的初始化逻辑。
				- 代码如下：
```
const Sub = function VueComponent (options) {
  this._init(options)
}
```

   - 2. 安装组件钩子函数：installComponentHooks(data)

		- installComponentHooks 的过程就是把 componentVNodeHooks 的钩子函数合并到 data.hook 中，在 VNode 执行 patch 的过程中执行相关的钩子函数
		- 这里要注意的是合并策略，在合并过程中，如果某个时机的钩子已经存在 data.hook 中，那么通过执行 mergeHook 函数做合并，这个逻辑很简单，就是在最终执行的时候，依次执行这两个钩子函数即可。

	- 3. 实例化 vnode

		- 通过 new VNode 实例化一个 vnode 并返回。需要注意的是和普通元素节点的 vnode 不同，组件的 vnode 是没有 children 的，这点很关键

	- 4. 总结

		- createComponent 后返回的是组件 vnode，它也一样走到 vm._update 方法，进而执行了 patch 函数。

**3. path**

- 一个组件的 VNode 是如何创建、初始化、渲染的过程

**4. 合并配置**

- 1. 外部调用场景

	- 外部我们的代码主动调用 new Vue(options) 的方式实例化一个 Vue 对象。

- 2. 组件场景

	- 上一节分析的组件过程中内部通过 new Vue(options) 实例化子组件。

- 3. 总结

	- 子组件初始化过程通过 initInternalComponent 方式要比外部初始化 Vue 通过 mergeOptions 的过程要快，合并完的结果保留在 vm.$options 中。

**5. 生命周期**

- 完整生命周期的讲解，请看 vue 官网: https://cn.vuejs.org/v2/guide/instance.html 。

> 注意：activated 和 deactivated 钩子函数是专门为 keep-alive 组件定制的钩子。

![](https://upload-images.jianshu.io/upload_images/12890819-73de368bb76528b7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**6. 组件注册**

- 1. 全局注册：Vue.component(tagName, options)
- 2. 局部注册
- 3. 总结

	- 注意，局部注册和全局注册不同的是，只有该类型的组件才可以访问局部注册的子组件，而全局注册是扩展到 Vue.options 下，所以在所有组件创建的过程中，都会从全局的 Vue.options.components 扩展到当前组件的 vm.$options.components 下，这就是全局注册的组件能被任意使用的原因。

**7. 异步组件**

- 1. 普通函数异步组件

代码
```
Vue.component('async-example', function (resolve, reject) {
   // 这个特殊的 require 语法告诉 webpack
   // 自动将编译后的代码分割成不同的块，
   // 这些块将通过 Ajax 请求自动下载。
   require(['./my-async-component'], resolve)
})

```

- 2. Promise 异步组件

代码

```
Vue.component(
  'async-webpack-example',
  // 该 `import` 函数返回一个 `Promise` 对象。
  () => import('./my-async-component')
)
```

- 3. 高级异步组件

代码

```
const AsyncComp = () => ({
  // 需要加载的组件。应当是一个 Promise
  component: import('./MyComp.vue'),
  // 加载中应当渲染的组件
  loading: LoadingComp,
  // 出错时渲染的组件
  error: ErrorComp,
  // 渲染加载中组件前的等待时间。默认：200ms。
  delay: 200,
  // 最长等待时间。超出此时间则渲染错误组件。默认：Infinity
  timeout: 3000
})
Vue.component('async-example', AsyncComp)
```

- 4. 总结

异步组件实现的本质是 2 次渲染，除了 0 delay 的高级异步组件第一次直接渲染成 loading 组件外，其它都是第一次渲染生成一个注释节点，当异步获取组件成功后，再通过 forceRender 强制重新渲染，这样就能正确渲染出我们异步加载的组件了。

## 6. 深入响应式原理

![](https://upload-images.jianshu.io/upload_images/12890819-f7d67d439ddf2acd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**1. 响应式对象：Vue.js 实现响应式的核心是利用了 ES5 的 Object.defineProperty。**

- 1. Object.defineProperty

直接在一个对象上定义一个新属性，或者修改一个对象的现有属性

- 2. initState：在 Vue 的初始化阶段，_init 方法执行的时候，会执行 initState(vm) 方法

	- 主要是对 props、methods、data、computed 和 wathcer 等属性做了初始化操作

		- 1. initProps：props 的初始化主要过程，就是遍历定义的 props 配置

			- 1. 一个是调用 defineReactive 方法把每个 prop 对应的值变成响应式，可以通过 vm._props.xxx 访问到定义 props 中对应的属性。
			- 2. 通过 proxy 把 vm._props.xxx 的访问代理到 vm.xxx 上

		- 2. initData

			- 1. 一个是对定义 data 函数返回对象的遍历，通过 proxy 把每一个值 vm._data.xxx 都代理到 vm.xxx 上；
			- 2. 另一个是调用 observe 方法观测整个 data 的变化，把 data 也变成响应式，可以通过 vm._data.xxx 访问到定义 data 返回函数中对应的属性

		- 3. proxy：代理的作用是把 props 和 data 上的属性代理到 vm 实例上

			- proxy 方法的实现很简单，通过 Object.defineProperty 把 target[sourceKey][key] 的读写变成了对 target[key] 的读写。

				- 比如 data ，对 vm._data.xxxx 的读写变成了对 vm.xxxx 的读写。

		- 4. 总结

			- 无论是 props 或是 data 的初始化都是把它们变成响应式对象

- 3. observe ：功能就是用来监测数据的变化

	- observe 方法的作用就是给非 VNode 的对象类型数据添加一个 Observer，如果已经添加过则直接返回，否则在满足一定条件下去实例化一个 Observer 对象实例

- 4. Observer 是一个类，它的作用是给对象的属性添加 getter 和 setter，用于依赖收集和派发更新
- 5. defineReactive： 功能就是定义一个响应式对象，给对象动态添加 getter 和 setter。
- 6. 总结

	- 响应式对象，核心就是利用 Object.defineProperty 给数据添加了 getter 和 setter，目的就是为了在我们访问数据以及写数据的时候能自动执行一些逻辑：getter 做的事情是依赖收集，setter 做的事情是派发更新

**2. 依赖收集：响应式对象 getter 相关的逻辑就是做依赖收集**

- 1. Dep：整个 getter 依赖收集的核心

	- Dep 实际上就是对 Watcher 的一种管理。而且在同一时间只能有一个全局的 Watcher 被计算

- 2. Watcher

	- Watcher 是一个 Class，定义了一些和 Dep 相关的属性， 还定义了一些原型的方法，和依赖收集相关的有 get、addDep 和 cleanupDeps 方法。
	- 总结

		- 在添加 deps 的订阅过程，可以通过 id 去重避免重复订阅。在每次添加完新的订阅，会移除掉旧的订阅

- 3. 总结

	- 收集依赖就是订阅数据变化的 watcher 的收集。收集依赖的目的是为了当这些响应式数据发生变化，触发它们的 setter 的时候，能知道应该通知哪些订阅者去做相应的逻辑处理，我们把这个过程叫派发更新，其实 Watcher 和 Dep 就是一个非常经典的观察者设计模式的实现

**3. 派发更新**

- 修改值的时候，会触发 setter ，会对新设置的值变成一个响应式对象，并通过 dep.notify() 通知所有的订阅者

	- 做派发更新的时候的一个优化的点，它并不会每次数据改变都触发 watcher 的回调，而是把这些 watcher 先添加到一个队列里，然后在 nextTick 后执行 flushSchedulerQueue
	- 总结

		- 当数据发生变化的时候，触发 setter 逻辑，把在依赖过程中订阅的的所有观察者，也就是 watcher，都触发它们的 update 过程，这个过程又利用了队列做了进一步优化，在 nextTick 后执行所有 watcher 的 run，最后执行它们的回调函数。

**4. nextTick**

- 1. JS 运行机制

	- JS 执行是单线程的，它是基于事件循环的。事件循环大致分为以下几个步骤

		- 1. 所有同步任务都在主线程上执行，形成一个执行栈（execution context stack）。
		- 2. 主线程之外，还存在一个"任务队列"（task queue）。只要异步任务有了运行结果，就在"任务队列"之中放置一个事件。
		- 3.  一旦"执行栈"中的所有同步任务执行完毕，系统就会读取"任务队列"，看看里面有哪些事件。那些对应的异步任务，于是结束等待状态，进入执行栈，开始执行。
		- 4. 主线程不断重复上面的第三步。
		- 5. 代码演示 macro task 和 micro task 执行顺序

```
for (macroTask of macroTaskQueue) {
    // 1. Handle current MACRO-TASK
    handleMacroTask();
      
    // 2. Handle all MICRO-TASK
    for (microTask of microTaskQueue) {
        handleMicroTask(microTask);
    }
}

// 在浏览器环境中，常见的 macro task 有 setTimeout、MessageChannel、postMessage、setImmediate；
// 常见的 micro task 有 MutationObsever 和 Promise.then。
```

- 6. 总结

主线程的执行过程就是一个 tick，而所有的异步结果都是通过 “任务队列” 来调度。 消息队列中存放的是一个个的任务（task）。 规范中规定 task 分为两大类，分别是 macro task 和 micro task，并且每个 macro task 结束后，都要清空所有的 micro task。

- vue 中 nextTick 实现

	- 1. 申明了 microTimerFunc 和 macroTimerFunc 2 个变量，它们分别对应的是 micro task 的函数和 macro task 的函数。
	- 2. 对于 macro task 的实现，优先检测是否支持原生 setImmediate，这是一个高版本 IE 和 Edge 才支持的特性，不支持的话再去检测是否支持原生的 MessageChannel，如果也不支持的话就会降级为 setTimeout 0；
	- 3. 而对于 micro task 的实现，则检测浏览器是否原生支持 Promise，不支持的话直接指向 macro task 的实现。
	- 4. nextTick 把传入的回调函数 cb 压入 callbacks 数组，最后一次性地根据 useMacroTask 条件执行 macroTimerFunc 或者是 microTimerFunc，而它们都会在下一个 tick 执行 flushCallbacks，flushCallbacks 的逻辑非常简单，对 callbacks 遍历，然后执行相应的回调函数。

这里使用 callbacks 而不是直接在 nextTick 中执行回调函数的原因是保证在同一个 tick 内多次执行 nextTick，不会开启多个异步任务，而把这些异步任务都压成一个同步任务，在下一个 tick 执行完毕。

	- 5. next-tick.js 还对外暴露了 withMacroTask 函数，它是对函数做一层包装，确保函数执行过程中对数据任意的修改，触发变化执行 nextTick 的时候强制走 macroTimerFunc。比如对于一些 DOM 交互事件，如 v-on 绑定的事件回调函数的处理，会强制走 macro task。
	- 6. 总结

对 nextTick 的分析，并结合上一节的 setter 分析，我们了解到数据的变化到 DOM 的重新渲染是一个异步过程，发生在下一个 tick。这就是我们平时在开发的过程中，比如从服务端接口去获取数据的时候，数据做了修改，如果我们的某些方法去依赖了数据修改后的 DOM 变化，我们就必须在 nextTick 后执行。

Vue.js 提供了 2 种调用 nextTick 的方式，一种是全局 API Vue.nextTick，一种是实例上的方法 vm.$nextTick，无论我们使用哪一种，最后都是调用 next-tick.js 中实现的 nextTick 方法。

**5. 检测变化的注意事项**

- 1. 对象添加属性

对于使用 Object.defineProperty 实现响应式的对象，当我们去给这个对象添加一个新的属性的时候，是不能够触发它的 setter 的

```
var vm = new Vue({
  data:{
    a:1
  }
})
// vm.b 是非响应的
vm.b = 2
```

要用 Vue.set 方法

set 方法是在对象上设置属性。添加新属性和如果属性不存在，通过 defineReactive(ob.value, key, val) 把新添加的属性变成响应式对象，然后再通过 ob.dep.notify() 手动的触发依赖通知。

- 2. 数组

	- Vue 也是不能检测到以下变动的数组

		- 1. 当你利用索引直接设置一个项时，例如：vm.items[indexOfItem] = newValue

			- 可以使用：Vue.set(example1.items, indexOfItem, newValue)

		- 2. 当你修改数组的长度时，例如：vm.items.length = newLength

			- 可以使用 vm.items.splice(newLength)

	- 总结

		- vue 通过 arrayMethods 继承了 Array，然后对数组中所有能改变数组自身的方法，如 push、pop 等这些方法进行重写，重写后的方法会先执行它们本身原有的逻辑，并对能增加数组长度的 3 个方法 push、unshift、splice 方法做了判断，获取到插入的值，然后把新添加的值变成一个响应式对象，并且再调用 ob.dep.notify() 手动触发依赖通知，这就很好地解释了之前的示例中调用 vm.items.splice(newLength) 方法可以检测到变化。

**6. 计算属性 VS 侦听属性**

- 1. computd

	- 计算属性本质上就是一个 computed watcher，确保不仅仅是计算属性依赖的值发生变化，而是当计算属性最终计算的值发生变化才会触发渲染 watcher 重新渲染。

		- computed watcher

- 2. watch

	- 本质上侦听属性也是基于 Watcher 实现的，它是一个 user watcher

		- deep watcher
		- user watcher

			- 通过 vm.$watch 创建的 watcher 是一个 user watcher，其实它的功能很简单，在对 watcher 求值以及在执行回调函数的时候，会处理一下错误

		- computed watcher
		- sync watcher

- 3. 总结

	- 计算属性本质上是 computed watcher，而侦听属性本质上是 user watcher。就应用场景而言，计算属性适合用在模板渲染中，某个值是依赖了其它的响应式对象甚至是计算属性计算而来；而侦听属性适用于观测某个值的变化去完成一段复杂的业务逻辑。

**7. 组件更新：过程的核心就是新旧 vnode diff，对新旧节点相同以及不同的情况分别做不同的处理。**

- 1. 新旧节点不同

	- 新旧 vnode 不同，本质上是要替换已存在的节点。

		- 1. 创建新节点

			- 以当前旧节点为参考节点，创建新的节点，并插入到 DOM 中

		- 2. 更新父的占位符节点
		- 3. 删除旧节点

			- 删除节点就是遍历待删除的 vnodes 做删除

- 2. 新旧节点相同
- 3. updateChildren

## 7. 编译

![](https://upload-images.jianshu.io/upload_images/12890819-2257b1dd6c529899.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**1. introduction**

- 模板到真实 DOM 渲染的过程，中间有一个环节是把模板编译成 render 函数，这个过程我们把它称作编译。

**2. 编译入口**

- mount 的时候，通过 compileToFunctions 方法就是把模板 template 编译生成 render 以及 staticRenderFns

	- 1. 解析模板字符串生成 AST

		- const ast = parse(template.trim(), options)

	- 2. 优化语法树

		- optimize(ast, options)

	- 3. 生成代码

		- const code = generate(ast, options)

	- 4. 总结

		- 编译入口逻辑之所以这么绕，是因为 Vue.js 在不同的平台下都会有编译的过程，因此编译过程中的依赖的配置 baseOptions 会有所不同。而编译过程会多次执行，但这同一个平台下每一次的编译过程配置又是相同的，为了不让这些配置在每次编译过程都通过参数传入，Vue.js 利用了函数柯里化的技巧很好的实现了 baseOptions 的参数保留。同样，Vue.js 也是利用函数柯里化技巧把基础的编译过程函数抽出来，通过 createCompilerCreator(baseCompile) 的方式把真正编译的过程和其它逻辑如对编译配置处理、缓存处理等剥离开，这样的设计还是非常巧妙的。

**3. parse**

- 编译过程首先就是对模板做解析，生成 AST，它是一种抽象语法树，是对源代码的抽象语法结构的树状表现形式。在很多编译技术中，如 babel 编译 ES6 的代码都会先生成 AST。

	- 整体流程

		- 1. 从 options 中获取方法和配置, 如伪代码 getFnsAndConfigFromOptions(options)

			- 这些属性和方法之所以放到 platforms 目录下是因为它们在不同的平台（web 和 weex）的实现是不同的。

		- 2. 解析 HTML 模板， 对应伪代码  parseHTML(template, options)

			- 整体来说它的逻辑就是循环解析 template ，用正则做各种匹配，对于不同情况分别进行不同的处理，直到整个 template 被解析完毕。 在匹配的过程中会利用 advance 函数不断前进整个模板字符串，直到字符串末尾。

				- 匹配的过程中主要利用了正则表达式，通过一系列正则表达式，可以匹配注释节点、文档类型节点、文本、开始标签、闭合标签等。

		- 3. 处理开始标签

			- 1. 创建 AST 元素
			- 2. 处理 AST 元素

				- 这过程会判断 element 是否包含各种指令通过 processXXX 做相应的处理，处理的结果就是扩展 AST 元素的属性。比如 v-for、v-if 指令。

			- 3. AST 树管理

				- 在处理开始标签的时候为每一个标签创建了一个 AST 元素，在不断解析模板创建 AST 元素的时候，我们也要为它们建立父子关系，就像 DOM 元素的父子关系那样。

					- AST 树管理的目标是构建一颗 AST 树，本质上它要维护 root 根节点和当前父节点 currentParent。为了保证元素可以正确闭合，这里也利用了 stack 栈的数据结构，和我们之前解析模板时用到的 stack 类似。

		- 4. 处理闭合标签

			- 对应伪代码：
end () {
  treeManagement()
  closeElement()
}

		- 5. 处理文本内容

			- 对应伪代码：
chars (text: string) {
  handleText()
  createChildrenASTOfText()
}

		- 6. 总结

			- parse 的目标是把 template 模板字符串转换成 AST 树，它是一种用 JavaScript 对象的形式来描述整个模板。那么整个 parse 的过程是利用正则表达式顺序解析模板，当解析到开始标签、闭合标签、文本的时候都会分别执行对应的回调函数，来达到构造 AST 树的目的。                    AST 元素节点总共有 3 种类型，type 为 1 表示是普通元素，为 2 表示是表达式，为 3 表示是纯文本。其实这里我觉得源码写的不够友好，这种是典型的魔术数字，如果转换成用常量表达会更利于源码阅读。
			- 当 AST 树构造完毕，下一步就是 optimize 优化这颗树。

**4. optimize**

- 当我们的模板 template 经过 parse 过程后，会输出生成 AST 树，那么接下来我们需要对这颗树做优化，Vue 是数据驱动，是响应式的，但是我们的模板并不是所有数据都是响应式的，也有很多数据是首次渲染后就永远不会变化的，那么这部分数据生成的 DOM 也不会变化，我们可以在 patch 的过程跳过对他们的比对。

	- 1. 标记静态节点 markStatic(root)
	- 2. 标记静态根 markStaticRoots(root, false)

- 总结

	- optimize 的过程，就是深度遍历这个 AST 树，去检测它的每一颗子树是不是静态节点，如果是静态节点则它们生成 DOM 永远不需要改变，这对运行时对模板的更新起到极大的优化作用。

**5. codegen**

- 编译的最后一步就是把优化后的 AST 树转换成可执行的代码

例子：
```
<ul :class="bindCls" class="list" v-if="isShow">
    <li v-for="(item,index) in data" @click="clickItem(index)">{{item}}:{{index}}</li>
</ul>
```

- 它经过编译，执行 const code = generate(ast, options)，生成的 render 代码串如下：

```
with(this){
  return (isShow) ?
    _c('ul', {
        staticClass: "list",
        class: bindCls
      },
      _l((data), function(item, index) {
        return _c('li', {
          on: {
            "click": function($event) {
              clickItem(index)
            }
          }
        },
        [_v(_s(item) + ":" + _s(index))])
      })
    ) : _e()
}
```

- codegen 的目标是把 AST 树转换成代码字符串，整个 codegen 过程就是深度遍历 AST 树，根据不同条件生成不同代码的过程。

## 8. 扩展

![](https://upload-images.jianshu.io/upload_images/12890819-228d9b7ad530fff2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**1. event**

- 1. 编译

	- 1. 先从编译阶段开始看起，在 parse 阶段，会执行 processAttrs 方法。
	- 2. processAttrs 方法在对标签属性的处理过程中，判断如果是指令，首先通过 parseModifiers 解析出修饰符，然后判断如果事件的指令，则执行 addHandler(el, name, value, modifiers, false, warn) 方法。
	- 3. addHandler 函数实际上就做了 3 件事情，首先根据 modifier 修饰符对事件名 name 做处理，接着根据 modifier.native 判断是一个纯原生事件还是普通事件，分别对应 el.nativeEvents 和 el.events，最后按照 name 对事件做归类，并把回调函数的字符串保留到对应的事件中。
	- 4. 然后在 codegen 的阶段，会在 genData 函数中根据 AST 元素节点上的 events 和 nativeEvents 生成 data 数据，也即是事件的代码字符串。

- 2. DOM 事件

	- 1. 原生 DOM 事件

- 3. 自定义事件

	- Vue 还支持了自定义事件，并且自定义事件只能作用在组件上，如果在组件上使用原生事件，需要加 .native 修饰符，普通元素上使用 .native 修饰符无效
	- 总结

		- Vue 支持 2 种事件类型，原生 DOM 事件和自定义事件，它们主要的区别在于添加和删除事件的方式不一样，并且自定义事件的派发是往当前实例上派发，但是可以利用在父组件环境定义回调函数来实现父子组件的通讯。另外要注意一点，只有组件节点才可以添加自定义事件，并且添加原生 DOM 事件需要使用 native 修饰符；而普通元素使用 .native 修饰符是没有作用的，也只能添加原生 DOM 事件。

**2. v-model**

实现

 在理解 Vue 的时候都把 Vue 的数据响应原理理解为双向绑定，但实际上这是不准确的，我们之前提到的数据响应，都是通过数据的改变去驱动 DOM 视图的变化，而双向绑定除了数据驱动 DOM 外， DOM 的变化反过来影响数据，是一个双向关系，在 Vue 中，我们可以通过 v-model 来实现双向绑定。

- 1. 表单元素

结合示例来分析:

```
let vm = new Vue({
  el: '#app',
  template: '<div>'
  + '<input v-model="message" placeholder="edit me">' +
  '<p>Message is: {{ message }}</p>' +
  '</div>',
  data() {
    return {
      message: ''
    }
  }
})
```

1. 首先在 parse 阶段， v-model 被当做普通的指令解析到 el.directives 中，然后在 codegen 阶段，执行 genData ，最终的生成的 code 
```
if($event.target.composing)return; message=$event.target.value
```
2. code 生成完后，又执行了 2 句非常关键的代码：

```
addProp(el, 'value', `(${value})`)
addHandler(el, event, code, null, true)  
```

这实际上就是 input 实现 v-model 的精髓，通过修改 AST 元素，给 el 添加一个 prop，相当于我们在 input 上动态绑定了 value，又给 el 添加了事件处理，相当于在 input 上绑定了 input 事件，其实转换成模板如下：

```
<input
  v-bind:value="message"
  v-on:input="message=$event.target.value">
```

其实就是动态绑定了 input 的 value 指向了 messgae 变量，并且在触发 input 事件的时候去动态把 message 设置为目标值，这样实际上就完成了数据双向绑定了，所以说 v-model 实际上就是语法糖。

3. 最终生成的 render 代码如下：

```
with(this) {
  return _c('div',[_c('input',{
    directives:[{
      name:"model",
      rawName:"v-model",
      value:(message),
      expression:"message"
    }],
    attrs:{"placeholder":"edit me"},
    domProps:{"value":(message)},
    on:{"input":function($event){
      if($event.target.composing)
        return;
      message=$event.target.value
    }}}),_c('p',[_v("Message is: "+_s(message))])
    ])
}
```

- 2. 组件

通过一个例子分析：

```
let Child = {
  template: '<div>'
  + '<input :value="value" @input="updateValue" placeholder="edit me">' +
  '</div>',
  props: ['value'],
  methods: {
    updateValue(e) {
      this.$emit('input', e.target.value)
    }
  }
}

let vm = new Vue({
  el: '#app',
  template: '<div>' +
  '<child v-model="message"></child>' +
  '<p>Message is: {{ message }}</p>' +
  '</div>',
  data() {
    return {
      message: ''
    }
  },
  components: {
    Child
  }
})
```

可以看到，父组件引用 child 子组件的地方使用了 v-model 关联了数据 message；而子组件定义了一个 value 的 prop，并且在 input 事件的回调函数中，通过 this.$emit('input', e.target.value) 派发了一个事件，为了让 v-model 生效，这两点是必须的。

其实就相当于我们在这样编写父组件：

```
let vm = new Vue({
  el: '#app',
  template: '<div>' +
  '<child :value="message" @input="message=arguments[0]"></child>' +
  '<p>Message is: {{ message }}</p>' +
  '</div>',
  data() {
    return {
      message: ''
    }
  },
  components: {
    Child
  }
})
```

子组件传递的 value 绑定到当前父组件的 message，同时监听自定义 input 事件，当子组件派发 input 事件的时候，父组件会在事件回调函数中修改 message 的值，同时 value 也会发生变化，子组件的 input 值被更新。

- 总结

	- v-model 实现双向绑定的本质上就是一种语法糖，它即可以支持原生表单元素，也可以支持自定义组件。在组件的实现中，我们是可以配置子组件接收的 prop 名称，以及派发的事件名称。

**3. slot**

- 1. 编译

	- 编译是发生在调用 vm.$mount 的时候，所以编译的顺序是先编译父组件，再编译子组件。

- 2. 普通插槽

	- 1. 有定义对应 name 的是具名插槽
	- 2. 没有定义 name 的是默认插槽

- 3. 作用域插槽
- 总结

	- 它们有一个很大的差别是数据作用域，普通插槽是在父组件编译和渲染阶段生成 vnodes，所以数据的作用域是父组件实例，子组件渲染的时候直接拿到这些渲染好的 vnodes。而对于作用域插槽，父组件在编译和渲染阶段并不会直接生成 vnodes，而是在父节点 vnode 的 data 中保留一个 scopedSlots 对象，存储着不同名称的插槽以及它们对应的渲染函数，只有在编译和渲染子组件阶段才会执行这个渲染函数生成 vnodes，由于是在子组件环境执行的，所以对应的数据作用域是子组件实例。

**4. keep-alive**

- 1. 内置组件

	- <keep-alive> 是 Vue 源码中实现的一个组件，是 Vue 的内置组件，是下抽象组件，形式 “有点像” 平时写的 Vue 的组件，但是做了缓存的处理。

- 2. 组件渲染
- 3. 生命周期
- 4. 总结

	- 1.  <keep-alive> 组件是一个抽象组件，它的实现通过自定义 render 函数并且利用了插槽，并且 <keep-alive> 缓存 vnode，组件包裹的子元素——也就是插槽是如何做更新的
	- 2. 且在 patch 过程中对于已缓存的组件不会执行 mounted，所以不会有一般的组件的生命周期函数但是又提供了 activated 和 deactivated 钩子函数。
	- 3. <keep-alive> 的 props 除了 include 和 exclude 还有文档中没有提到的 max，它能控制我们缓存的个数。

**5. transition**

- 1. 内置组件

	- <transition> 组件和 <keep-alive> 组件一样，都是 Vue 的内置组件，同样是抽象组件，同样直接实现 render 函数，同样利用了默认插槽。而且 <transition> 组件是 web 平台独有的

- 2. transition module

	- 动画相关的逻辑，过渡动画提供了 2 个时机，一个是 create 和 activate 的时候提供了 entering 进入动画，一个是 remove 的时候提供了 leaving 离开动画

- 3. entering

	- 主要发生在组件插入后

- 4. leaving

	- 主要发生在组件销毁前

- 5. 总结

	- 1. 自动嗅探目标元素是否应用了 CSS 过渡或动画，如果是，在恰当的时机添加/删除 CSS 类名。
	- 2. 如果过渡组件提供了 JavaScript 钩子函数，这些钩子函数将在恰当的时机被调用。
	- 3. 如果没有找到 JavaScript 钩子并且也没有检测到 CSS 过渡/动画，DOM 操作 (插入/删除) 在下一帧中立即执行。
	- 总结

		- 所以真正执行动画的是我们写的 CSS 或者是 JavaScript 钩子函数，而 Vue 的 <transition> 只是帮我们很好地管理了这些 CSS 的添加/删除，以及钩子函数的执行时机。

**6. transition-group**

- 1. render 函数

	- <transition-group> 组件也是由 render 函数渲染生成 vnode，不同于 <transition> 组件，<transition-group> 组件非抽象组件，它会渲染成一个真实元素，默认 tag 是 span。

- 2. move 过渡实现

- 3. 总结

	- <transtion-group> 和 <transition> 组件相比，实现了列表的过渡，以及它会渲染成真实的元素。当我们去修改列表的数据的时候，如果是添加或者删除数据，则会触发相应元素本身的过渡动画，这点和 <transition> 组件实现效果一样，除此之外 <transtion-group> 还实现了 move 的过渡效果，让我们的列表过渡动画更加丰富。

## 9. Vue-Router

![](https://upload-images.jianshu.io/upload_images/12890819-facaee2ee08767ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**1. introduction**

![](https://upload-images.jianshu.io/upload_images/12890819-83538269d41514ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- Vue-Router 的能力十分强大，它支持 hash、history、abstract 3 种路由方式，提供了 <router-link> 和 <router-view> 2 种组件，还提供了简单的路由配置和一系列好用的 API。注意：本思维导图主要讲的是 hash 模式下的。

**2. 路由注册**

![](https://upload-images.jianshu.io/upload_images/12890819-c20923bc2697cd76.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- 1. Vue.use

	- Vue 提供了 Vue.use 的全局 API 来注册这些插件，比如注册 VueRouter。

- 2. 路由安装

	- 1. VueRouter 本质上是一个类，实现了 install 的静态方法：VueRouter.install = install，当执行 Vue.use(VueRouter) 的时候，实际上就是在执行 install 函数
	- 2. Vue-Router 安装最重要的一步就是利用 Vue.mixin 去把 beforeCreate 和 destroyed 钩子函数注入到每一个组件中。
	- 3. 通过 Vue.component 方法定义了全局的 <router-link> 和 <router-view> 2 个组件，这也是为什么我们在写模板的时候可以使用这两个标签

**3. VueRouter 对象**

![](https://upload-images.jianshu.io/upload_images/12890819-28eff7d99d9725f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 1. VueRouter 的实现是一个类，定义了一些属性和方法。
- 2. 当我们执行 new VueRouter 的时候

	- 1. 在浏览器不支持 history.pushState 的情况下，根据传入的 fallback 配置参数，决定是否回退到 hash 模式。
	- 2. 实例化 VueRouter 后会返回它的实例 router

- 3. 组件在执行 beforeCreate 钩子函数的时候，如果传入了 router 实例，都会执行 router.init 进行初始化。
- 4. 然后又会执行 history.transitionTo 方法做路由过渡，进而引出了 matcher 的概念。

**4. matcher**

![](https://upload-images.jianshu.io/upload_images/12890819-f96c095cf5ea7570.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 1. createMatcher

	- createMatcher 的初始化就是根据路由的配置描述建立映射表，包括路径、名称到路由 record 的映射关系。

- 2. addRoutes

	- addRoutes 方法的作用是动态添加路由配置，因为在实际开发中有些场景是不能提前把路由写死的，需要根据一些条件动态添加路由

		- function addRoutes (routes) {
  createRouteMap(routes, pathList, pathMap, nameMap)
}

			- addRoutes 的方法十分简单，再次调用 createRouteMap 即可，传入新的 routes 配置，由于 pathList、pathMap、nameMap 都是引用类型，执行 addRoutes 后会修改它们的值。

- 3. match

	- match 会根据传入的位置和路径计算出新的位置并匹配到相应的路由 record ，然后根据新的位置 和 record 创建新的路径并返回。

		- 通过 matcher 的 match 方法，我们会找到匹配的路径 Route，这个对 Route 的切换，组件的渲染都有非常重要的指导意义。

**5. 路径切换**

![](https://upload-images.jianshu.io/upload_images/12890819-9aaef94d98dab920.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 1.  history.transitionTo

	- 1. 前一节我们分析了 matcher 的相关实现，知道它是如何找到匹配的新线路，那么匹配到新线路后，当我们切换路由线路的时候，就会执行到方法 transitionTo。
	- 2. 拿到新的路径后，那么接下来就会执行 confirmTransition 方法去做真正的切换，由于这个过程可能有一些异步的操作（如异步组件），所以整个 confirmTransition API 设计成带有成功回调函数和失败回调函数。
	- 3. 拿到 updated、activated、deactivated 3 个 ReouteRecord 数组后，接下来就是路径变换后的一个重要部分，执行一系列的钩子函数，也就是导航守卫。

- 2. 导航守卫

	- 实际上就是发生在路由路径切换的时候，执行的一系列钩子函数。

		- 完整的导航解析流程

			- 1. 导航被触发。
			- 2. 在失活的组件里调用离开守卫。
			- 3. 调用全局的 beforeEach 守卫。
			- 4. 在重用的组件里调用 beforeRouteUpdate 守卫 (2.2+)。
			- 5. 在路由配置里调用 beforeEnter。
			- 6. 解析异步路由组件。
			- 7. 在被激活的组件里调用 beforeRouteEnter。
			- 8. 调用全局的 beforeResolve 守卫 (2.5+)。
			- 9. 导航被确认。
			- 10. 调用全局的 afterEach 钩子。
			- 11. 触发 DOM 更新。
			- 12. 用创建好的实例调用 beforeRouteEnter 守卫中传给 next 的回调函数。

- 3. url ( hash 模式 )

	- 1. 当我们点击 router-link 的时候，实际上最终会执行 router.push。
	- 2. push 函数会先执行 this.transitionTo 做路径切换，在切换完成的回调函数中，执行 pushHash 函数
	- 3. pushState 会调用浏览器原生的 history 的 pushState 接口或者 replaceState 接口，更新浏览器的 url 地址，并把当前 url 压入历史栈中。
	- 4. ensureSlash

		- 开发项目时，打开调试页面 http://localhost:8080 后会自动把 url 修改为 http://localhost:8080/#/。这是因为在实例化 HashHistory 的时候，构造函数会执行 ensureSlash() 方法，修改了 url 的原因。

- 4. 组件

路由最终的渲染离不开组件，Vue-Router 内置了 <router-view> 组件。<router-view> 是一个 functional 组件，它的渲染也是依赖 render 函数。

1. <router-view> 是支持嵌套的，回到 render 函数，其中定义了 depth 的概念，它表示 <router-view> 嵌套的深度。
2. 每个 <router-view> 在渲染的时候，会进行一个循环，就是从当前的 <router-view> 的父节点向上找，一直找到根 Vue 实例，在这个过程，如果碰到了父节点也是 <router-view> 的时候，说明 <router-view> 有嵌套的情况，depth++。遍历完成后，根据当前线路匹配的路径和 depth 找到对应的 RouteRecord，进而找到该渲染的组件。
3. 注册路由实例

```
const registerInstance = (vm, callVal) => {
  let i = vm.$options._parentVnode
  if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
    i(vm, callVal)
  }
}

Vue.mixin({
  beforeCreate () {
    // ...
    registerInstance(this, this)
  },
  destroyed () {
    registerInstance(this)
  }
})
```

在混入的 beforeCreate 钩子函数中，会执行 registerInstance 方法，进而执行 render 函数中定义的 registerRouteInstance 方法，从而给 matched.instances[name] 赋值当前组件的 vm 实例。

4. render 函数的最后根据 component 渲染出对应的组件 vonde：

return h(component, data, children)

5.  当我们执行 transitionTo 来更改路由线路后，组件是如何重新渲染 ？

  1. 在 Vue  混入的 beforeCreate 钩子函数中，我们把根 Vue 实例的 _route 属性定义成响应式的了。
                    
```
 if (isDef(this.$options.router)) {
      Vue.util.defineReactive(this, '_route', this._router.history.current)
    }
```
  2. 访问 this._routerRoot._route，触发了它的 getter，相当于 <router-view> 对它有依赖，然后再执行完 transitionTo 后，修改 app._route 的时候，又触发了setter，因此会通知 <router-view> 的渲染 watcher 更新，重新渲染组件。

 - 5. <router-link>

Vue-Router 还内置了另一个组件 <router-link>，它支持用户在具有路由功能的应用中（点击）导航。 通过 to 属性指定目标地址，默认渲染成带有正确链接的 <a> 标签，可以通过配置 tag 属性生成别的标签。另外，当目标路由成功激活时，链接元素自动设置一个表示激活的 CSS 类名。

1. 首先做了路由解析
2. router.resolve 计算出最终跳转的 href
3. 对 exactActiveClass 和 activeClass 做处理
4. 创建了一个守卫函数 handler，最终会监听点击事件或者其它可以通过 prop 传入的事件类型，执行 hanlder 函数，最终执行 router.push 或者 router.replace 函数

实际上就是执行了 history 的 push 和 replace 方法做路由跳转。

5. 最后判断当前 tag 是否是 <a> 标签，<router-link> 默认会渲染成 <a> 标签，当然我们也可以修改 tag 的 prop 渲染成其他节点，这种情况下会尝试找它子元素的 <a> 标签，如果有则把事件绑定到 <a> 标签上并添加 href 属性，否则绑定到外层元素本身。

- 6. 总结

	- 路径变化是路由中最重要的功能，我们要记住以下内容：路由始终会维护当前的线路，路由切换的时候会把当前线路切换到目标线路，切换过程中会执行一系列的导航守卫钩子函数，会更改 url，同样也会渲染对应的组件，切换完毕后会把目标线路更新替换当前线路，这样就会作为下一次的路径切换的依据。

**6. 总结**

- 路由始终会维护当前的线路，路由切换的时候会把当前线路切换到目标线路，切换过程中会执行一系列的导航守卫钩子函数，会更改 url，同样也会渲染对应的组件，切换完毕后会把目标线路更新替换当前线路，这样就会作为下一次的路径切换的依据。

## 10. Vuex

![](https://upload-images.jianshu.io/upload_images/12890819-5710f3019ec2b189.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**1. introduction**

![](https://upload-images.jianshu.io/upload_images/12890819-840551bd55102022.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- Vuex 是一个专为 Vue.js 应用程序开发的状态管理模式。它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。
- 什么是“状态管理模式”？

	- 1. state，驱动应用的数据源；

	- 2. view，以声明方式将 state 映射到视图；
	- 3. actions，响应在 view 上的用户输入导致的状态变化。

- Vuex 核心思想

	- 1. Vuex 应用的核心就是 store（仓库）。“store”基本上就是一个容器，它包含着你的应用中大部分的状态 (state)
	- 2. Vuex 的状态存储是响应式的。当 Vue 组件从 store 中读取状态的时候，若 store 中的状态发生变化，那么相应的组件也会相应地得到高效更新。
	- 3. 你不能直接改变 store 中的状态。改变 store 中的状态的唯一途径就是显式地提交 (commit) mutation。这样使得我们可以方便地跟踪每一个状态的变化，从而让我们能够实现一些工具帮助我们更好地了解我们的应用。

**2. Vuex 初始化**

![](https://upload-images.jianshu.io/upload_images/12890819-bd0c497f48f4430c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- 1. 安装

1. 当我们在代码中通过 import Vuex from 'vuex' 的时候，实际上引用的是一个对象，它的定义在 src/index.js 中：

```
export default {
  Store,
  install,
  version: '__VERSION__',
  mapState,
  mapMutations,
  mapGetters,
  mapActions,
  createNamespacedHelpers
}
```

2. 和 Vue-Router 一样，Vuex 也同样存在一个静态的 install 方法，它的定义在 src/store.js 中：

```
export function install (_Vue) {
  if (Vue && _Vue === Vue) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}
```
3. install 的逻辑很简单，把传入的 _Vue 赋值给 Vue 并执行了 applyMixin(Vue) 方法，执行 Vue.mixin({ beforeCreate: vuexInit })。

它其实给 Vue 全局混入了一个 beforeCreate 钩子函数，它的实现非常简单，就是把 options.store 保存在所有组件的 this.$store 中，这个 options.store 就是我们在实例化 Store 对象的实例。

- 2. Store 实例化

1. 用法

```
const store = new Vuex.Store({
    strict: process.env.NODE_ENV !== "production",
    modules: {
        moduleA
    },
    state: initPageState(),
    mutations: {},
    actions: {}
});

export default store;

```

Store 对象的构造函数也是一个 class，接收一个对象参数，它包含 actions、getters、state、mutations、modules 等 Vuex 的核心概念

2. 初始化模块

1. Vuex 允许我们将 store 分割成模块（module）。每个模块拥有自己的 state、mutation、action、getter，甚至是嵌套子模块——从上至下进行同样方式的分割

```
- const moduleA = {
  state: { ... },
  mutations: { ... },
  actions: { ... },
  getters: { ... }
}

const moduleB = {
  state: { ... },
  mutations: { ... },
  actions: { ... },
  getters: { ... },
}

const store = new Vuex.Store({
  modules: {
    a: moduleA,
    b: moduleB
  }
})

store.state.a // -> moduleA 的状态
store.state.b // -> moduleB 的状态
```

从数据结构上来看，模块的设计就是一个树型结构，store 本身可以理解为一个 root module，它下面的 modules 就是子模块，Vuex 需要完成这颗树的构建。

- 2. 构建过程的入口

```
this._modules = new ModuleCollection(options)
```

1. 调用  register 方法，通过 const newModule = new Module(rawModule, runtime) 创建了一个 Module 的实例，Module 是用来描述单个模块的类。

2. register 首先根据路径获取到父模块，然后再调用父模块的 addChild 方法建立父子关系。

3. register 的最后一步，就是遍历当前模块定义中的所有 modules，根据 key 作为 path，递归调用 register 方法，这样就建立父子关系。

- 3. 安装模块

对模块中的 state、getters、mutations、actions 做初始化工作
它的入口代码是：

```
const state = this._modules.root.state;
installModule(this, state, [], this._modules.root);
```

1. 默认情况下，模块内部的 action、mutation 和 getter 是注册在全局命名空间的——这样使得多个模块能够对同一 mutation 或 action 作出响应。如果我们希望模块具有更高的封装度和复用性，可以通过添加 namespaced: true 的方式使其成为带命名空间的模块。当模块被注册后，它的所有 getter、action 及 mutation 都会自动根据模块注册的路径调整命名。

2. 构造了一个本地上下文环境：

```
const local = module.context = makeLocalContext(store, namespace, path);
```

3. registerMutation
4. registerAction
5. registerGetter

总结: 所以 installModule 实际上就是完成了模块下的 state、getters、actions、mutations 的初始化工作，并且通过递归遍历的方式，就完成了所有子模块的安装工作。

- 4. 初始化 store._vm

Store 实例化的最后一步，就是执行初始化 store._vm 的逻辑，它的入口代码是：

```
resetStoreVM(this, state);
```

resetStoreVM 的作用实际上是想建立 getters 和 state 的联系，因为从设计上 getters 的获取就依赖了 state ，并且希望它的依赖能被缓存起来，且只有当它的依赖值发生了改变才会被重新计算。因此这里利用了 Vue 中用 computed 计算属性来实现。

strict mode

当严格模式下，store._vm 会添加一个 wathcer 来观测 this._data.$$state 的变化，也就是当 store.state 被修改的时候, store._committing 必须为 true，否则在开发阶段会报警告。

```
if (store.strict) {
  enableStrictMode(store)
}

function enableStrictMode (store) {
  store._vm.$watch(function () { return this._data.$$state }, () => {
    if (process.env.NODE_ENV !== 'production') {
      assert(store._committing, `Do not mutate vuex store state outside mutation handlers.`)
    }
  }, { deep: true, sync: true })
}
```

- 3. 总结

我们要把 store 想象成一个数据仓库，为了更方便的管理仓库，我们把一个大的 store 拆成一些 modules，整个 modules 是一个树型结构。每个 module 又分别定义了 state，getters，mutations、actions，我们也通过递归遍历模块的方式都完成了它们的初始化。为了 module 具有更高的封装度和复用性，还定义了 namespace 的概念。最后我们还定义了一个内部的 Vue 实例，用来建立 state 到 getters 的联系，并且可以在严格模式下监测 state 的变化是不是来自外部，确保改变 state 的唯一途径就是显式地提交 mutation。

**3. API**

![](https://upload-images.jianshu.io/upload_images/12890819-af5afdab0bd77514.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 1. 数据获取

	- Vuex 最终存储的数据是在 state 上的，我们之前分析过在 store.state 存储的是 root state，那么对于模块上的 state，假设我们有 2 个嵌套的 modules，它们的 key 分别为 a 和 b，我们可以通过 store.state.a.b.xxx 的方式去获取。

		- 在递归执行 installModule 的过程中，就完成了整个 state 的建设，这样我们就可以通过 module 名的 path 去访问到一个深层 module 的 state。

- 2. 数据存储

	- 1. Vuex 对数据存储的存储本质上就是对 state 做修改，并且只允许我们通过提交 mutaion 的形式去修改 state。
	- 2. mutation 必须是同步函数
	- 3. action

		- action 类似于 mutation，不同在于 action 提交的是 mutation，而不是直接操作 state，并且它可以包含任意异步操作。

			- action 比我们自己写一个函数执行异步操作然后提交 muataion 的好处是在于它可以在参数中获取到当前模块的一些方法和状态，Vuex 帮我们做好了这些。

- 3. 语法糖

1. mapState

mapState 支持传入 namespace， 因此我们可以这么写：

```
computed: {
  mapState('some/nested/module', {
    a: state => state.a,
    b: state => state.b
  })
},
```

在 mapState 的实现中，如果有 namespace，则尝试去通过 getModuleByNamespace(this.$store, 'mapState', namespace) 对应的 module，然后把 state 和 getters 修改为 module 对应的 state 和 getters

主要原因是在 Vuex 初始化执行 installModule 的过程中，初始化了这个映射表：

```
function installModule (store, rootState, path, module, hot) {
  // ...
  const namespace = store._modules.getNamespace(path)

  // register in namespace map
  if (module.namespaced) {
    store._modulesNamespaceMap[namespace] = module
  }

  // ...
}
```

2. mapGetters

mapGetters 的用法：

```
import { mapGetters } from 'vuex'

export default {
  // ...
  computed: {
    // 使用对象展开运算符将 getter 混入 computed 对象中
    mapGetters([
      'doneTodosCount',
      'anotherGetter',
      // ...
    ])
  }
}
```

和 mapState 类似，mapGetters 是将 store 中的 getter 映射到局部计算属性

mapGetters 也同样支持 namespace，如果不写 namespace ，访问一个子 module 的属性需要写很长的 key，一旦我们使用了 namespace，就可以方便我们的书写，每个 mappedGetter 的实现实际上就是取 this.$store.getters[val]。

- 3. mapMutations

我们可以在组件中使用 this.$store.commit('xxx') 提交 mutation，或者使用 mapMutations 辅助函数将组件中的 methods 映射为 store.commit 的调用。mapMutations 支持传入一个数组或者一个对象，目标都是组件中对应的 methods 映射为 store.commit 的调用。

mapMutations 的用法：

```
import { mapMutations } from 'vuex'

export default {
  // ...
  methods: {
    ...mapMutations([
      'increment', // 将 `this.increment()` 映射为 `this.$store.commit('increment')`

      // `mapMutations` 也支持载荷：
      'incrementBy' // 将 `this.incrementBy(amount)` 映射为 `this.$store.commit('incrementBy', amount)`
    ]),
    ...mapMutations({
      add: 'increment' // 将 `this.add()` 映射为 `this.$store.commit('increment')`
    })
  }
}
```

mappedMutation 同样支持了 namespace，并且支持了传入额外的参数 args，作为提交 mutation 的 payload，最终就是执行了 store.commit 方法，并且这个 commit 会根据传入的 namespace 映射到对应 module 的 commit 上。

- 4. mapActions

在组件中使用 this.$store.dispatch('xxx') 提交 action，或者使用 mapActions 辅助函数将组件中的 methods 映射为 store.dispatch 的调用。

mapActions 在用法上和 mapMutations 几乎一样，实现也很类似，和 mapMutations 的实现几乎一样，不同的是把 commit 方法换成了 dispatch。

- 4. 动态更新模块

1. 模块动态注册 registerModule

在有一些场景下，我们需要动态去注入一些新的模块，Vuex 提供了模块动态注册功能，在 store 上提供了一个 registerModule 的 API。

registerModule 支持传入一个 path 模块路径 和 rawModule 模块定义，首先执行 register 方法扩展我们的模块树，接着执行 installModule 去安装模块，最后执行 resetStoreVM 重新实例化 store._vm，并销毁旧的 store._vm。

2. 动态卸载模块 unregisterModule

相对的，有动态注册模块的需求就有动态卸载模块的需求，Vuex 提供了模块动态卸载功能，在 store 上提供了一个 unregisterModule 的 API。

1. unregisterModule 支持传入一个 path 模块路径，首先执行 unregister 方法去修剪我们的模块树。 注意，这里只会移除我们运行时动态创建的模块。
2. 接着会删除 state 在该路径下的引用，最后执行 resetStore 方法。
3. 该方法就是把 store 下的对应存储的 _actions、_mutations、_wrappedGetters 和 _modulesNamespaceMap 都清空，然后重新执行 installModule 安装所有模块以及 resetStoreVM 重置 store._vm。

**4. 插件**

![](https://upload-images.jianshu.io/upload_images/12890819-ebb4c7e7a9903949.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Vuex 除了提供的存取能力，还提供了一种插件能力，让我们可以监控 store 的变化过程来做一些事情。

- 1. Vuex 的 store 接受 plugins 选项，我们在实例化 Store 的时候可以传入插件，它是一个数组，然后在执行 Store 构造函数的时候，会执行这些插件：

```
const {
  plugins = [],
  strict = false
} = options
// apply plugins
plugins.forEach(plugin => plugin(this))；
```      
 
- 2. Logger 插件

1. Logger 函数，它相当于订阅了 mutation 的提交，它的 prevState 表示之前的 state，nextState 表示提交 mutation 后的 state，这两个 state 都需要执行 deepCopy 方法拷贝一份对象的副本，这样对他们的修改就不会影响原始 store.state。
2. 接下来就构造一些格式化的消息，打印出一些时间消息 message， 之前的状态 prevState，对应的 mutation 操作 formattedMutation 以及下一个状态 nextState。


3. 最后更新 prevState = nextState，为下一次提交 mutation 输出日志做准备。
4. 总结

- Vuex 从设计上支持了插件，让我们很好地从外部追踪 store 内部的变化，Logger 插件在我们的开发阶段也提供了很好地指引作用。

## 11. 已完成与待完成

**已完成**：

- 思维导图

**待完成**：

- 继续完善 思维导图
- 添加 流程图

因为该项目都是业余时间做的，笔者能力与时间也有限，很多细节还没有完善。

如果你是大神，或者对 vue 源码有更好的见解，**欢迎提交 issue ，大家一起交流学习，一起打造一个像样的 讲解 Vue 全家桶源码架构 的开源项目**。

## 12. 总结

以上内容是笔者最近学习 Vue 源码时的收获与所做的笔记，本文内容大多是开源项目 **Vue.js 技术揭秘** 的内容，只不过是以思维导图的形式来展现，内容有省略，还加入了笔者的一点理解。

笔者之所以采用思维导图的形式来记录所学内容，是因为思维导图更能反映知识体系与结构，更能使人形成完整的知识架构，知识一旦形成一个体系，就会容易理解和不易忘记。

> **注意**：文章的图片可能上传时会经过压缩，可能有点模糊，不过本文用到的 所有 **超清图片** 都已经放在 [github](https://github.com/biaochenxuying/vue-family-mindmap) 上，而且还有 **pdf 格式、markdown 语法、思维导图 的原文件**，自己可以根据 **思维导图原文件** 导出相应的超清图片。

笔者文章常更地址：

[1. 微信公众号](https://mp.weixin.qq.com/s/-gwh7z1xjpBQ4IzD1ZJd-g)
[2. github](https://github.com/biaochenxuying/blog)
[3. 全栈修炼](https://biaochenxuying.cn/)

## 13. 最后

> 传承至善

如果你觉得本文章或者项目对你有启发，请给个赞或者  star 吧，点赞是一种美德，谢谢。

参考开源项目：

1. [https://github.com/ustbhuangyi/vue-analysis](https://github.com/ustbhuangyi/vue-analysis)
2. [https://github.com/HcySunYang/vue-design](https://github.com/HcySunYang/vue-design)


关注公众号并回复 **福利** 可领取免费学习资料，福利详情请猛戳：  [Python、Java、Linux、Go、node、vue、react、javaScript](https://biaochenxuying.cn/articleDetail?article_id=5bf4ba3c245730373274df61)

![全栈修炼](https://upload-images.jianshu.io/upload_images/12890819-bce9560fec5c49ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
