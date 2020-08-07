---
title: "DeepClone深克隆"
date: 2019-07-02T21:30:20+08:00
tags: ["JavaScript", "前端"]
categories: ["Code"]
featured_image:
draft: false
---


# DeepClone深克隆

之前去参加字节跳动面试的时候遇到了很尴尬的情况，连续两个面试官让我手写一个深克隆出来，尽管我明白需要深克隆的原因是什么，然而之前准备的时候虽然看到了有关深克隆的相关概念，因为看到“需要付出一定的性能代价，因此平常使用中需要尽量避免”这句话，于是便没有深究实现方法，结果当然也是没能够写出来。

于是打算利用这篇blog彻底理解这方面的概念和实现方法，也加深对JavaScript这门语言的认识

## 为什么会有深克隆的问题——谈谈弱类型

有深克隆就会有浅克隆，出现这两者的原因还是由于JavaScript的类型机制

说到类型机制，JavaScript作为一门弱类型的语言，在变量类型上帮助程序员做了太多的工作，由于我之前是以写C/C++为主，在刚接触JavaScript、Python和PHP的时候，惊奇于竟然有语言不需要再申明变量的时候使用类型定义符，感叹于总算不需要为某种变量不知该定义哪种类型头疼了。

于此同时，对于Python，PHP这种新变量直接赋值便可以使用的来说，我更喜欢JS的原因是有用于变量声明的`var`，`const`，`let`，因为这更贴近我所熟悉的C++语法，而PHP使用变量都需要打一个`$`符号也让我十分不习惯

当然有利就有弊，弱类型语言带来的最大的问题就是变量类型的不明确以及对于编辑器无法做出类型推导等。弱类型语言使得编译器（或解释器）不太去关注变量的类型，同时也会帮程序员做一些看不见的类型转换，虽然一定程度上让人可以更加专注于程序逻辑上，但往往会埋下一些潜在的问题

用一个最简单的例子

```javascript
//javascript
var a = 1
a = "123"
var b = a + 1	// '1231'
```

```python
# python
a = 1
a = '123'
b = a + 1 # type error
```

JavaScript是直接可以**字符串+int值**的，然而python中会报错，但是两者对于一个int型的变量赋值一个字符串都是合法的。这在c++里是不可想象的。

这就导致了在函数传参的时候又可能传进去参数的类型并不是所期待的类型，而这种错往往很难发现。

好在对于JavaScript，微软发明了一种TypeScript的语言作为JavaScript的超集，很好的解决了JavaScript最大的痛点，类型问题，而且在编辑器里有着很好的类型推导，这简直让人更加喜爱这门语言

话说回到深克隆问题

用过C++的人都知道，C++最大的一个优势之一就是凭借着指针类型变量可以方便的直接对内存进行操作。而对于JavaScript其实也是有指针变量，虽然并没有单独划分出一个类型而且功能没C++那么强大，但都是表示指向了内存的一块区域。

当时在学习C++类的拷贝构造函数时，老师就特意提醒对于类中的指针变量不可直接赋值，需要申请新的内存空间然后使用memcpy等函数进行内存复制。这是因为直接进行指针的赋值会导致多个指针指向同一块内存，引起访问冲突。而这，正是导致JavaScript中出现浅拷贝深拷贝问题的根源。

```javascript
var a = [1, 2, 3]
var b = a
```

在这里，a和b可以看作两个指针，同时指向了一个数组，对其中一个做出修改，另一个也会同时变化。

如果要实现一个浅克隆可以这样做

```javascript
var a = [1, 2, 3]
var b = []
for(let i = 0; i < a.length; i++) {
    b.push(a[i])
}
// 或者借助Object的assign方法
b = Object.assign([], a)
// ES5写法
b = a.concat()
// 使用ES6的扩展运算符
b = [...a]
```

⚠之所以称之为浅克隆是因为对于有多层嵌套的对象，里面可能有不同的数据类型，使用浅克隆并不能赋值深层对象的内存空间，仍然会出现多个变量指向同一个内存空间的情况，这种情况下，我们就需要深克隆

在js中有一个小技巧可以实现一个有限的深克隆

```javascript
var a = {a: 123, b: {c: '356', d: [7, 8, 9]}}
var b = JSON.parse(JSON.stringify(b))
```

