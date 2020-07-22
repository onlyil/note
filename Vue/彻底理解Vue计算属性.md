## 前言

提到计算属性，我们马上就会想到它的一个特性：缓存，Vue 文档也如是说：

> **计算属性是基于它们的响应式依赖进行缓存的**

那么计算属性如何缓存的呢，计算属性的观察者是如何进行依赖收集的呢，接下来深入原理看一下。

本文需要对基础的响应式原理和几个关键角色 `Observer、Dep、Watcher` 等有一定的了解，可参考 [Observer、Dep、Watcher 傻傻搞不清楚](https://juejin.im/post/5e83ed516fb9a03c550fcdb7)

以一个简单的例子开始：

```vue
<div id="app">
    <h2>{{ this.text }}</h2>
    <button @click="changeName">Change name</button>
</div>

const vm = new Vue({
    el: '#app',
    data() {
        return {
            name: 'xiaoming',
        }
    },
    computed: {
        text() {
            return `Hello, ${this.name}!`
        }
    },
    methods: {
        changeName() {
            this.name = 'onlyil'
        },
    },
})
```

初始展示 `Hello, xiaoming!`，点击按钮后展示 `Hello, onlyil!`

## Vue 初始化

还是从 vue 初始化看起，从 `new Vue()` 开始，构造函数会执行 `this._init`，在 `_init` 中会进行合并配置、初始化生命周期、事件、渲染等，最后执行 `vm.$mount` 进行挂载。

```javascript
// src/core/instance/index.js
function Vue (options) {
    // ...
    this._init(options)
}

// src/core/instance/init.js
Vue.prototype._init = function (options?: Object) {
    // 合并选项
    // ...
    // 一系列初始化
    // ...
    initState(vm)
    // ...
    
    // 挂载
    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
}
```

计算属性的初始化就在 `initState` 中：

```javascript
// src/core/instance/state.js
export function initState (vm: Component) {
    const opts = vm.$options
    // ...
    // 初始化 computed
    if (opts.computed) initComputed(vm, opts.computed)
    // ...
}
```

## computed 初始化

看一下 `initComputed` ：

```javascript
function initComputed(vm, computed) {
    const watchers = vm._computedWatchers = Object.create(null)
    // 遍历 computed 选项，依次进行定义
    for (const key in computed) {
        const getter = computed[key]

        // 为计算属性创建内部 watcher
        watchers[key] = new Watcher(
            vm,
            getter || noop, // 计算属性 text 函数
            noop,
            computedWatcherOptions // { lazy: true } ，指定 lazy 属性，表示要实例化 computedWatcher
        )

        // 为计算属性定义 getter
        defineComputed(vm, key, userDef)
    }
}
```

### 1. 定义 _computedWatchers

首先定义一个 `watchers` 空对象，同时挂在 `vm._computedWatchers` 上，用来存放该 vm 实例的所有 computedWatcher。

### 2. 实例化 computedWatcher

遍历 computed 选项并实例化 watcher，参数中的 `getter` 就是上边示例中计算属性 `text` 对应的函数：
```javascript
function () {
    return `Hello, ${this.name}!`
}
```

参数中的 `computedWatcherOptions` 为 `{ lazy: true }` ，指定 lazy 属性，表示要实例化的是 computedWatcher。

实例化 computedWatcher ：

```javascript
class Watcher {
    constructor(vm, expOrFn, cb, options) {
        // options 为 { lazy: true }
        if (options) {
            // ...
            this.lazy = !!options.lazy
            // ...
        }
        this.dirty = this.lazy // for lazy watchers, 初始 dirty 为 true

        this.getter = expOrFn

        // lazy 为 true，不进行求值，直接返回 undefined
        this.value = this.lazy
            ? undefined
            : this.get()
    }
}
```

执行构造函数时指定 `lazy` `dirty` 为 true，最后执行 watcher.value = undefined 并未执行 `get` 方法进行求值。什么时候求值呢？后面就知道了。

回到上边，为计算属性创建内部 watcher 之后的 watchers 对象是这样的：

```javascript
{
    text: Watcher {
        lazy: true,
        dirty: true,
        deps: [],
        getter: function () {
            return `Hello, ${this.name}!`
        },
        value: undefined, // 直接赋值为 undefined ，
    }
}
```

### 3. 定义计算属性的 getter

看一下 `defineComputed` 做了什么：

```javascript
function defineComputed(target, key, userDef) {
    Object.defineProperty(target, key, {
        get: function () {
            const watcher = this._computedWatchers && this._computedWatchers[key]
            if (watcher) {
                if (watcher.dirty) {
                    watcher.evaluate()
                }
                if (Dep.target) {
                    watcher.depend()
                }
                return watcher.value
            }
        }
    })
}
```

有没有很熟悉，在定义响应式数据的 `defineReactive` 方法中也使用了 `Object.defineProperty` 方法来定义访问器属性。这里在该 vm 实例上定义了 text 属性，每当访问到 `this.text` 时就会执行对应的 getter ，函数做了什么暂时可以不看。那什么时候会读取到 `this.text` 呢？答案在下边。

总结一下 computed 初始化过程：

- 定义 `vm._computedWatchers` 用来存放该 vm 实例的所有 computedWatcher
- 遍历 computed 选项并实例化 watcher，不求值，直接将 watcher.value = undefined
- 通过 `defineComputed` 定义计算属性的 getter ，等待后边读取时触发

## 首次渲染

初始化完成后，会进入 mount 阶段，在执行  ` render` 生成 vnode 时会读取到计算属性 `text` ，本文示例的 render 函数是这样：

```javascript
function render() {
    var h = arguments[0];
    return h("div", [
        h("h2", [this.text]), // 这里读取了计算属性 text
        h("button", {
            "on": {
                "click": this.changeName
            }
        }, ["changeName"]),
    ]);
}
```

### 1. 触发计算属性的 getter

这时会触发计算属性的 getter ，也就是上边定义的访问器属性：

```javascript
get: function () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
        // 此时 dirty 为 true ，进行求值
        if (watcher.dirty) {
            // 求值，对 data 进行依赖收集，使 computedWatcher 订阅 data
            // 这里的 data 就是 "this.name"
            watcher.evaluate()
        }
        if (Dep.target) {
            watcher.depend()
        }
        return watcher.value
    }
}
```

### 2. 求值 watcher.evaluate()

取出 `vm._computedWatchers` 中对应的 watcher ，此时 `watcher.dirty` 为 true，执行 `watcher.evaluate() ` 。

```javascript
// watcher.evaluate
evaluate() {
    this.value = this.get()
    this.dirty = false
}
```

这个函数做了两件事：

- 执行 get 进行求值，这里就解答了上面何时求值的问题；
- 将 dirty 置为 false 。

先看求值：

```javascript
// watcher.get
get() {
    pushTarget(this) // Dep.target 置为当前 computedWatcher
    let value
    const vm = this.vm
    try {
        // 触发响应数据的依赖收集
        value = this.getter.call(vm, vm)
    } catch (e) {
        // ...
    } finally {
        popTarget() // Dep.target 重新置为渲染 watcher
    }
    return value
}
```

这里需要知道一个前置内容，全局的 `Dep.target` 存的是当前正在求值的 watcher，它是用一个栈 `targetStack` 来维护的。当前是在渲染过程中，所以此时栈是这样：[ 渲染watcher ]。

```javascript
function pushTarget(target: ?Watcher) {
    targetStack.push(target)
    Dep.target = target
}
```

首先 `pushTarget(this) ` ， `Dep.target` 成为当前 computedWatcher，此时栈是这样的：[ 渲染watcher, computedWatcher ]。

然后执行 getter 触发响应数据的依赖收集。再回顾一下 getter ：

```javascript
function () {
    return `Hello, ${this.name}!`
}
```

很显然执行这个函数会读取 `this.name` ，name 的 dep 就会收集当前 computedWatcher ，当前 computedWatcher 就会订阅 name  的变化（这里不做详细介绍，参考响应式原理的依赖收集）。

收集完成后执行 `popTarget()` ，`Dep.target` 重新成为渲染 watcher ，此时的栈是这样：[ 渲染watcher ]。

然后返回计算得出的值，再将 dirty 置为 false 。此时 name 的 dep 是这样的：

```javascript
{
    id: 3,
    subs: [ computedWatcher ], // 收集了 computedWatcher
}
```

computedWatcher 是这样的：

```javascript
{
    dirty: false, // 求值完成，dirty 置为 false
    deps: [ name 的 dep ], // 订阅了 name
    value: "Hello, xiaoming!",
}
```

### 3.  watcher.depend()

此时计算属性的 get 访问执行到了这里：

```javascript
get: function () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
        // 求值...
        // 执行到了这里
        if (Dep.target) {
            watcher.depend()
        }
        return watcher.value
    }
}
```

此时  `Dep.target` 是渲染 watcher ，所以会执行 `watcher.depend()` 。

```javascript
// watcher.depend
depend() {
    let i = this.deps.length
    while (i--) {
        this.deps[i].depend()
    }
}
```

可以看到将 computedWatcher 订阅的 deps 依次执行 `dep.depend` ，熟悉响应式原理的应该马上就知道了，这是在对这些 dep 的响应式数据进行依赖收集，也就是对示例中的 `name` 进行依赖收集，收集的是谁呢？上面提到此时的 `Dep.target` 是渲染 watcher ，那么总结下来，这一步做的是：

**让 computedWatcher 订阅的响应式数据收集渲染 watcher**

这一步操作之后 `name` 的 dep 是这样的：

```javascript
{
    id: 3,
    subs: [ computedWatcher, 渲染 watcher ], // 收集了渲染 watcher
}
```

最后，返回 watcher.value ，get 访问结束，`render` 函数继续往下走，之后渲染出最终页面。

## 触发更新

当点击按钮时，执行 `this.name = 'onlyil'` ，会触发 `name` 的访问器属性 set ，执行 `dep.notify()` ，依次触发它所收集的 watcher   的更新逻辑，也就是 `[ computedWatcher, 渲染 watcher ]` 的 update 。

### 1. 触发 computedWatcher 更新

```javascript
// watcher.update
update() {
    // computedWatcher 的 lazy 为 true
    if (this.lazy) {
        this.dirty = true
    }
    // ...
}
```

只做了一件事，就是将 dirty 置为 true，表示该计算属性“脏”了，需要重新计算，什么时候重新求值呢，往下看。

### 2. 触发渲染 watcher 更新

```javascript
// watcher.update
update() {
    // ...
    // 
    queueWatcher(this)
}
```

这里就是加入异步更新队列，最终又会执行到 `render` 函数来生成 vnode ，同首次渲染一样，在 `render` 过程中又会读取到计算属性 `text` ，再次触发它的 getter ：

```javascript
get: function () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
        // 此时 dirty 为 true ，"脏"了
        if (watcher.dirty) {
            // 重新求值
            watcher.evaluate()
        }
        // ...
        return watcher.value
    }
}
```

重新求值也就会重新执行我们在 computed 选项定义的函数，页面就展示了新值 `Hello, onlyil!` 

```javascript
function () {
    return `Hello, ${this.name}!`
}
```

至此，更新结束。

## 如何缓存

通过上边的过程分析，可以做出如下总结：

- 首次渲染时实例化 computedWatcher 并定义属性 `dirty: false` ，在 render 过程中求值并进行依赖收集；
- 当 computedWatcher 订阅的响应式数据也就是 `name` 改变时，触发 computedWatcher 的更新，修改 `dirty: true` ；
- render 函数执行时读取计算属性 `text` ，发现 `dirty` 为 true ，重新求值，页面视图更新。

可以发现一个关键点，computedWatcher 的更新只做了一件事：修改 `dirty: true` ，求值操作始终都在 render 过程中。

现在我们修改示例，新增一个数据 `count` 和方法 `add` ：

```vue
<div id="app">
    <h2>{{ this.text }}</h2>
    <h2>{{ this.count }}</h2>
    <button @click="changeName">Change name</button>
    <button @click="add">Add</button>
</div>

const vm = new Vue({
    el: '#app',
    data() {
        return {
            name: 'xiaoming',
			count: 0,
        }
    },
    computed: {
        text() {
            return `Hello, ${this.name}!`
        }
    },
    methods: {
        changeName() {
            this.name = 'onlyil'
        },
		add() {
            this.count += 1
        },
    },
})
```

点击 Add 按钮 `count ` 会发生改变，那么在重渲染时 computedWatcher 会重新求值吗？

答案是不会，缓存的关键就在于此。回头再看下文章开头引用 Vue 文档里一句话：

> **计算属性是基于它们的响应式依赖进行缓存的**

结合上边的分析是不是恍然大悟，计算属性 `text` 的 getter 函数并没有读取 `count` ，所以它的 computedWatcher 不会订阅 `count` 的变化，即 `count` 的 dep 也不会收集该 computedWatcher 。

所以当 `count ` 改变时，不会触发 computedWatcher 的更新，`dirty` 仍为 false ，说明这个计算属性不“脏”。那在之后的 render 过程中读取到计算属性 `text` 时就不会重新求值，这样就起到了缓存的效果。

## 后记

整个过程下来脑子还是有点吃力的，但多来几遍就会逐渐理解 computed 实现的巧妙之处。其实缓存的原理很简单，就是一个标志位而已～🤣

注：文中代码做了删减，结合源码断点捋捋更佳。

