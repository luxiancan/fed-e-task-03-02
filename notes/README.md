## Vue.js 源码剖析-响应式原理

### 准备工作

#### Vue 源码获取
- 项目地址： https://github.com/vuejs/vue
- Fork 一份到自己的仓库，克隆到本地，可以自己写注释提交到 github
- 为什么分析 Vue 2.6
  + 到目前为止 Vue 3.0 的正式版还没有发布
  + 新版本发布后，现有项目不会升级到 3.0 ，2.x 还有很长的一段过渡期
  + 3.0 项目地址： https://github.com/vuejs/vue-next

#### 了解 Flow
- 官网： https://flow.org/
- JavaScript 的静态类型检查器
- Flow 的静态类型检查错误是通过静态类型推断实现的
- 文件开头通过 `// @flow` 或者 `/* @flow */` 声明


#### 调式设置

打包

打包工具 Rollup
- Vue.js 源码的打包工具使用的是 Rollup , 比 webpack 轻量
- webpack 把所有文件当做模块，Rollup 只处理 js 文件更适合在 Vue.js 这样的库中使用
- Rollup 打包不会生成冗余的代码

设置 sourcemap
- package.json 文件中的 dev 脚本中添加参数 --sourcemap
- `"dev": "rollup -w -c scripts/config.js --sourcemap --environment TARGET:web-full-dev",`

执行 dev
- npm run dev 执行打包，用的是 rollup , -w 参数是监听文件的变化，文件变化自动重新打包

#### Vue的不同构建版本

- npm run build 重新打包所有文件

术语
- 完整版：同时包含编译器和运行时的版本
- 编译器：用来将模板字符串编译称为 JS 渲染函数的代码，体积大、效率低
- 运行时：用来创建 Vue 实例、渲染并处理虚拟 DOM 等的代码，体积小、效率高、基本上就是除去编译器的代码
- UMD：UMD 版本是通用的模板版本，支持多种模块方式。vue.js 默认文件就是运行时 + 编译器的 UMD 版本
- CommonJS(cjs): CommonJS 版本用来配合老的打包工具比如 Browserify 或 webpack 1
- ES Module: 从 2.6 开始 Vue 会提供两个 ES Modules 构建文件，为现代打包工具提供的版本
  + ESM 格式被设计为可以被静态分析，所以打包工具可以利用这一点来进行 tree-shaking 并将用不到的代码排除出最终的包
  + ES6 模块与 CommonJS 模块的差异，参考  https://es6.ruanyifeng.com/?search=export&x=0&y=0#docs/module-loader#ES6-%E6%A8%A1%E5%9D%97%E4%B8%8E-CommonJS-%E6%A8%A1%E5%9D%97%E7%9A%84%E5%B7%AE%E5%BC%82


### 寻找入口文件
  - 查看 dist/vue.js 的构建过程
  - 执行 npm run dev
  - script/config.js 的执行过程
    + 作用：生成 rollup 构建的配置文件
    + 使用环境变量 TARGET = web-full-dev

#### 从入口开始
- src\platforms\web\entry-runtime-with-compiler.js
- 阅读源码记录
  + el 不能是 body 或者 html 标签
  + 如果没有 render ，把 template 转换成 render 函数
  + 如果有 render 方法，直接调用 mount 挂载 DOM


### Vue初始化的过程

#### 四个导出Vue的模块
- src\platforms\web\entry-runtime-with-compiler.js
  + web 平台相关的入口
  + 重写了平台相关的 $mount() 方法
  + 注册了 Vue.compile() 方法，传递一个 HTML 字符串返回 render 函数
- src\platforms\web\runtime\index.js
  + web平台相关
  + 注册和平台相关的全局指令： v-model v-show
  + 注册和平台相关的全局组件： v-transition  v-transition-group
  + 全局方法： `__patch__`: 把虚拟dom转换成真实DOM; `$mount`: 挂载方法
- src\core\index.js
  + 与平台无关
  + 设置了 vue的静态方法，initGlobalAPI(Vue)
- src\core\instance\index.js
  + 与平台无关
  + 定义了狗仔函数，调用了 this._init(options) 方法
  + 给 Vue 中混入了常用的实例成员


### 数据响应式原理

#### 通过查看源码解决下面的问题
- vm.msg = { count: 0 } ,重新给属性赋值，是否是相应式的？
- vm.arr[0] = 4 , 给数组元素赋值，视图是否会更新？
- vm.arr.length = 0 , 修改数组的 length ，视图是否会更新
- vm.arr.push(4) , 视图是否会更新

#### 相应式处理的入口
- src/core/instance/init.js
  + initState(vm) vm 状态的初始化
  + 初始化了 _data , _props , methods 等等
- src/core/instance/state.js

