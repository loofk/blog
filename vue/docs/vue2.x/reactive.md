# 响应式原理

## 响应式对象

核心是`Object.defineProperty`方法，通过挟持对象属性（给对象属性添加get、set方法），每次取值或者改值的时候都执行额外的操作

Vue在初始化过程中，会把props和data都变成响应式对象，如果属性也是对象则递归把该对象也变成响应式

## 依赖收集

我们知道响应式对象就是挟持对象属性，接下来我们分析get方法中干了什么，在组件挂载过程中会执行patch函数将VNode渲染到真实DOM中，render函数调用期间就会从data、props中取值，也就是触发了我们设置的get方法，get方法主要做两件事：一件事是返回该属性所对应的值，这也是其本职工作；另一件事就是记录谁调用了该属性，通俗点就是谁使用了我，我以后变了该通知谁，这也就是所谓的依赖收集

get的依赖收集实现的很巧妙，借用了Dep这个中间类，每个被监听的属性都实例化了这个类dep并作为闭包保存在内存中，dep对象负责记录哪些watcher使用了该属性，具体做法是调用闭包dep对象的depend方法，在之前Dep类维护了一个全局静态变量`target`，记录当前渲染/计算的watcher，所以depend方法实际上是把自身（dep）作为参数传入Dep.target的addDep方法中，然后在当前渲染watcher的addDep方法中将自身（watcher）添加到闭包dep的subs数组中

听起来有点绕，为啥不直接把Dep.target加到subs中呢？这是因为我们还需要在watcher实例中处理这个dep，我们在`watcher.js`中初始化了这几个属性，下面用例子来解释它们的作用
```
this.deps = [] // 上一次渲染用到的每个属性的闭包dep集合
this.newDeps = [] // 当前渲染用到的每个属性的闭包dep集合
this.depIds = new Set() // 上一次渲染用到的每个属性的闭包id集合
this.newDepIds = new Set() // 当前渲染用到的每个属性的闭包id集合
```

举一个很常见的例子，一个数据在模板中多次使用，这样每次使用都会触发addDep，这时就需要在watcher实例中用newDepIds和newDeps记录当前已经被订阅过的数据，通过depIds判断避免重复订阅
```
addDep (dep: Dep) {
  const id = dep.id
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id)
    this.newDeps.push(dep)
    // 避免重复订阅
    if (!this.depIds.has(id)) {
      dep.addSub(this)
    }
  }
}
```

另一个例子是如果我们进行条件渲染，模板两个地方使用的数据部分不同，就会存在前一次渲染中用到的数据后面已经不再使用的情况，即这些数据的变化已经对渲染无用了，这时就需要移除这些数据上的订阅：具体做法是在cleanupDeps方法中，每次watcher实例收集完依赖时，就判断上次渲染的数据deps中有哪些是这次渲染的数据newDepIds中不存在的，不存在则移除该订阅，并保存当前渲染数据到deps和depIds，最后清空newDeps和newDepIds，方便下次渲染
```
cleanupDeps () {
  let i = this.deps.length
  // 当原来更新过的属性已经不在本次渲染用到的数据集合中时，将该属性的订阅移除，防止无谓的渲染
  while (i--) {
    const dep = this.deps[i]
    if (!this.newDepIds.has(dep.id)) {
      dep.removeSub(this)
    }
  }
  // 把newDepIds和newDeps赋值给depIds和deps，然后清空newDepIds和newDeps
  // 也就是说每次都删除上次的订阅，保留当前订阅
  let tmp = this.depIds
  this.depIds = this.newDepIds
  this.newDepIds = tmp
  this.newDepIds.clear()
  tmp = this.deps
  this.deps = this.newDeps
  this.newDeps = tmp
  this.newDeps.length = 0
}
```

## 派发更新

set函数的一个细节，针对没有设置setter方法的属性，Vue直接把新值赋值给val，这样也实现了对该属性的修改，这是因为val这个变量是闭包，在set方法中修改其值，等到取值时，在get方法中再使用闭包val就已经是新值了
```
// val的初始赋值
if ((!getter || setter) && arguments.length === 2) {
  val = obj[key]
}

// get
const value = getter ? getter.call(obj) : val

// set
if (setter) {
  setter.call(obj, newVal)
} else {
  val = newVal
}
```

派发更新简单来说就是数据修改触发set，然后调用dep实例的notify通知订阅更新的watcher实例列表，触发其更新，每个wacther都有一个update方法，该方法在正常情况下会将自身推入队列queue中，然后在下一个tick触发更新，具体点是执行watcher实例的run方法，其中会调用get方法求值，如果是渲染wathcer其`getter`就是updateComponent，因此会调用传入的updateComponent方法触发重新渲染，关键代码是下面这一句：
```
vm._update(vm._render(), hydrating)
```

