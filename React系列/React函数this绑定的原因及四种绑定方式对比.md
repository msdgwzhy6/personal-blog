# React函数this绑定的原因及四种绑定方式对比

react组件在绑定函数时，通常有三种方法，此文将对三种方法性能和写法展开写...

## 为什么要绑定this
首先，第一个很重要的问题就是，绑定this是什么原因？

js里面的this绑定是代码执行的时候进行绑定的，而不是编写的时候，所以this的指向取决于函数调用时的各种条件。

```javascript
import React, { Component } from 'react';

class BindEvent extends Component {
    constructor(props) {
        super(props)
        this.state = {
            name: 'jacky'
        }
    }
    handleClick() {
        console.log(this.state.name)
    }
    render() {
        return (
            <div>
                <button onClick={this.handleClick}>Click</button>
            </div>
        );
    }
}

export default BindEvent;
```
来看上面这个例子，运行点击按钮，发现程序报错，即this指向不是所在类。由于类的方法默认不会绑定this，因此在调用的时候如果没有绑定，this的值将会是undefined。故我们需要手动绑定this
```
 Uncaught TypeError: Cannot read property 'state' of undefined
```
需要我们自己绑定this，其实这不是react的锅，本质原因是 JavaScript 的 this 机制问题

## this机制
来看下面一个例子：
```javascript
const obj = {
    name: 'obj',
    getName: function() {
        console.log(this, this.name);
    }
};
obj.getName(); // {name: "obj", getName: ƒ} "obj"
```
结果理所应当，使用`.`操作符调用函数obj.getName()，this指向的是obj对象，即obj对象是函数getName的调用者，但如果改成这样：
```javascript
const middleObj = obj.getName;
middleObj(); // Window {...} ""
```
将getName函数引用赋值给middleObj变量，并使用这个新的函数引用去调用该函数时，打印出this为Window对象，关于this的指向，可简单粗暴理解为**this 永远指向最后调用它的那个对象**。

如果没有显式调用一个函数，JS 的解释器就会把全局对象当作调用者，在浏览器则是Window对象为调用者。

如果使用严格模式，那么没有显式的使用调用者的情况下，this指向undefined 。

理解完上面，我们用类来作为参照：

下面定义了一个类，类里的display访问this.name属性
```javascript
class Foo {
    constructor(name) {
        this.name = name
    }
    display() {
        console.log(this); // 运行打印： Foo {name: "jacky"}
        console.log(this.name);
    }
}

let foo = new Foo('jacky');
foo.display(); // jacky

let display = foo.display; 
display();  
// undefined
// Uncaught TypeError: Cannot read property 'name' of undefined
```
当我们将一个函数引用赋值给某个其他变量，并使用这个新的函数引用去调用该函数时，在 display() 方法this为`undefined`，故找不到name值报错。

在这里又有一个问题了，display这样直接调用，在非严格模式下调用者不是应该为Window对象吗？在这里，要知道一个知识点：

**ES6 的 class 语法，所有在 class 中声明的方法都会自动地使用严格模式**

## React的JSX

说了这么多，这跟React那个有什么联系呢，想搞清楚为什么绑定this这个问题前，就想先弄明白JSX到底是一个什么东西。
> 本质上来讲，JSX 只是为 React.createElement(component, props, ...children) 方法提供的语法糖。

```javascript
<button className="btn" onClick={this.handleClick}>
    Click Me
</button>
```
经babel编译为：
```javascript
React.createElement(
    "button", 
    { className: "btn", onClick: this.handleClick }, 
    "Click Me"
);
```
我们把render方法里的JSX手动写成编译后这种形式：
```javascript
class Example extends React.Component {
    constructor(props){
        super(props);
    }
    
    handleClick(e) {
        console.log(e);
    }
    
    render(){
        return React.createElement(
            "button", 
            { className: "btn", onClick: this.handleClick }, 
            "Click Me"
        );
    }
}
```

