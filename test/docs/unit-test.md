## 什么是单元测试

单元测试，你可以理解为将你的代码切割到最小单元后，对这些单元进行的断言，即这个最小单元能否满足你的期望

原来我所理解的单元测试是针对每一个函数进行的输入测试，后面发现不仅可以是函数，也可以是一个类或者其他你觉得可以当成一个独立的黑盒式的模块，我们不需要去关心其内部实现、逻辑，我们要做的很简单，提供该单元所需要的所有服务，断言其输出是否满足我们的需求

由于单元测试是整个系统（项目）中最底层的环节，不同于其上的服务测试和UI测试，单元测试涉及到的模块关联是最小的，整个系统被划分为成百上千个单元，我们针对这些单元的测试都通过，则可以明确的在软件设计阶段把BUG数降到最小，这极大的减少了整个软件开发的生命周期

编写单元测试的过程实际上是程序员编码过程中的自检，检查自己的逻辑有无问题，对该需求是否理解的足够深刻，对编码的每一个环节是否考虑周全，此外，好的单元测试可以取代原有的手动回归测试，原来的每次改动需要重复进行的测试，可以交给计算机几秒内就检验完成

编写单元测试同时也是对软件设计的一种规范，优秀的代码是可维护和可阅读的，单元测试正是保证这两点的最有效武器，因此，如何编写单元测试，如何让自己的代码更易于编写单元测试是每个程序员在动手之前需要考虑清楚的，一旦你开始思考这些问题，在编码时你就会有意识的运用模块化技巧和函数式编程

好的单元测试和优秀的代码是相辅相成的，正如前面提到的，模块化的代码让你编写单元测试时毫不费力，而好的单元测试让你在后续修改代码时无需执行重复的人工校验，并且能够确认修改是否背离需求

也许很多情况，你觉得代码不需要单元测试，这是令人信服的，因为我们实现一个类似于简单计算器的功能，代码不超过100行，对这些代码的每个环节你在后续都能很快理解和修改，也能很容易的添加新的功能，你觉得这是不需要单元测试的

但是当面对一个大型的后台管理系统时，整个系统功能繁多，十分复杂，函数成百上千，如果不对每个函数进行测试，在系统测试阶段，所耗费的排查时间无法想象，随着后续的迭代和修改，后面的问题排查时间几乎是指数级增长的

## 为什么选择Jest

Jest是Facebook推出的一款单元测试框架，相对于其他测试框架，其最大的特点是开箱即用，内置的断言库、测试覆盖率工具让我们不需要关心其他，只要掌握语法，就能轻松上手

这里我们粗略了解一下前端的测试相关工具

### 断言库

chai，知名的断言库，提供形如Should、Expect、Assert等基本的断言关键字

### 测试框架

karma，测试驱动，非jsdom，基于本地的client/server，提供真实的浏览器环境，用来运行测试用例

mocha，比较老的库，提供基本的测试API，如describe，通常搭配chai使用

jasmine，比mocha早，API基本相同，不同于mocha用于支持NodeJs，它需要karma或者Chutzpah的支持

jest，基于jasmine语法，大而全的测试框架，集成了mocha、chai、jsdom、sinon的功能，用户无需关注其内部实现

### 测试覆盖率工具

istanbul，用于统计测试用例的覆盖情况，主要涉及语句、分支、函数、行数等方面

### mock方法

sinon， 一个简单的mock库，拥有三个主要函数，分别是spies, stub, mock

1. spies主要是虚构一个同名函数，不影响原函数功能

2. stub完全取代原函数调用

3. mock替换模块中的多个方法

了解了测试需要的相关环境后，我们能够看出Jest的优势，功能全面，继承了老的测试框架的功能语法，学习成本低，所以选择Jest作为单元测试的框架


## 如何在Vue项目上进行单元测试

如果你是使用vue-cli3.0以上构建的项目，可以在可视化界面选择@vue/cli-plugin-unit-jest插件安装即可