注意，其中的_render函数会在生成VNode节点时引用数据，再一次进行依赖收集，其实首次渲染时也会执行，进行初次的依赖收集

派发更新使用队列进行了渲染的优化，首先使用has数组记录wathcer的id确保不会重复添加（其实并不能完全避免，后面会提），然后使用waiting变量避免不会重复触发flushSchedulerQueue，这个设计很巧妙，因为JS是单线程的，所以在执行`nextTick(flushSchedulerQueue)`前，把waiting置为true，确保后面调用queueWatcher方法不会再执行nextTick，这样就能把数据的更新渲染统一推迟到下一个tick执行，提高效率
```
if (!waiting) {
  waiting = true

  if (process.env.NODE_ENV !== 'production' && !config.async) {
    flushSchedulerQueue()
    return
  }
  nextTick(flushSchedulerQueue)
}
```

执行flushSchedulerQueue时，首先对queue进行从小到大排序，这是出于`先父后子`、`先用户watcher后渲染watcher`、`父watcher在run时会销毁子watcher`三点考虑的，接着遍历queue，这里做了一个无限渲染判断，即我们思考这样一种情形，在用户watcher监听某个数据时，其回调再次修改了该数据，这会导致在执行回调时又一次触发queueWatcher并将该用户watcher插入queue从而陷入无限循环，Vue在检查到某个wathcer的id被run了100次以上时会主动跳出遍历并报错

## nextTick

nextTick的实现利用了JS的事件循环，针对浏览器的支持情况不断降级，先尝试Promise，不支持再尝试MutationObserver，接着setImmediate，最后setTimeout，执行的nextTick将回调函数推入callbacks回调队列并在下一tick同步执行

由于Vue内部更新时也使用了nextTick，如果我们主动调用nextTick，渲染的回调和用户回调会按照推入callback的先后依次执行，这表明我们最好在修改数据后再调用nextTick获取修改后的数据

## 响应式对象的注意事项

在把对象包装成响应式对象时，会检测对象中某个属性的值是否仍为对象，满足则会对该值进行响应式，这意味着我们在初次渲染时就会对所有使用的数据（包括嵌套对象和数组）都收集依赖，这其实已经可以满足一些需求了，但我们注意到，Vue在收集依赖时还会判断当前操作响应式对象的某个属性是否为对象，并调用子对象的observe对象上的dep属性进行依赖收集，这看起来不明就里，按照常理该嵌套对象已经是响应式对象，其内部的每个属性都可以通过其闭包dep派发更新了，那么这个‘多余的dep属性’是用在哪里呢？

我们知道由于JS的限制（插一句，对于对象，由于是在初始化的时候对data、prop中的数据进行响应式，所以后来的属性不具备响应式是正常的，但是对于数组来说，我们操作下标修改数组元素无法获得响应式是不合常理的，由源码可知，Vue并没有对数组的key进行响应式，而是直接对元素进行响应式，Vue的开发者尤大回应是由于性能原因，推测我们一般是使用数组来储存列表这种数据量大的数据，相对于对象来说，一个个转化响应式的开销不值得，不如在有需要时调用API触发响应式，另一个原因是数组的长度不可控制，容易引发意料之外的问题），Vue无法跟踪新增属性和直接操作数组下标所引发的变化，那么我们就需要实现API来手动触发，Vue暴露了set函数，内部流程是根据传入的参数修改对象，然后把该新值转化为响应式属性，由于我们无法在definReactive函数外部取得闭包dep，当前操作的响应式对象的ob属性中保存的dep属性就有用了，通过它派发更新就能触发重新渲染

准确来说dep对象在响应式对象本身是保存在__ob__属性的dep属性中的，而在属性上则是通过闭包存在内存中

## props

### 初始化

首先我们对于组件的props选项在合并组件配置时，会进行规范，因为我们可以传入数组或者对象作为props，数组情况我们把其中的每个元素作为key，转化为值为包含属性`type: null`的对象；而对于对象，则看每个key的值是否是对象，如果是则直接将值转化成属性形如`type: Boolean`的对象

规范化后的props是一个对象，其内部每个key的值也为对象，接着我们在initState中处理规范后的props对象，首先校验每个prop是否符合prop定义
- 如果key的类型是Boolean，单独处理，这块逻辑有点乱，下面详细讲
- 如果不是Boolean类型，其他类型不传值则为undefined，会去找该prop定义时的默认值并把它设置成响应式对象
- 最后对值进行一系列的断言，比如看值是否符合预期类型，如果传入了用户自定义的validator函数，则调用该函数，不满足报错

