## 前言

最近我在了解babel原理的时候，看了一段@babel/core中transform函数的源码，发现其源码很多的函数调用都使用了如下的形式：

    (0, func)()
  
当时觉得很难理解，于是去谷歌了一下

## 结果

首先我们是知道立即执行函数的用法的

    (function(a) { return a })(1)
   
这行代码的输出结果为1

接着我了解到逗号操作符的运算逻辑为从左到右依次执行

    const b = 2
    let a = (1+2, 5, b * 2)
    
这里a的值为4，说明在逗号操作符里，从左到右执行完成表达式后，取的值为最后一个表达式的值

这里我们明白(0, func)()其实0并没有用，我们可以拆解这个式子成：

    0
    var b = func
    b()
    
这里根据JS的语法，我们知道如果函数是某个对象的方法，这样的赋值其实等价于改变this的指向为全局对象，严格模式下，指向undefined

    const obj = {
      foo() {
        console.log('hello, world')
        console.log(this)
      }
    }
    
    (obj.foo)()
    (0, foo)()
    
 上面两个调用输出的结果不一样，前者会输出obj对象本身，而后者在浏览器中会输出windows对象
 
 ## 思考
 
 既然我们搞明白了这种调用方式的目的是改变函数调用中的this指向为全局对象，那么我们为什么不使用apply/call之类的原型方法呢？
 
 个人理解，这种写法主要是考虑到在封装库或者插件时，会修改prototype，这导致我们无法访问到原始的原型方法，
 而为了保证能够在调用时修改this指向，节约代码的就是使用这种方式，如果后续有新的理解再补充
