1.JavaScript的类型可分为可变类型和不可变类型，不可变类型如数字、布尔值、null和undefined；可变类型如数组、对象，他们可以修改属性值或者数组元素的值。要注意的是字符串虽然可以看成是字符组成的数组，但它是不可变的，JavaScript并未提供修改已知字符串的方法

2.JavaScript里数字的类型只有Number类型，也就是64位的浮点数，其对整数的表示等同于双精度浮点数对整数的表示
  - 双精度浮点数由1位符号、11位指数、52位尾数构成，计算机储存数字是按照科学计数法来存的，我们知道所有的十位数都可以表示为`1.xxx * 10 ^ n`，但是计算机并不认识十进制，我们只能把数字转化成二进制的科学计数法`1.xxx * 2 ^ n`
  - 十进制浮点数转为二进制时整数部分采用`除2取余，逆序排列`，小数部分采用`乘2取整，顺序排列`，举个例子，8.25转成二进制为1000.01，用科学计数法表示为`1.0001 * 2 ^ 3`，由于指数位是采用移位储存的，指数位都在原数值的基础上加127进行储存，所以指数位的储存值为130，尾数位由于首位都为1，可省略不存，即`0 10000010 0001...0`
  - 现在我们根据这个来计算最大整数，我们推测出当二进制科学计数法为`1.111...111 * 2 ^ 52`时达到最大，因为再加1则尾数总位数变为53位，超出52位尾数的表示范围了，多出的部分被截断，意思就是我们无法区分`2 ^ 53`和`2 ^ 53 + 1`，也就是不安全了，所以安全整数的范围为`-2 ^ 53 - 2 ^ 53`（不包含边界）