---
title: 三道题增强JS基础
comments: true
date: 2019-02-15 16:21:35
tags:
    - js
    - 基础
---

> 阅读本文需15分钟，可以学习：

1. 强化js基础
2. 检验自己对闭包, this指向的理解  

**本文中题目都在非严格模式下执行**

## 第一题

### 原始题目

- 题干
```javascript
for (var i = 0; i < 5; i++) {
    setTimeout(function() {
        console.log(i);
    }, 1000);
}
 
console.log(i);

// 选项
A. 0,1,2,3,4,5
B. 5,0,1,2,3,4
C. 5,5,5,5,5,5
```
- 答案：C
- 解析：setTimeout会将传入的方法移至下一js队列执行，当前队列执行完for循环已结束，增长到5，所以最终输出全部是5
 
### 追问1

- 如果我们约定，**用箭头表示其前后的两次输出之间有 1 秒的时间间隔，而逗号表示其前后的两次输出之间的时间间隔可以忽略**，代码实际运行的结果该如何描述？
```javascript
// 选项
A. 5 -> 5 -> 5 -> 5 -> 5	即每个 5 之间都有 1 秒的时间间隔
B. 5 -> 5, 5, 5, 5, 5		即第 1 个 5 直接输出，1 秒之后，输出 5 个 5
```
- 答案：B
- 解析：setTimeout会将传入的方法移至下一js队列，并等待相应时间后触发（本题中为1s）。所以当前队列最后一行console先输出一个5，进入下一队列，等待1s后，4个定时器几乎同时输出结果，全部为5

### 追问2

- 如果期望代码的输出变成：5 -> 0, 1, 2, 3, 4，该怎么改造代码？
- 方法1
```javascript
// 知识点：闭包
for (var i = 0; i < 5; i++) {
    (function(j) {
    setTimeout(function() {
        console.log(j);
        }, 1000);
    })(i);
}
 
console.log(i);
```
- 方法2
```javascript
// 知识点：函数参数按值传递
var output = function (i) {
    setTimeout(function() {
        console.log(i);
    }, 1000);
};
 
for (var i = 0; i < 5; i++) {
    output(i);
}

console.log(i);
```
- 方法3
```javascript
// 知识点：setTimeout api
for (var i = 0; i < 5; i++) {
    setTimeout(function(j) {
        console.log(j);
    }, 1000, i);
}
console.log(i);
```
- 方法4（如果你的答案是这个，只能算对一半）
```javascript
for (let i = 0; i < 5; i++) {
    setTimeout(function() {
        console.log(i);
    }, 1000);
}
// 运行至此会报错，因为 let i 只在自己的块级作用域内生效。
console.log(i);
```

## 第二题

### 原始题目

- 题干
```javascript
var name = 'global';

var obj = {
    name: 'local',
    foo: function(){
        this.name = 'foo';
        return function(){
            return this.name;
        }
    }       
}

console.log(obj.foo().call(this))

// 选项
A. global  
B. local
C. foo
D. undefined
```
- 答案：A
- 解析：call中的this指向obj.foo()的返回值，是一个匿名函数，该函数中并没有name这个变量，沿着作用域链向上解析到达全局环境，所以返回全局变量中的name，即'global'。看一个更容易理解的等价变形：
```javascript
obj.foo().call(this) 
=>
var xxx = obj.foo();
xxx.call(this);
```

### 追问1

- 题干
```javascript
var name = 'global';

var obj = {
    name: 'local',
    foo: function(){
        this.name = 'foo';
        return function(){
            return this.name;
        }.bind(this);
    }
}

console.log(obj.foo().call(this))

// 选项
A. global  
B. local
C. foo
D. undefined
```
- 答案：C
- 解析：由于return的function中用了bind，所以相当于固定了this，外边再call什么进来，也只是碍眼法。如果还是不理解，看一个更容易理解的等价变形：
```javascript
var name = 'global';

var obj = {
    name : 'obj',
    foo : function(){
        var that = this;
        this.name = 'foo';
        return function(){
            return that.name;
        } 
    }
}

console.log(obj.foo().call(this))
```

### 追问2

- 题干
```javascript
var name = 'global';

var obj = {
    name: 'local',
    foo: function(){
        this.name = 'foo';
    }.bind(window)
}

var bar = new obj.foo();

console.log(bar.name, window.name);

// 选项
A. global global  
B. foo global
C. local global
D. local foo
```
- 答案：B
- 解析：
    - 先复习下 new 调用：绑定到新创建的对象，注意：显示return函数或对象，返回值不是新创建的对象，而是显式返回的函数或对象。
    - new 对 this 影响的优先级高于bind，obj.foo虽然通过bind绑定了this指向为window，但在使用new调用时，this 仍然指向以obj.foo为构造函数的实例。

## 第三题

### 原始题目

- 题干
```javascript
function fun(n,o) {
    console.log(o)
    return {
        fun: function(m){
            return fun(m,n);
        }
    };
}

var a = fun(0);  a.fun(1);  a.fun(2);  a.fun(3);	// undefined,?,?,?
var b = fun(0).fun(1).fun(2).fun(3);			// undefined,?,?,?
var c = fun(0).fun(1);  c.fun(2);  c.fun(3);	        // undefined,?,?,?

// 问:三行a,b,c的输出分别是什么？
```
- 答案：
```javascript
第一行: undefined,0,0,0
第二行: undefined,0,1,2
第三行: undefined,0,1,1
```
- 解析：
    - 重点在于理清题干中三个fun函数的关系是什么？
        - ① `fun (n, o)`: 第一个fun函数，属于标准具名函数声明，是新创建的函数，他的返回值是一个对象字面量表达式，属于一个新的object。
        - ② `{}.fun`: 这个新的object内部包含一个也叫fun的属性，属于匿名函数表达式，即fun这个属性中存放的是一个新创建匿名函数表达式。第一个fun函数与第二个fun函数不相同，均为新创建的函数。
        - ③ `fun (m, n)`: 匿名函数中引用的fun，在函数内部作用域找不到fun的声明，会向上到外部环境（全局）找到fun的声明。故此第三个fun函数引用的是第一个fun函数
    - 理清了三个fun函数就好理解了（下面用fun①, fun②, fun③分别表示上述三个fun函数，以防混淆）：
        - **第一行：**这里的a是`fun(0)`返回的一个对象，`a.fun`通过闭包引用着`fun(0)`的第一个参数0，作为内部fun③的形参n，所以`a.fun`这个匿名函数中的形参n无论被调用几次都是0。上文讲过，fun③引用的是fun①，fun③的形参n对应fun①的形参`o`，所以三次输出的`o`都为0。
        - **第二行：**这行中`fun(0).fun(1)`的输出与第一行是一样是0，相当于一个等价变形。不同之处在于接下来`.fun(2).fun(3)`使用了 `.` 运算符进行尾调用，每次尾调用都形成新的闭包并被下次调用引用，形参`o`随之更新（更新过程为: `m -> n -> o` ），所以每次调用fun输出的`o`即为上次调用传入的`m`，结果为0，1，2。
        - **第三行：**理解了前两行，第三行为什么会这样输出相信你已经明白了，就是一，二行的结合。