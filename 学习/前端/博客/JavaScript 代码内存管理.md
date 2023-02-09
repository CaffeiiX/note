>博客链接 https://mp.weixin.qq.com/s/sJX0IwkoX_DEVJzCpUVuQg 

Js引擎V8内存的管理方式
![[Pasted image 20221223000258.jpg]]
V8内存大体可以分为 `栈、堆、常量池`三大区域
**栈**：存储连续的区域，其中undefined是栈空间中表示未定义含义的一块的固定的内存区域
``` javascript
var a;
var b = "hello";
```
1. a,b 在栈空间中定义
2. 变量提升过程中，a,b指针均指向undefined区域
3. 执行代码中，遇到赋值，将b指向的内存地址修改未常量池中的物理地址
**常量池**：存储常量的区域，均为基本类型
1. 整个内存中是唯一的
2. 常量池区域是唯一的
3. 常量是唯一的
**堆**：存储的均为对象
```javascript
typeof null;// object 在堆内存空间中具有固定内存地址且唯一存在的一个内置对象
```
![[Pasted image 20221223001244.jpg]]
```javascript
name = '政采云前端团队'  
var a = {  
    name: '政采云前端团队'  
}  
console.log(name === a.name); // true
```
上面这小段代码，执行过程中会在栈中创建 `a` 和 `name` 两个变量。针对于给 `a` 赋值的这个对象，v8 会在堆区中分配一块内存区域。并且区域内部依然会有内部的栈区和堆区，这就是精妙的**分型思想**

```javascript
function Animal(name) {  
    this.name = name;  
}  
  
Animal.prototype.eat = function () {  
    console.log('Animal eat');  
};  
  
function Dog(name) {  
    Animal.apply(this, arguments);  
}  
  
var animal = new Animal();  
  
Dog.prototype = animal;  
Dog.prototype.constructor = Dog;  
var dog = new Dog();  
dog.eat();  
  
console.log(Animal.prototype === animal.__proto__); // true
```
![[Pasted image 20221223001809.jpg]]