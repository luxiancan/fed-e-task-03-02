# 卢显灿 ｜ Part 3 | 模块二

## 简答题

### 1、请简述 Vue 首次渲染的过程。

Vue 首次渲染的过程：

（1）**Vue 初始化**：初始化 vue 的实例成员、静态成员；

（2）**new Vue()**：初始化结束之后，调用 vue 构造函数。构造函数中调用了 this._init() ，这个方法相当于vue的入口，最终调用 vm.$mount() ；

（3）**调用入口文件的 vm.$mount()**：这个方法主要是将模板编译成 render 函数。先判断是否传递了 render 选项，如果没有传递 render ，就把模板编译成 render 函数。这个过程是通过 compileToFunctions() 生成 render() 渲染函数 （new Function）。最后将 options.render = render；

（4）**调用runtime版本中的vm.$mount()**：在这个方法中会重新获取 el

（5）**调用 lifecycle.js 中的 mountComponent(this, el)**：方法中先判断是否有render选项，如果没有但是传入了模板，并且当前是开发环境的话会发送警告；触发 beforeMount ；然后定义 updateComponent ，这个方法中会调用 vm._render() 渲染虚拟dom ，调用 vm._update() 将虚拟dom转换成真实dom；然后创建 Watcher 实例，创建过程中传入了 updateComponent 会在 Watcher 内部调用 ，再调用 get() 方法；然后触发 mounted ，最终返回 vm ；

（6）**watcher.get()**：创建完 Watcher 会调用一次 get ，get内部调用 updateComponent() ；调用 vm._render() 创建 VNode ：调用实例化时用户传入的 render 或者编译 template 生成的 render ，返回 VNode ；调用 vm._update() : 内部调用了 `vm.__patch__(vm.$el, vnode)` 去挂载真实dom 并记录 vm.$el 。


### 2、请简述 Vue 响应式原理。

Vue 响应式原理其实是在 vm._init() 中完成的，调用顺序 initState() --> initData() --> observe() 。observe() 就是响应式的入口函数。

（1）**observe(value)**：这个方法接收一个参数 value ，就是需要处理成响应式的对象；判断 value 是否为对象，如果不是直接返回；判断 value 对象是否有 `__ob__` 属性，如果有直接返回；如果没有，**创建 observer 对象**；返回 observer 对象；

（2）**Observer**：给 value 对象定义不可枚举的 `__ob__` 属性，记录当前的 observer 对象；数组的响应式处理，覆盖原生的 push/splice/unshift 等方法，它们会改变原数组，当这些方法被调用时会发送通知；对象的响应式处理，调用 walk 方法，遍历对象的每个属性，**调用 defineReactive** ；

（3）**defineReactive**：为每一个属性创建 dep 对象，如果当前属性的值是对象，再调用 observe ；**定义 getter** ，收集依赖，返回属性的值；**定义 setter** ，保存新值，如果新值是对象，调用 observe，派发更新(发送通知)，调用 dep.notify() ;

（4）**依赖收集**：在 Watcher 对象的 get 方法中调用 pushTarget 记录 Dep.target 属性；访问 data 中的成员时收集依赖， defineReactive 的 getter 中收集依赖；把属性对应的 watcher 对象添加到 dep 的 subs 数组中；给 childOb 收集依赖，目的是子对象添加和删除成员时发送通知；

（5）**Watcher**：dep.notify 在调用 watcher 对象的 update() 方法时，调用 queueWatcher() ，判断 watcher 是否被处理，如果没有的话添加到 queue 队列中，并调用 flushSchedulerQueue() : 触发 beforeUpdate 钩子，调用 watcher.run() , run() --> get() --> getter() --> updateComponent ，清空上一次的依赖，触发 actived 钩子，触发 updated 钩子。


### 3、请简述虚拟 DOM 中 Key 的作用和好处。

Key 的作用：
主要用来在虚拟 DOM 的 diff 算法中，在新旧节点的对比时辨别 vnode ，使用 key 时，Vue 会基于 key 的变化重新排列元素顺序，尽可能的复用页面元素，只找出必须更新的DOM，最终可以减少DOM操作。常见的列子是结合 v-for 来进行列表渲染，或者用于强制替换元素/组件。

设置 Key 的好处：
（1）数据更新时，可以尽可能的减少DOM操作；
（2）列表渲染时，可以提高列表渲染的效率，提高页面的性能；


### 4、请简述 Vue 中模板编译的过程。

模板编译的过程：

（1）**compileToFunctions(template, ...)**：模板编译的入口函数，先从缓存中加载编译好的 render 函数，如果缓冲中没有，则调用 compile(template, options) 开始编译；

（2）**compile(template, options)**：先合并选项 options ，再调用 baseCompile(template.trim(), finalOptions) ；compile 的核心是合并 options ，真正处理模板是在 baseCompile 中完成的；

（3）**baseCompile(template.trim(), finalOptions)**：先调用 parse() 把 template 转换成 AST tree ；然后调用 optimize() 优化 AST ，先标记 AST tree 中的静态子树，检测到静态子树，设置为静态，不需要在每次重新渲染的时候重新生成节点，patch 阶段跳过静态子树；调用 generate() 将 AST tree 生成js代码；

（4）**compileToFunctions(template, ...)**：继续把上一步中生成的字符串形式的js代码转换为函数，调用 createFunction() 通过 `new Function(code)` 将字符串转换成函数； render 和 staticRenderFns 初始化完毕，挂载到 Vue 实例的 options 对应的属性中。

