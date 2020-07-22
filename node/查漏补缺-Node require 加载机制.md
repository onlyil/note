## 前言

项目工程里一直用的老版本的 `promise-polyfill` ，很坑的是它只支持 `then` `catch` 不支持 `finally` ， 所以打算使用 `es6-promise` 。`es6-promise` 支持判断当前环境是否存在 `Promise` ，不存在时才会进行 polyfill :

```javascript
require('es6-promise/auto');
```

随手点进去看了看源码：

```js
// auto.js
module.exports = require('./').polyfill();
```

`require('./')` 这个知道，在当前目录下找下 `index` ，嗯？没有 `index` 命名的文件？这是第一个盲点，答案是找到 package.json 中的 main 指定的文件。

然后我又打开了 `lib/es6-promise.auto.js` :

```javascript
// lib/es6-promise.auto.js
import Promise from './es6-promise';
```

同样的，打开 `es6-promise/` 目录，嗯？还是没有 `index` ，当然这里也不会有 package.json ，这么骚的操作吗，没听说过啊。。。

好吧实际上是我眼瞎没看到同级目录还有一个 `es6-promise.js` 。。。

```shell
├── lib
│   ├── es6-promise
|	│   ├── polyfill.js
|	|	└── then.js
│   ├── es6-promise.js
│   └── es6-promise.auto.js
```

这个时候就必须要搞清楚 require 究竟是如何工作的了。

## require

当在路径为 Y 的文件里 `require(X)` 时，按下面的顺序处理：

### 1. 如果 X 是一个核心模块

a. 返回这个核心模块（内置模块）

b. 停止

### 2. 如果 X 以 “/” 开头

将路径 Y 设置为文件系统的根路径 

### 3. 如果 X 以 “./” “/” “../” 开头

a. 将 X 当作文件，如下依次查找，如果存在就返回，然后停止

- X
- X.js
- X.json
- X.node

b. 将 X 当作目录，如下依次查找，如果存在就返回，然后停止

- X/package.json 的 main 字段
- X/index.js
- X/index.json
- X/index.node

### 4. 如果不带路径

a. 从当前模块的 node_modules 依次往上层查找

b. 在 node_modules 中将执行步骤 3 

### 5. 抛出错误 "not found"

---

详细过程参考 Node 官网：https://nodejs.org/api/modules.html#modules_all_together