如果是命令行调用，则执行

```
vue add unit-jest
```

或者直接使用npm安装

```
npm i -D @vue/cli-plugin-unit-jest
```

安装该插件后，需要进行配置

    // jest.config.js
    preset: '@vue/cli-plugin-unit-jest'

如果不使用该插件，则需要自己安装Jest用到的相关依赖

```
npm i -D jest vue-jest babel-jest babel-core @babel/core jest-serializer-vue @babel/preset-env
```

其中，jest是测试框架本身；而vue-jest是用来jest如何处理*.vue文件的预处理器；babel-jest、babel-core和@babel/preset-env是用来处理待测试的JS文件的，因为我们希望使用ES Module语法和stage-x的特性；jest-serializer-vue则是一个用来序列化快照的工具

~~这里我在配置babel-jest遇到了很多坑，初步看可能是我项目中使用的是老的babel6，但是jest现在支持babel7的语法导致的兼容问题，暂时没有解决，只能使用@vue/cli-plugin-unit-jest，使用这个插件不需要配置babel也不需要安装babel-jest和vue-jest，因为插件内部已经使用了这两个包~~

**配置babel-jest时遇到的坑，主要是由于其安装的依赖是babel-core6.0的版本，babel-jest插件说明的很清楚，由于我当前项目使用的是babel7，所以需要安装7.0以上版本的babel-core**

具体做法是：

    npx babel-upgrade --write
    npm i
    
上面命令的目的是将项目中使用到的babel从6升级为7，之后重新安装一下包就好了

接着需要配置babel：

    // babel.config.js
    module.exports = {
      presets: [
        '@vue/app',
        // 新增
        '@babel/preset-env',
      ]
    }


上面弄完之后，还需要配置jest，可以在package.json文件里，也可以在根目录下创建一个jest.config.js文件（使用jest --init命令），配置如下

```
module.exports = {
  // 如果没有安装@vue/cli-plugin-unit-jest插件无需该行
  preset: "@vue/cli-plugin-unit-jest",
  
  // 告诉jest处理哪些后缀文件
  moduleFileExtensions: ["js", "vue"],
  
  // 告诉jest用什么插件处理这些文件
  transform: {
    "^.+\\.vue$": "<rootDir>/node_modules/vue-jest",
    "^.+\\.js$": "<rootDir>/node_modules/babel-jest"
  },
  transformIgnorePatterns: ["<rootDir>/node_modules/"],

  // 如果使用webpack配置了别名，可以对测试文件的引用使用相同的别名
  moduleNameMapper: {
    "^@/(.*)$": "<rootDir>/src/$1"
  },

  // jest的启动文件
  setupFiles: ["<rootDir>/tests/unit/setup"],

  // 测试文件的匹配范围，默认项目下所有spec.js和test.js文件
  // testMatch: ["<rootDir>/tests/\*\*/\*.spec.js"],

  // 快照序列化
  snapshotSerializers: ["jest-serializer-vue"],

  // 测试覆盖率统计忽略文件范围
  coveragePathIgnorePatterns: [
  ],
  
  // 覆盖率统计文件范围
  collectCoverageFrom: [
    "src/**/*.{js,vue}",
    "!src/main.js",
    "!**/node_modules/**"
  ],
  
  // 测试报告格式
  // coverageReporters: ["html", "text-summary"],
  
  // 是否开启测试覆盖率报告
  // collectCoverage: true
};
```

接着需要配置script命令，因为使用的是@vue/cli-plugin-unit-jest插件，所以命令为：

```
"test:cov": "vue-cli-service test:unit --coverage --verbose --color",
"test": "vue-cli-service test:unit --verbose --color",
"test:snop": "vue-cli-service test:unit -u --verbose --color",
```

否则为：

