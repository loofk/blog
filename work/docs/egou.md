新零售项目经历了两个阶段，一个是前面兼容多端采用的UNI-APP框架开发，一个是后面专注小程序使用原生开发

## 技术栈

- uni-app基于Vue和微信小程序API
- 微信原生开发

使用uni-app的优势

- 开箱即用，如果已经熟悉Vue的语法上手快，直接使用Vue全家桶所以不再需要进行工程化构建
- 代码维护轻松，通过条件编译就可以兼容多端，不必维护多套代码
- 社区活跃，内部的组件市场拥有较多的组件，不必自己再造轮子，缩短开发周期，适合项目快速迭代

缺点

- 毕竟是一个起步没多久的多端框架，周边生态还不够成熟
- 项目组维护代码积极但迭代版本BUG较多，在项目中经常出现各种报错问题
- 反馈不够及时，BUG不能及时的修复，只能自己查阅源码并修改，这样在更新时就得重复劳作

## 工程构建

在使用原生开发后，就失去了工程的能力，微信官方并没有提供一套工程化的解决方案，只能自己选型，使用合适的工具工程化

我个人是选择的gulp+less+ts的组合，使用gulp工具主要原因就是小程序原生是将整个代码包发布到微信服务器运行的，其代码体积不会很大，所以不需要用到webpack这种打包工具，我们只需要实现一些基础的less->css，ts->js的能力就能进行开发，由于在上传时会压缩混淆代码，所以我们甚至不需要自己压缩代码

选型确定后，我们相比原来，开发效率已经提升很多了，但我想在开发风格上更接近Vue的语法，降低一些心智负担，所以我针对小程序的API进行了预处理

具体做法是

- 定义一组映射map，小程序的API和Vue的API的命名一一映射，比如一些options，在微信中是property，而在Vue中叫做props
- 定义好映射后，我们在编写时就可以按照Vue的格式编写组件，然后把他们映射成微信的options，最后再传入Component中
- 不仅如此，我们还可以在这期间注入一些全局属性和方法，合并到options中，实现类似Vue.mixin的能力

类似的，我们可以利用这个原理为Page方法扩展一个mixins属性，方便我们混入一些公共方法，这样我们就可以提出一些页面间复用的方法

原生开发有一个比较恶心的就是，每新开一个页面或者组件，就要新建一个文件夹，然后新建需要的wxml、wxss等文件，浪费时间，所以我想到编写一个脚本，自动生成所需的文件夹及文件

编写脚本的大致做法是找到新建文件的目录，然后根据传入的参数，创建对应名称的文件，文件源可以自己提前编写好模板存放在项目中，可以针对命令行优化一下，使用npm包实现问答式参数选择，这样就可以方便一些

## 性能优化

- 在编写代码时注重代码复用，针对重复的逻辑抽离成公共方法或组件
- 针对资源（图片、字体）先压缩再使用，精灵图
- 针对接口，使用前端localStorage缓存一些不经常变更的数据，在首屏渲染时先加载本地的缓存数据，然后等接口返回后更新本地缓存
- 针对页面渲染，使用骨架屏技术
- 针对服务端，启用http缓存，加快接口的请求速度
- 针对首屏，使用懒加载技术，对图片和不再第一屏的列表数据都延时渲染，加快首屏加载速度
- 分包，分包预加载，独立分包
- 懒注入，只注入当前页面需要的代码，其他代码后续注入
- 提前首屏数据请求，比如数据预拉取和周期性更新，通过在小程序管理界面配置需要预请求或周期性更新的接口实现
- 针对setData进行优化：不频繁调用，合并调用，传递必要数据，不在后台调用
- 减少大图的使用，可以控制大图或者长列表图片的数量
  
## 难点

### 请求自动登录

由于需求需要在启动小程序时静默登录，所以我们每次都需要先登录再请求数据，而我们的其他接口大部分又必须在登录后才能请求数据，这就需要我们实现一个登录逻辑

在实现过程中遇到了一个异步的问题，就是我们需要保证登录的请求先于数据请求，这样造成在编码时代码比较冗余复杂，逻辑不清晰，所以打算封装一个登录组件，在需要登录的页面插入这个组件

这样在数据请求前就会先去登录，但是这样的实现有一个问题，就是我们这个业务的绝大多数接口都需要登录，所以就要在几乎每个页面都写一遍这个组件，极其恶心，后面我想到封装一个请求类来实现登录流程

具体做法是自己封装一个请求类，继承自动登录和错误处理功能，这样就可以在每个接口请求前完成登录的过程

### sku级联

在做电商项目时会需要选择sku，因为各个sku库存的不同，所以要实现根据已选判断出剩余的sku选项哪些能选，就是要实现sku级联

一种做法是维护一个path数组，然后根据用户的选择，进行全排列，找出剩余有库存的sku，返回，这种做法缺点就是耗时长

另一种做法是使用无向图邻接矩阵，把sku抽象成一个个的点，把有库存的sku组合用边连接起来，这样我们就可以通过判断sku的数量以及选中sku的邻接点个数来判断是否存在通路，即如果是有库存的sku组合，在遍历时就会记录一次邻接次数，如果一个sku与传入的sku数组都邻接，则说明他们的组合存在通路，代表他们是有库存的，这样就可以快速的找出剩余有库存的sku

这种做法有一个bug，举个例子，如果是三个sku的组合，我在初始化时，每两个sku都取一次，但三个的组合是不存在的，这样就会造成最后即使没有这个组合的库存，因为它们也联通，导致计算出错

后面我修复了这个问题，做法是在最后一次选择sku之前改用上面的第一种方法，由于这时，选择已经接近完整，所以排列的可能大大减少，耗时几乎可以忽略不计
