### 新旧节点不同

- 创建新节点
- 更新父占位节点(`create` 钩子函数)
- 删除旧节点

### 新旧节点不同

patchVnode

- 执行 `prepatch` 钩子函数(占位符 `vm.$vnode` 的更新、`slot` 的更新，`listeners` 的更新，`props` 的更新等等)

- 执行 `update` 钩子函数

- 完成 `patch` 过程

  - 找到对应的真实dom，称为`el`

  - 判断`Vnode`和`oldVnode`是否指向同一个对象，如果是，那么直接`return`

  - 如果他们都有文本节点并且不相等，那么将`el`的文本节点设置为`Vnode`的文本节点。

  - 如果`oldVnode`有子节点而`Vnode`没有，则删除`el`的子节点

  - 如果`oldVnode`没有子节点而`Vnode`有，则将`Vnode`的子节点真实化之后添加到`el`

  - 如果两者都有子节点，则执行`updateChildren`函数比较子节点，这一步很重要
    - `oldCh`和`vCh`各有两个头尾的变量`StartIdx`和`EndIdx`，一共有4种比较方式，这一步是为了快速复用
    - 如果4种比较都没匹配：
      - 如果新旧子节点都存在key，那么会根据`oldChild`的key生成一张hash表，节点与hash表做匹配，匹配成功就判断新节点和匹配节点是否为`sameNode`，如果是，就在真实dom中将成功的节点移动位置，否则，将`S`生成对应的节点插入到dom中对应的位置。
      - 如果没有key,则直接将`S`生成新的节点插入`真实DOM`（ps：这下可以解释为什么v-for的时候需要设置key了，如果没有key那么就只会做四种匹配，就算指针中间有可复用的节点都不能被复用了）

- 执行 `postpatch` 钩子函数