```
"test:cov": "jest --coverage --verbose --color",
"test": "jest --verbose --color",
"test:snop": "jest -u --verbose --color",
```

上面都配置完成后，已经可以正常的跑测试文件了，我们可以随便写一个加法函数，接着写其测试文件

```
// demo.js

export function add (a, b) {
  return a + b
}

// demo.spec.js
import { add } from 'demo.js'

describe('test add', () => {
  it('add correctly', () => {
    expect(add(1, 2)).toBe(3)
  })
})
```

跑一下可以看到测试通过了，代表环境配置没有问题，否则需要检查配置

由于我们是测试Vue文件，所以最好引入Vue官方推荐的@vue/test-utils包，其内部封装了很多操作dom、创建渲染dom的API，开箱即用，十分方便

```
npm i -D @vue/test-utils
```

接着介绍一些常见的写法

### mock方法

我们在编写测试用例之前，需要作一些准备工作，jest提供了几个钩子，让我们能够在测试的几个周期绑定函数

- beforeAll：在一个测试套内所有测试桩运行之前执行
- beforeEach：在每个测试桩运行之前执行
- afterAll：在所有测试桩运行之后执行
- afterEach：在每个测试桩运行之后执行，一般用于销毁Vue实例

jest自带的几个mock方法

- jest.fn：普通的mock，接收方法作为参数，实际执行传入的方法代替需要mock的方法
- jest.mock：通常用来mock整个模块，接收模块的路径字符串
- jest.spyOn：mock的方法会被执行，但是不影响实际结果