利用JSON转为字符串在进行json解析，可以实现一个方便的深克隆，但是要求对象中只能是一些数值类型变量，对于对象中的函数，以及嵌套引用却无能为力

接下来从JavaScript的数据类型开始讨论深克隆的实现

## JavaScript数据类型

JavaScript总共有七种数据类型

>  六种原始数据类型 Undefined, Null, Boolean, Number, String, Symbol
> 一种引用类型 Object

对于原始数据类型，是直接可以通过等号`=`进行复制的，相当于C++中的**普通类型变量**

而对于Object类型变量，JavaScript是通过变量引用一块内存空间实现的，相当于C++的**指针类型变量**

因此，我们需要能够区分变量类型从而做出不同的处理

使用`typeof`运算符只能区分以上七种以及’function‘

对于Object中类型的区分，需要借助Object原型链上的toString函数，可以将数据类型表示为类似'[object Array]'的字符串，可以很好的区分基本数据类型以及各种object

特殊的，对于DOM元素需要使用`instanceof`来判断，若直接用toString判断，不同标签会返回不同的字符串

这样，实现一个准确判断变量类型的函数

```javascript
function type(obj) {
    var toString = Object.prototype.toString;
    var map = {
        '[object Boolean]'  : 'boolean', 
        '[object Number]'   : 'number', 
        '[object String]'   : 'string', 
        '[object Function]' : 'function', 
        '[object Array]'    : 'array', 
        '[object Date]'     : 'date', 
        '[object RegExp]'   : 'regExp', 
        '[object Undefined]': 'undefined',
        '[object Null]'     : 'null', 
        '[object Object]'   : 'object'
    };
    if(obj instanceof Element) {
        return 'element';
    }
    return map[toString.call(obj)];
}

```

## 实现DeepClone

简单版——未考虑循环引用以及dom元素

```javascript
function deepClone(data) {
    var t = type(data), o;
    
    if(t === 'array') {
        o = [];
    }else if( t === 'object') {
        o = {};
    }else {  // 直接赋值的变量
        return data;
    }
    
    if(t === 'array') {
        for (let i = 0; i < data.length; i++) {
            o.push(deepClone(data[i]));
        }
        return o;
    }else if( t === 'object') {
        for(let i in data) {
            o[i] = deepClone(data[i]);
        }
        return o;
    }
}
```

对于function，直接引用复制即可，因为函数中的this是在运行是才决定指向，对于新的对象，调用的函数即使是原对象的函数引用，this也能正确指向

考虑循环引用的版本

```javascript
/**
 * 区分Object数据类型
 * @param {Object} obj 
 * @returns {String} typename
 */
function objType(obj) {
    if (typeof obj !== 'object')
        return 'notObject'

    var toString = Object.prototype.toString
    var map = {
        '[object Array]': 'array',
        '[object RegExp]': 'regExp',
        '[object Date]': 'date',
        '[object Object]': 'object',
    }

    return map[toString.call(obj)]
}

/**
 * 获取正则对象的flag
 * @param {RegExp} re 正则对象
 */
function getRegFlag(re) {
    var flag = ''
    if (re.global)
        flag += 'g'
    if (re.ignoreCase)
        flag += 'i'
    if (re.multiline)
        flag += 'm'

    return flag
}

/**
 * DeepClone
 * @param {Object} obj 深克隆对象
 * @returns {Object} 克隆后的新对象
 */
function deepClone(obj) {
    // 维护两个数组存储已经克隆过的对象处理循环引用
    var sourceList = []
    var targetList = []

    function _clone(source) {
        // null 会被typeof认为是object 一个一直未被解决的bug
        if (source === null)
            return null

        if (typeof source !== 'object') // 包括function也直接返回
            return source

        let type = objType(source)
        let proto, target

        if (type === 'array') {
            target = []
        }
        else if(type === 'RegExp') {
            target = new RegExp(source.source, getRegFlag(source))
        }
        else if(type === 'date') {
            target = new Data(source.getTime())
        }
        else
        {
            // 复制原型链
            proto = Object.getPrototypeOf(source)
            target = Object.create(proto)
        }

        // 如果已经克隆过
        const index = sourceList.indexOf(source)
        if(index != -1) { 
            return sourceList[index]
        }

        sourceList.push(source)
        targetList.push(target)

        // 递归克隆
        for (let item in source) {
            target[item] = _clone(source[item])
        }
        
        return target
    }


    return _clone(obj)
}
```

