---
title: ES6 箭头函数
categories: 前端
abbrlink: '4e410719'
date: 2019-08-25 21:42:02
copyright_author: liuliu
copyright_url: 'https://segmentfault.com/a/1190000020178946'
---

ES6 中添加了函数新的定义语法——`箭头函数`，当有大于一个形参的时候，必须使用`()`代表部分参数，函数体大于一行时，必须使用`{}`将函数体括起来，并使用 `return` 返回。

### 箭头函数不会创建自己的 this

箭头函数会在自己的作用域链上的上一层寻找 this。所以箭头函数会在定义时找到自己外层的 this，并继承这个 this 的值。在后面的任何操作中，this 的值都不会改变。

箭头函数的实现：

```javascript
var a =  1;
function func() {
    setTimeout(() => {
        console.log(this.a);
    }, 1000);
}
func.call({a: 2});
// 2
```

`setTimeout` 使用的是箭头函数，所以this在定义的时候就确定了，继为外层 func 的 `this` 的值。在函数执行的时候，通过 `call` 改变了 func 的 this 指向 `{a:2}`, 所以箭头函数继承 func 在执行环境中的指向，所以打印输出2。

普通函数的实现：

```javascript
var a =  1;
let func = function () {
    setTimeout(function () 
    {
        console.log(this.a);
    }, 1000);
}
func.call({a: 2});
//1
```
`setTimeout` 使用的是普通的函数，所以其 `this` 指向为函数运行时的所在的作用域。在1秒之后函数是在全局作用域中执行的，此时 this 指向 window 对象，所以输出为1。

### 不能定义对象方法

```javascript 
let obj = {
    name: 'lisi',
    a: function() {
        console.log(this.name);
    },
    b: () =>{
        console.log(this.name);
        this.a();
    }
}

let name = "zhangsan";
obj.a();
obj.b();

//lisi
//zhangsan
//Uncaught TypeError: this.a is not a function
```

JS中对象方法的定义方式是在对象上定义一个指向函数的属性，当方法被调用的时候，方法内的 `this` 就会指向方法所属的对象。所以方法 a 中的 this 指向对象obj。但是在 obj 对象的`{}`无法形成一个单独的作用域，所以 obj 是在 window 作用域中，方法 b 为箭头函数，指向上层 obj 的 `this === window`，所以 a 方法输出 obj 的 name 属性，b 方法输出全局 name 的值。且由于全局作用域上没有 a 方法，所以 this.a() 执行报错。所以用箭头函数定义对象方法时，不会有预想的输出。当要定义对象方法时，请使用函数表达式或者方法简写（es6）。

### 箭头函数不能当作构造函数 


```javascript
function Person (name, age) {
     this.age = age;
     this.name = name;
}
let person = new Person('lisi', 23);
console.log(person.name, person.age);
//lisi 23
```
js 在 new 生成一个新的实例对象的时候，首先在内部生成一个新的空对象，然后将this指向该对象，接着执行构造函数的指令，最后将该对象作为实例返回。

```javascript
let Person = (name, age) => {
    this.name = name;
    this.age = age;
}
let person = new Person('lisi', 23);
console.log(person.name, person.age);
//Uncaught TypeError: Person is not a constructor
```

但是箭头函数并没有自己的 this, 其 this 是继承了外层执行环境的 this，且不能改变，因此不能作为构造函数，此时，js 引擎会在报错。Person 不是一个构造函数。

### this 的指向改变

箭头函数 this 的指向不会因为 call()、bind()、apply() 而改变：

```javascript
var a = 'haha'
let func = () =>{
    console.log(this.a)
}
func();
func.bind({a: 1})()
func.apply({a: 2});
func.call({a: 3});
//haha
//haha
//haha
//haha
```

在 js 中，可以使用 bind()、apply()、call() 来动态绑定函数的作用域，但是在箭头函数中，this由于已经固化在上层作用域链上，所以这三种方法不能改变 this 的指向，且不报错。

### 箭头函数不能使用 arguments、super、new.target 

```javascript
function foo() {
  setTimeout(() => {
    console.log('args:', arguments);
  }, 100);
}

foo(2, 4, 6, 8)
// args: [2, 4, 6, 8]
```

由于定时器的回调函数是一个箭头函数，所以其this指向上一层的foo函数的环境中的值。

在 ES6 中可以使用 rest 参数代替 arguments 来访问函数的参数列表：

```javascript
let func = (...args) => {
    console.log(args);
}

func(1, 2, 3, 4, 5);
//[1,2,3,4,5]
```

### 没有原型

由于箭头函数没有自己的 this，,所以箭头函数没有自己的原型。

```javascript
let sayHi = () => {
    console.log('Hello World !')
};
console.log(sayHi.prototype); // undefined
```

### 能作为事件的回调 

在 js 中，事件的回调函数中，this 会动态的指向监听的对象，但是由于监听是一个全局函数，所以箭头函数的回调中 this 指向 window。

```javascript
var button = document.getElementById('press');
button.addEventListener('click', () => {
  this.innerHtml = 'hello'
});
```

在全局上下文下定义的箭头函数执行时 this 会指向 window，当单击事件发生时，浏览器会尝试用 button 作为上下文来执行事件回调函数，但是箭头函数预定义的上下文是不能被修改的，这样 this.innerHTML 就等价于 window.innerHTML，而后者是没有任何意义的。
