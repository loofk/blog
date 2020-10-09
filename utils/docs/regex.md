## 前言

正则表达式一直是编程语言中我们又爱又恨的一种语言，使用得当可以大大减少代码，使用不当会引发未知的错误，同时编写阅读的复杂度导致维护困难，今天，我们就初步学习一下正则表达式的基本语法

## 元字符

正则表达式中有一些元字符，它们是正则表达式规定的特殊代码，它们被赋予了固定的含义并且无法从表面看出其含义，只能通过记忆来理解

- .（点），匹配除换行符外的任意字符
- \w，匹配字母或数字或下划线或汉字
- \s，匹配任意空白符
- \d，匹配任意数字
- \b，匹配单词的开始或结束
- ^，匹配字符串的开始
- $，匹配字符串的结束

## 字符转义

如果我们要匹配元字符本身，我们就需要`\`来取消这些元字符的特殊含义，比如我们要匹配\，可以使用`\\`

## 重复

有时候我们需要多次匹配某些规则，正则表达式规定了一些元字符或式子用来制定匹配的次数

- *，重复零次或多次
- +，重复一次或多次
- ？，重复零次或一次
- {n}，重复n次
- {n,}，重复n次或多次
- {n,m}，重复n次到m次

## 字符类

我们可以用`\d`来匹配任意数字，这是因为预设了这个元字符，但如果我们想要匹配所有元音字符，就无法使用现成的元字符了，这时我们可以使用`[]`来包含一组我们需要匹配的字符的集合，括号中的字符是或的关系，只要满足一个就能匹配，比如我们可以使用`[aeiou]`来匹配任意一个元音字符

有了字符类，我们就可以轻松的划定一个范围，更复杂一点的比如`[a-z0-9A-Z]`的含义和`\w`是一样的

## 分支条件

正则表达式中的分支条件指有几种规则，只要满足其中任意一种规则都应当匹配，具体是用`|`把不同的规则分隔开来

```
/(\0\d{2}\)[- ]?\d{8}|0\d{2}[- ]?\d{8}/
```
上面的表达式表示匹配可以用括号括起区号来也可以不用，区号和本地号之间可以使用`-`或空格分隔，也可以不用

## 分组

重复单个字符可以使用限定符，但是如果要重复多个字符就必须使用`()`指定子表达式，然后对这个子表达式使用限定符，当然也可以使用分组匹配匹配不同的分组

```
/(\d{1,3}\.){3}\d{1,3}/
```
上面表示一个简单IP地址的匹配，但我们注意到IP地址的每位不能大于255且允许前导零，所以我们要改进一下，我们使用`2[0-4]{2}|25[0-5]|[01]?\d\d?`来表示每一位，这样完整的正则表达式就是：

```
/(2[0-4]{2}|25[0-5]|[01]?\d\d?\.){3}2[0-4]{2}|25[0-5]|[01]?\d\d?/
```

## 反义

有时候我们需要查找的字符集是除去某个简单字符集合规则之外的任意字符集，这个时候就需要用到反义，常见的反义元字符有

- \W，匹配任意不是字母或数字或下划线或汉字的字符
- \S，匹配任意不是空白符的字符
- \D，匹配任意非数字的字符
- \B，匹配不是单词开始或结束的位置
- [^x]，匹配除了x之外的任意字符
- [^aeiou]，匹配不是元音字符外的任意字符

## 后向引用

上面我们提到可以使用`()`指定子表达式，这个子表达式被称为分组，对于分组，我们可以在表达式或应用程序中继续使用做进一步处理，默认情况下，每个分组会被自动命名，分组0代表整个正则表达式，以左括号为标识，1代表第一个分组，依次类推

更复杂的是，对于分组命名，其实正则表达式会进行两次扫描，第一次扫描所有未命名分组，第二次扫描命名分组，分配数字编号，所以所有的命名分组的组号都要大于未命名分组，此外，当我们使用零宽断言时，可以剥夺对一个分组的组号分配参与权

对于拥有组号的分组，我们使用`\1`这样的形式对其进行引用，比如

```
/(\w){2}\1{3}/
```
表示先匹配2个字母然后匹配3个字母

我们可以自行指定分组的名称，这样的被称为命名分组，格式为`(?<name>exp)`或者`(?'name'exp)`，引用命名分组使用格式为`\k<name>`

常用的分组语法

- (exp)，匹配并捕获exp到自动命名的分组里
- (?<name>exp)，匹配exp并将该分组命名为name
- (?:exp)，匹配exp但不捕获该分组且不给该分组分配组号
- (?=exp)，匹配后面跟的是exp的位置
- (?<=exp)，匹配前面跟的是exp的位置
- (?!exp)，匹配后面跟的不是exp的位置
- (?<!exp)，匹配前面跟的不是exp的位置
- (?#comment)，这种类型的分组不对正则表达式的处理产生影响，仅提供注释用以阅读

## 贪婪与懒惰

当表达式中包含接收重复的限定符时，通常的行为是匹配尽可能多的字符，这被称为贪婪匹配，有时我们更需要懒惰匹配，也就是匹配尽可能少的字符，我们需要做的只是在前面给出的限定符后面加上一个`?`，举个例子，我们使用`a.*b`匹配`aabab`，在懒惰模式下只能匹配到`aab`和`ab`（第四到第五个字符）

这时我们会有一个疑问，为什么第一次不是匹配`ab`呢，这是因为还有一条规则优先级高于贪婪和懒惰模式，那就是最先开始的匹配拥有最高优先权

## 处理选项

我们可以使用几个选项来改变正则表达式的处理方法，比如忽略大小写等

- i，匹配时不区分大小写
- g，匹配时进行全局匹配直到匹配到字符串末尾
- m，多行模式，更改`^`和`$`的含义使之分别在任意一行的行首和行尾进行匹配，其实是匹配`\n`之前的位置和字符串结束的位置
- s，单行模式，修改`.`的含义使之匹配`\n`

## 平衡组

有时我们需要匹配嵌套的层次性结构，比如说我们要匹配出最长的配对括号之间的内容，就不能使用类似`\(.+\)`这样简单的正则表达式，因为在左右括号出现的次数不相等时是无法满足的

我们考虑使用下面的语法构造

- (?'group')，把捕获的内容命名为group并压入栈
- (?'-group')，从栈中弹出最后压入的命名为group的捕获内容，如果本来为空则匹配失败
- (?(group)yes|no)，如果栈中存在命名为group的捕获内容，继续匹配yes部分的表达式，否则继续匹配no部分
- (?!)，零负向先行断言，由于没有后缀表达式，试图匹配总是失败

```
<[^<>]*(((?'Open'<)[^<>]*)+((?'Open'<)[^<>]*)+)*(?(Open)(?!))>
```