### Watcher 类
- Watcher 分为三种，Computed Watcher 、用户 Watcher （侦听器）、渲染 Watcher
- 渲染 Watcher 的创建时机： /src/core/instance/lifecycle.js

#### set 方法源码
- Vue.set()  global-api/index.js
- vm.$set()  instance/index.js

#### delete 方法源码
- Vue.delete() global-api/index.js
- vm.$delete() instance/index.js
- 删除对象的属性。如果对象是响应式的，确保删除能触发更新视图。这个方法主要用于避开 Vue 不能检测到属性被删除的限制，但是应该很少会使用它。
- 注意：目标对象不能是一个 Vue 实例或 Vue 实例的跟数据对象

#### vm.$watch
vm.$watch(expOrFn, callback, [options])
- 观察 Vue 实例变化的一个表达式或计算属性函数。回调函数得到的参数为新值和旧值。表达式只接受监督的键路径。对于更复杂的表达式，用一个函数取代。
- expOrFn: 要监视的 $data 中的属性，可以是表达式或函数
- callback: 数据变化后执行的函数
  + 函数：回调函数
  + 对象： 具有 handler 属性（字符串或者函数），如果该属性为字符串则 methods 中相应的定义
- options: 可选的选项
  + deep: 布尔类型，深度监听
  + immediate: 布尔类型，是否立即执行一次回调函数


#### 三种类型的 Watcher 对象
- 没有静态方法，因为 $watch 方法中要使用 Vue 的实例
- Watcher 分三种：计算属性 Watcher、用户 Watcher（侦听器）、渲染 Watcher
- 创建顺序：计算属性 Watcher、用户 Watcher（侦听器）、渲染 Watcher
- vm.$watch() : src/core/instance/state.js

#### 异步更新队列-nextTick()
- Vue 更新 DOM 是异步执行的，批量的
- 在下次 DOM 更新循环结束之后执行延迟回调。在修改数据之后立即使用这个方法，获取更新后的 DOM
- `vm.$nextTick(function () { /*操作DOM*/ })` 或 `Vue.nextTick(function () {})`
- 源码位置： src/core/instance/render.js

源码
- 手动调用 vm.$nextTick()
- 在 Watcher 的 queueWatcher 中执行 nextTick()
- src/core/util/next-tick.js
- 核心是 timerFunc 函数的处理，Promise -> MutationObserver -> setImmediate -> setTimeout


## Vue.js 源码剖析-虚拟 DOM

### 什么是虚拟DOM
- 虚拟 DOM (Virtual DOM) 是使用 JavaScript 对象描述真实 DOM
- Vue.js 中的虚拟 DOM 借鉴 Snabbdom ，并添加了 Vue.js 的特性。例如：指令和组件机制

### 为什么要使用虚拟 DOM
- 避免直接操作 DOM，提高开发效率
- 作为一个中间件可以跨平台
- 虚拟 DOM 不一定可以提高性能
  + 首次渲染的时候会增加开销
  + 复杂视图情况下提升渲染性能

### h 函数
vm.$createElement(tag, data, children, normalizeChildren)
- tag: 标签名或者组件对象
- data: 描述 tag ,可以设置 DOM 的属性或者标签的属性
- children: tag 中的文本内容或者子节点

### 设置 key 的情况
设置 Key 的好处：  
（1）数据更新时，可以尽可能的减少DOM操作；  
（2）列表渲染时，可以提高列表渲染的效率，提高页面的性能；  


## Vue.js 源码剖析-模板编译和组件化

### 模板编译的作用
- Vue 2.x 使用 VNode 描述视图以及各种交互，用户自己编写 VNode 比较复杂
- 用户只需要编写类似 HTML 的代码 - Vue.js 模板，通过编译器将模板转换为返回 VNode 的 render 函数
- .vue 文件会被 webpack 在构建的过程中转换成 render 函数

### 编译生成的函数的位置
- _c()  src\core\instance\render.js
- _m()/_v()/_s()  \src\core\instance\render-helpers\index.js

### 抽象语法树

什么是抽象语法树
- 抽象语法树简称 AST (Abstract Syntax ZTree)
- 使用对象的形式描述树形的代码结构
- 此处的抽象语法树是用来描述树形结构的 HTML 字符串

为什么要使用抽象语法树
- 模板字符串转换成 AST 后，可以通过 AST 对模板做优化处理
- 标记模板中的静态内容，在 patch 的时候直接跳过静态内容
- 在 patch 的过程中静态内容不需要对比和重新渲染

### 组件化

- 一个 Vue 组件就是一个拥有预定义选项的一个 Vue 实例
- 一个组件可以组成页面上一个功能完备的区域，组件可以包含脚本、样式、模板

#### 首次渲染过程
- Vue 构造函数
- this._init()
- this.$mount()
- mountComponent()
- new Watcher() 渲染 Watcher
- updateComponent()
- vm._render() -> createElement()
- vm._update()