React.createElement的第二个参数，传入的是一个对象，而这个对象里面有属性的值是取this 对象里面的属性 ，当这个对象放入React.createElement执行后，去取这个this.handleClick属性的时候，this已经不是我们在书写的时候认为的绑定在Example组件上了。this.handleClick这里的this会默认绑定，但是又是在ES6的class中，所以this绑定了undefined。

 在JSX语法中: `onClick={ this.handleClick }`中onClick这个属性就是相当于上面的"中间变量"。
 
 即是将`this.handleClick`函数赋值给onClick这个中间变量，后面不仅要进行JSX语法转化,将JSX组件转换成**JS对象**,还要再将Javascript对象转换成真实DOM。把onClick作为中间变量,指向一个函数的时候,后面的一系列处理中，使用onClick这个中间变量所指向的函数，里面的this自然就丢失掉了，不是再指向组件实例了。
 
 所以，当在组件定义方法想访问this，才需要手动绑定。
 
 ## 绑定this的方法
 
 ### render方法中绑定this
 
 ```javascript
 class BindEvent extends Component {
    constructor(props) {
        super(props)
        this.state = {
            name: 'jacky'
        }
    }
    handleClick () {
        console.log(this.state.name)
    }
    render() {
        return (
            <div>
                <button onClick={this.handleClick.bind(this)}>点击</button>
            </div>
        );
    }
}
 ```
这种方法即是在事件函数后使用`.bind(this)`将this绑定到当前组件中。因为 bind 函数会返回一个新的函数，所以每次父组件刷新时，都会重新生成一个函数，即使父组件传递给子组件其他的 props 值不变，子组件每次都会刷新，这将会影响性能，故不推荐使用。

 ### render方法中使用箭头函数
 
 ```javascript
 class BindEvent extends Component {
    constructor(props) {
        super(props)
        this.state = {
            name: 'jacky'
        }
    }
    handleClick () {
        console.log(this.state.name)
    }
    render() {
        return (
            <div>
                <button onClick={() => this.handleClick()}>点击</button>
            </div>
        );
    }
}
 ```
首选要明确一点，**箭头函数中，this 永远绑定了定义箭头函数所在的那个对象**

这种方法写法比较简洁， 最大好处就是传参很灵活，父组件刷新的时候，即使两个箭头函数的函数体是一样的，都会生成一个新的箭头函数。这种方式重新创建函数性能的损耗小于第1种。

 ### 构造函数绑定this
 
 ```javascript
class BindEvent extends Component {
    constructor(props) {
        super(props)
        this.state = {
            name: 'jacky'
        }
        this.handleClick = this.handleClick.bind(this)
    }
    handleClick () {
        console.log(this.state.name)
    }
    render() {
        return (
            <div>
                <button onClick={this.handleClick}>点击</button> 
            </div>
        );
    }
}
 ```
 为了避免在render中绑定this引发可能的性能问题，可以在constructor中预先进行绑定。好处是仅需要绑定一次，避免每次渲染时都要重新绑定，也是常推荐的写法，就是写起来繁琐一点，要单独绑定。
 
  ### 在定义阶段使用箭头函数绑定
  
  ```javascript
  class BindEvent extends Component {
    constructor(props) {
        super(props)
        this.state = {
            name: 'jacky'
        }
    }
    handleClick = () => {
        console.log(this.state.name)
    }
    render() {
        return (
            <div>
                <button onClick={this.handleClick}>点击</button> 
            </div>
        );
    }
}
  ```
  
这种写法是ES7的写法，ES6并不支持，不过我们可以配置你的开发环境支持ES7。

这种方法避免了第1种和第2种的可能潜在的性能问题，也比第3种方法简洁，也是常推荐的写法。

至于哪一种是最好的写法，目前还真的无法定论，有待以后开发中得出最佳实践...

<br>

- ps： [个人技术博文Github仓库](https://github.com/Jacky-Summer/personal-blog)，欢迎star