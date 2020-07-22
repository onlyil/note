## 速度优化

### 利用缓存

#### 1. babel-loader

使用`loader: 'babel-loader?cacheDirectory=true'`开启缓存，babel-loader 将使用默认的缓存目录 `node_modules/.cache/babel-loader`，之后的 webpack 构建将会尝试读取缓存，来避免在每次执行时可能产生的高性能消耗的 Babel 重新编译过程。

```javascript
{
    test: /\.js$/,
    exclude: /(node_modules)/,
    loader: 'babel-loader?cacheDirectory=true',
},
```

#### 2. HardSourceWebpackPlugin

[HardSourceWebpackPlugin](https://github.com/mzgoddard/hard-source-webpack-plugin) 可以对模块进行缓存，对于二次构建的速度提升非常显著。

### 多进程构建

#### 1. thread-loader

[thread-loader](https://github.com/webpack-contrib/thread-loader) 可以使其之后的 loader 在一个单独的 worker pool 中运行，每个 worker 都是一个单独的有 600ms 限制的 node.js 进程

```javascript
{
    test: /\.js$/,
    exclude: /(node_modules)/,
    use: [
      'thread-loader',
      'babel-loader?cacheDirectory=true',
    ],
},
```

#### 2. happypack

[happypack](https://github.com/amireh/happypack) 会在 webpack 每次解析一个模块时都会拿到该模块和它的依赖，将它们分发给多个 node 子进程，处理完后再将结果返回给主进程。

#### 3. 并行压缩

webpack 在 production 环境默认使用 [TerserPlugin](https://github.com/webpack-contrib/terser-webpack-plugin) 压缩 js，配置选项 `parallel: true` 开启并行压缩：

```javascript
optimization: {
    minimizer: [
        new TerserPlugin({
            parallel: true,
        }),
    ],
},
```

### 预编译

通过使用 [DLLPlugin](https://webpack.js.org/plugins/dll-plugin/) 和 DLLReferencePlugin 拆分 bundles大大提升构建的速度。通常像 vue 这样不常发生变动的库可以预编译为单独的包给 html 引用，这样就避免 webpack 频繁地构建这些库，大大节省编译时间。

1.使用如下单独配置打包 dll.js
```javascript
// webpack.dll.js
module.exports = {
    mode: 'none',
    entry: {
        vendors: [
            'vue',
            'vue-router',
        ],
    },
    output: {
        path: path.join(__dirname, './static/lib'),
        filename: '[name].dll.js',
        library: '[name]_[hash:8]',
    },
    plugins: [
        new webpack.DllPlugin({
            name: '[name]_[hash:8]',
            path: path.join(__dirname, './static/lib/[name]-manifest.json'),
        }),
    ],
}
```
2.在 webpack 配置中添加 `DllReferencePlugin` 插件：

```javascript
new webpack.DllReferencePlugin({
    manifest: require('./static/lib/vendors-manifest.json'),
}),
```

3.在 html 中引入 dll.js：

```html
<script src="../static/lib/vendors.dll.js"></script>
```

## 体积优化

### 提取公共模块

[SplitChunksPlugin](https://webpack.js.org/plugins/split-chunks-plugin/) 是 webpack4 升级的最重要的配置之一，它的作用是进行代码拆分，我们可以立利用它提取公共模块。

```javascript
optimization: {
    splitChunks: {
        chunks: 'all',
        name: 'commons',
    },
},
```
### 预编译

同上。

### 其他

webpack 在 production 会默认开启一些优化功能：
#### tree shaking

- 生产环境默认开启（“usedExports” 为 true ）
- package.json 有一个特殊的属性 sideEffects 
  - `true` 是默认值，这意味着所有的文件都有副作用，也就是没有一个文件可以 tree-shaking
  - `false` 告诉 Webpack 没有文件有副作用，所有文件都可以 tree-shaking
  - 第三个值 `[…]` 是文件路径数组。它告诉 webpack，除了数组中包含的文件外，你的任何文件都没有副作用

- Webpack 使用它的模块规则系统来控制各种类型文件的加载。每种文件类型的每个规则都有自己的 `sideEffects` 标志。这会覆盖之前为匹配规则的文件设置的所有 `sideEffects` 标志
- 基于 ES Module ，所以需要告诉 babel 不要将模块转化为 commonjs 模块

#### scope hoisting

## 代码/性能优化

### 1. 动态导入

动态导入 (dynamic import) 是 webpack 实现 [懒加载](https://webpack.js.org/guides/lazy-loading/)（按需加载）的方式，会先把代码在一些逻辑断点处分离开，然后在主代码块中完成某些操作后，通过 `requireEnsure` 方法动态创建 script 标签加载相应 js。vue 中可以使用 [异步组件](https://cn.vuejs.org/v2/guide/components-dynamic-async.html#异步组件) 实现组件的懒加载。

### 2. 预取/预加载模块

Webpack v4.6.0+ 增加了对预获取和预加载 (prefetch/preload module) 的支持。

在使用上述动态导入是通过魔术注释告知 webpack 。例如下面的 vue 异步组件，通过 `/* webpackPrefetch: true */` 标记为需要 prefetch 的模块资源：

```javascript
{
	components: {
        MyText: () => import(/* webpackPrefetch: true */ './components/MyText'),
    },
}
```

就会在页面 head 中添加：

```html
<link rel="prefetch" as="script" href="components/MyText.chunk.js">
```

---

webpack 官网也提供了一些改进构建/编译性能的实用技巧：[构建性能](https://webpack.docschina.org/guides/build-performance/)。