针对prop是Boolean类型的情况：
1. 父组件在调用子组件时如果没有设置该prop且该prop没有设置默认值，则把值设为false
2. 如果设置了该prop但没有传值或者传值等于key本身，则判断预期类型数组（如果存在的话）中是否有String，满足未设置String类型或者预期的类型数组中Boolean在String前面，置为true
3. 如果传入其他值，则会在props断言中检查值是否是Boolean类型，不是则报错，但不影响值的使用

校验完成后，我们拿到prop的值，把prop变成响应式对象

在设置响应式时提供了一个setter函数，如果我们在组件内部修改prop的值就会提示最好不要直接在组件中修改prop的值，应该在父组件中修改，报错只针对用户修改的情况，系统修改时不报错

这里还进行了一个优化，如果不是根实例在响应式时不对值为对象或数组的情况再进行observe，后面再组件更新时详述

响应式之后我们进行代理，把_props上的prop全部代理到当前实例vm上，方便调用，其实对于组件并不是在initProps方法中进行代理，而是发生在extend时

### props更新

我们知道，修改父组件中的某个数据，会触发父组件的重新渲染，如果这个数据作为prop传入子组件中，也会触发子组件的重新渲染，这个子组件的渲染是如何完成的呢？

父组件重新渲染最终会进行patch，在patchVnode方法中遇到组件节点时，会执行组件的prepatch钩子函数，内部执行updateChildComponent方法更新props，这个时候发生对props的重新赋值，触发props的set方法通知子组件重新渲染，值得注意的是，在传入对象作为prop时，因为传入的对象在父组件中已经进行了响应式，而子组件在渲染时会访问其内部属性，触发getter收集子组件的render watcher，这样在属性变更时就会提前通知子组件触发重新渲染，等到执行updateChildComponent方法时，因为对象数据是引用的已经被修改，无法触发set通知子组件；如果未传值，但默认值为对象，则会对这个对象进行响应式，这也能监听到对象内部属性的变更触发子组件重新渲染

这也是toggleObserving在props中频繁调用的原因，在initProps方法和updateChildComponent方法中，由于引用类型已经在父组件被递归响应式处理，所以不需要再次响应式了，而对于prop默认值为对象的情况则需要递归对其进行响应式处理

## computed和watch

computed和watch其本质都是watcher实例，我们在定义computed、watch或者调用watchAPI时都是实例化watcher并在取值期间进行依赖收集，这样数据变更时就能马上重新计算或执行回调

对于computed来说，它生成的watcher的lazy属性是为真的，这和普通的wathcer有两点不同，一是在实例化时不会马上求值，而是等到被引用时才求值，求值结果保存在wathcer的value属性中，只有等到其内部依赖变化时才会重新求值并找到当前引用它的watcher实例进行依赖收集；二是派发更新时不会执行run方法，只是重置dirty属性以便重新求值

更新：computed有过两个版本的变更，原来的实现是只要computed内部的数据变化了就重新求值，如果是在模板中使用了则会重新渲染页面；后面针对这个点进行了优化，实现后可以做到只有在computed的计算结果更新时才触发页面渲染，然而后面的实现引发了一系列BUG，具体见 [8446](https://github.com/vuejs/vue/issues/8446)，最终回退为原实现

对于watch来说简单点，它支持传入对象或者函数，对象的话可以指定deep和immediately属性，指定immediately属性时，会在创建时马上执行回调，指定deep时，则会在取值时深度遍历，对嵌套对象内部的属性进行依赖收集；watch在定义时会依据属性名（如果是函数则是函数名）在当前组件实例中寻找求值并进行依赖收集，接着在数据变更时派发更新，统一在下一个tick执行run方法，不同于渲染wathcer，在调用run中的get方法重新求值后，会执行用户回调

watch中deep属性的原理，我们前面了解了在初始化组件实例时，会对data（包括自身）中的每个属性都创建一个dep类，嵌套对象会为子对象创建一个__ob__属性同时也创建一个dep类，但是在watch取值时，我们是根据key来取值的，如果我们为一个属性设置侦听，但它的值是一个对象，我们只能收集到这个属性的dep和其子对象的dep，对于子对象中的属性以及更深层次的属性是无法收集到的，这时如果设置了watcher的deep属性为真，Vue会调用深度遍历，判断侦听的属性值是否为对象并层层取值，把内部所有属性的dep都进行依赖收集，这样就能侦听深层次的数据变更
