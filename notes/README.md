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


### 首次渲染过程