上面的三个mock方法都能自定义传参和返回值，调用的方法参照[Jest文档](https://jestjs.io/docs/zh-Hans/mock-functions)

### 挂载全局变量

1.如果只是挂载Vue的全局变量，可以使用挂载函数的mocks选项，内部传入的属性都会替代实际的方法，比如自定义的$http属性方法，如果在mocks选项中修改为
```
mocks: {
  $http: () => 1
}
```
最后就会得到1的返回值

2.如果是项目中全局变量，比如uni-app中的uni，微信小程序中的wx等，则需要在jest的启动文件中挂载到全局对象global上，具体操作为

```
// mock/uni.js
class Uni {
  login() {}
}

export default new Uni()

// setup.js
import uni from "./mock/uni";

global.uni = uni
```

这样操作的弊端是如果测试涉及到的全局对象方法很多，就需要每个方法都进行mock，很耗时

### 操作DOM

@vue/test-utils提供了两个挂载方法，分别是shallowMount和mount，前者不会挂载子组件，后者会

```
// home.spec.vue

import abnormal from '@/home/home'
import { shallowMount } from '@vue/test-utils'

describe('test home', () => {
  let wrapper, vm

  it('测试', () => {
    wrapper = shallowMount(home)
    vm = wrapper.vm
  })
})
```

我们挂载组件后，就能够访问到组件中的数据和方法，挂载有几个选项，可以使得我们选择如何挂载组件

```
wrapper = shallowMount(home, {
  data() {
    // 初始化data数据
  },
  propsData: {
    // 初始化props
  },
  localVue, // 创建的Vue的拷贝，防止全局Vue被污染
  slots: {
    default: '<div></div>' // 插槽
  },
  mocks: {
    // 伪造挂载在Vue.prototype上的属性，比如$store、$router
  }
})
```

其他非常用选项参照[挂载选项](https://vue-test-utils.vuejs.org/zh/api/options.html)

接着介绍wrapper几个常用的属性方法

- find/findComponent：前者匹配选择器，后者匹配组件
- findAll：匹配多个元素，使用at取出想要的
- exists：断言该wrapper是否存在
- destory：销毁挂载的该实例
- html：获取DOM节点的html字符串
- attributes：取得该DOM节点的属性对象
- classes：取得该DOM的类名数组
- emitted：返回vm触发的自定义事件对象
- trigger：触发该DOM节点的事件，第二个参数为options对象，其内的属性会添加到事件上

下面的是异步方法

- setProps：修改props
- setData：修改data
- setSelected：设置checkbox或者radio的checked值并更新v-model绑定的数据
- setValue：设置文本或select元素的值并更新v-model绑定的数据

```
it('异步更新', async () => {
  wrapper.setProps({ value: 2 })
  await Vue.nextTick()
  
  expect(vm.value).toBe(2)
})
```

### 异步处理注意事项

1.如果是更新属性这种来自Vue的内部更新，根据Vue的设计原则，会在下一个tick批量更新，避免数据反复变化导致不必要的渲染，也就是说，我们调用上面的几个设置方法时需要，人为的调用Vue.nextTick()

2.如果我们在编码时调用了Vue.nextTick()，也是需要在测试时调用Vue.nextTick()才能看到正确结果的，此外，如果是定时器一类的，则需要更新写法

```
it('异步更新', done => {
  wrapper.setProps({ value: 2 })
 
  setTimeout(() => {
    expect(vm.value).toBe(2)
    done()
  }, 100)
})
```

关于定时器，其实jest提供了mock方法，我们在文件开头或者beforeEach加上jest.useFakeTimers()，这样就mock了常用的几个定时器

```
jest.useFakeTimers();

it('waits 1 second before ending the game', () => {
  const timerGame = require('../timerGame');
  timerGame();

  expect(setTimeout).toHaveBeenCalledTimes(1);
  expect(setTimeout).toHaveBeenLastCalledWith(expect.any(Function), 1000);
});
```

如果想断言回调函数，可以使用jest.runAllTimers()使得所有定时器回调被马上执行

```
it('calls the callback after 1 second', () => {
  const timerGame = require('../timerGame');
  const callback = jest.fn();

  timerGame(callback);

  // 在这个时间点，定时器的回调不应该被执行
  expect(callback).not.toBeCalled();

  // “快进”时间使得所有定时器回调被执行
  jest.runAllTimers();

  // 现在回调函数应该被调用了！
  expect(callback).toBeCalled();
  expect(callback).toHaveBeenCalledTimes(1);
});
```

3.如果是来自外部行为的更新，比如点击按钮触发的数据请求，则需要引入npm仓库中的flush-promises，提升代码可读性

```
import { shallowMount } from '@vue/test-utils'
import flushPromises from 'flush-promises'
import Foo from './Foo'
jest.mock('axios')

it('fetches async when a button is clicked', async () => {
  const wrapper = shallowMount(Foo)
  wrapper.find('button').trigger('click')
  await flushPromises()
  expect(wrapper.vm.value).toBe('value')
})
```

## 快照测试

快照测试我比较陌生，后面专门去了解了一下，使用起来其实十分简单，就和它字面意思一样，我们在过去如果认为某个结果是正确的，就拍下照片，作为参考，如果后面发现和前面的快照不一致，说明发生了改动，那么我们需要判断改动是否符合预期

快照测试主要用于html和配置文件，防止UI和配置被无意修改

用法为

```
expect(wrapper.html()).toMatchSnapshot()
```

没有快照时，会在该测试文件目录下生成一个__snapshots__文件夹，存放快照，第二次运行测试时会进行比对，不一致则报错

如果我们认为后面的改动是符合预期的，可以运行jest -u命令更新快照

## 测试结果分析

我们在跑测试样例时，加上--coverage参数就能得到测试覆盖率的结果，形如表格，统计出测试覆盖的语句、逻辑分支、代码行数等，以此来评估测试样例编写的是否恰当，测试是否周全

通过jest.config.js文件的collectCoverageFrom选项，我们指定从哪些文件或文件夹收集测试报告，通过coverageReporters选项确定测试报告的格式

加上--coverage参数后，不仅能在命令行得到整体测试结果，也会在根目录生成一个coverage文件夹，里面包含每个被测试文件的详细情况
