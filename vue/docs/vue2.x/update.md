# 更新视图

页面渲染更新的大致流程

![渲染流程图](../img/render.PNG)

## 虚拟DOM

虚拟DOM由真实DOM节点衍生而来，Vue的虚拟dom借鉴了开源库 snabbdom

为什么我们使用虚拟DOM呢，这是因为真实的DOM节点被设计成非常庞大复杂，如果直接操作真实DOM就会引发性能问题，但如果把DOM中的关键属性如标签名、属性值、内容、子节点等抽象出来，构造成一个较小的对象`VNode`，就可以操作这个轻量的对象，得到最终结果后再操作DOM，提升性能

## 更新策略

### 前后父节点不是同类节点

比如模板中通过变量控制不同类型的节点展示的例子，我们分为三步：第一步根据新生成的vnode创建出新的DOM节点；接着我们要做的就是修改父VNode上组件占位符VNode的elm，并在修改前后调用DOM的各个属性（类、样式、属性）的钩子，如果发现占位符VNode的parent仍为占位符VNode，则表示组件嵌套组件（一个组件的根节点是另一个组件），就需要递归到最上层的占位符节点，最终会回溯到body上并将生成的DOM节点挂载到body上；最后我们在父DOM上删除掉老节点

### 前后父节点是同类节点

比如我们只是修改组件中某个DOM的文本或者展示顺序，这时就需要patchVnode了，patchVnode的主要逻辑分为两种情况：

新节点是文本节点：这种情况直接把elm设置成对应文本就行了

新节点不是文本节点

- 如果新节点存在子节点而老节点不存在子节点，我们再判断一下老节点是否为文本节点，如果是则先清空文本，再把子节点添加到DOM中，否则直接添加子节点到DOM中
- 如果只存在老节点的子节点，则直接从DOM中删除所有老节点的子节点
- 如果新老节点都不存在子节点，再判断一下老节点是否为文本节点，是的话从DOM中清空文本
- 如果新老节点都存在子节点，则进入子节点的更新流程updateChildren


updateChildren的主要逻辑分为四种情况

1. 比较新老节点的开始节点，如果是同类节点则patchVnode这两个节点，然后开始下标都向后移一位
2. 比较新老节点的结束节点，如果是同类节点则patchVnode这两个节点，然后结束下标都向前移一位
3. 比较老节点的开始子节点和新节点的结束子节点，如果是同类节点则patchVnode这两个节点，并且把老节点的开始节点插入到老节点的结束节点的下一个节点（也就是null），实际上是插入到末尾，然后老节点开始下标往后移一位，新节点结束下标往前移一位
4. 比较老节点的结束子节点和新节点的开始子节点，如果是同类节点则patchVnode这两个节点，并且把老节点的结束子节从末尾插入到开头，然后把老的子节点结束下标往前移一位，新的子节点开始下标往后移一位，

我调试了两种分支的情况，发现了一些有趣的事实
```
<li v-for="item in items" :key="item.id">{{ item.val }}</li>

items: [
  {id: 0, val: 'A'},
  {id: 1, val: 'B'},
  {id: 2, val: 'C'},
  {id: 3, val: 'D'},
]

this.items.reverse().push({id: 4, val: 'E'},)
```
上面的例子由于以id为key，在比较相同节点时会命中updateChildren的第四种情况，就会去比较li内部的文本节点是否相等，结果是相等的，所以文本节点的patchVnode什么也没干，回到外部执行insertBefore操作，将老节点D插入到老节点的开始子节点之前（也就是开头），这样页面就变成了`D -> A -> B -> C`

```
<li v-for="(item, index) in items" :key="index">{{ item.val }}</li>

items: [
  {id: 0, val: 'A'},
  {id: 1, val: 'B'},
  {id: 2, val: 'C'},
  {id: 3, val: 'D'},
]

this.items.reverse().push({id: 4, val: 'E'},)
```
这种情况就比较有意思了，因为是以index为key，所以在比较相同li节点时直接命中第一种情况，接着进入li内部文本节点的patchVnode，发现文本不同，将老节点的开始节点的文本更新成了D，这样页面就变成了`D -> B -> C -> D`

**注意，对于文本节点而言，sameVnode的返回值都是true**

## 操作DOM的API封装

Vue是一个跨平台的MVVM框架，不仅支持浏览器，还支持WEEX，但这两个平台的操作DOM方法不一致

Vue的实现借助了createPatchFunction这个工厂函数，外部传入不同的操作node的API和DOM属性模块钩子，生成不同平台的patch函数，这给了我们开发的一个启示，如果涉及到某个功能在总体流程都是一致的，只是具体的操作方法处理方式不一致，可以在外层封装好统一格式的API，以参数的形式传入内部实现函数调用，将差异在外层就进行消除，避免在实现核心逻辑时条理不清晰，代码充斥着一堆if/else
