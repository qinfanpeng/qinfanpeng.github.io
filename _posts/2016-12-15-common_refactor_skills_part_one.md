---
layout: post
title:  "Common Reactor Skills Part One"
date:   2016-12-15 12:55:19 +0800
categories: jekyll update
---

[slide]
# Common Reactor Skills(part one)
# 常见重构手法（一）
## qinfanpeng

[slide]
## Agenda
* Introduce Explaining Variable
* Replace Temp with Query
* Inline Temp
* Extract Method
* Inline Method
* Substitue Algorithm
* Split Temoary Variable
* Remove Assignments to Parameters

[slide]
## Agenda
* **Introduce Explaining Variable**
* **Replace Temp with Query**
* Inline Temp
* **Extract Method**
* Inline Method
* **Substitue Algorithm**
* Split Temoary Variable
* Remove Assignments to Parameters


[slide]

```javascript
const calculatePrice = ({ itemPrice, quantity }) => {
  // price is base price - quantity discount + shipping
  return itemPrice * quantity -
      Math.max(0, quantity - 500) * itemPrice * 0.05 +
      Math.min(100, itemPrice * quantity * 0.1)
}
```
### 重构信号？

* 注释 {:&.fadeIn}
* 表达式逻辑杂糅于细节中，复杂难懂
* 可读性不高
* 重复

[slide]
## Introduce Explaining Variable(引入解释变量)
```javascript
const calculatePrice = ({ itemPrice, quantity }) => {
  // price is base price - quantity discount + shipping
  return itemPrice * quantity -
      Math.max(0, quantity - 500) * itemPrice * 0.05 +
      Math.min(100, itemPrice * quantity * 0.1)
}
```

```javascript
const calculatePrice = ({ itemPrice, quantity }) => {
    const basePrice = itemPrice * quantity
    const quantityDiscount = Math.max(0, quantity - 500) * itemPrice * 0.05
    const shipping = Math.min(100, basePrice * 0.1)

    return basePrice - quantityDiscount + shipping
}

```
[note]
  Demo
  好处？
1. 把主逻辑从底层细节中剥离了，更可读
2. 代码自身就充当了注释的作用，
3. 消除了重复
[/note]

[slide]

```javascript
const calculatePrice = ({ itemPrice, quantity }) => {
    const basePrice = itemPrice * quantity
    const quantityDiscount = Math.max(0, quantity - 500) * itemPrice * 0.05
    const shipping = Math.min(100, basePrice * 0.1)

    return basePrice - quantityDiscount + shipping
}
```
### 下一步？

* 是否真的需要暴露底层的细节 {:&.fadeIn}
* 是否考虑重用
* 是否会滋生更长的函数
* 变量是否值回票价

[slide]
## Replace Temp with Query(用查询取代临时变量)
```javascript
const calculatePrice = ({ itemPrice, quantity }) => {
    const basePrice = itemPrice * quantity
    const quantityDiscount = Math.max(0, quantity - 500) * itemPrice * 0.05
    const shipping = Math.min(100, basePrice * 0.1)

    return basePrice - quantityDiscount + shipping
}
```

```javascript
const calculatePrice = (order) => {
    return basePrice(order) - quantityDiscount(order) + shipping(order)
}

const basePrice = ({ itemPrice, quantity }) => {
    return itemPrice * quantity
}

const quantityDiscount = ({ itemPrice, quantity }) => {
    return Math.max(0, quantity - 500) * itemPrice * 0.05
}

var shipping = function (order) {
    return Math.min(100, basePrice(order) * 0.1)
}
```

[note]
好处？
* 代码即注释，一目了然
* 更可重用
* 抽象层级更协调
[/note]


[slide]
## Performance concern of More Small Functions
* 一般都不会有明显的性能差别 {:&.fadeIn}
* 即使有细小的区别，代码可读性价值更高
* 若真有大问题，回退并不麻烦
* 总之，性能顾虑不能成为我们不抽方法的理由

[slide]

```javascript
const isBigDeal = (order) => {
  const basePrice = order.basePrice
  return basePrice > 1000
}
```
### 变量是否值回票价？

* 此处变量并没有带来更清晰的语义，反而显得冗余了 {:&.fadeIn}

[slide]
## Inline Temp Variable
```javascript
const isBigDeal = (order) => {
  const basePrice = order.basePrice
  return basePrice > 1000
}
```

```javascript
const isBigDeal = (order) => {
  return order.basePrice > 1000
}
```

[slide]

```javascript
let temp = 2 * height * width
console.log('perimeter: ', temp)

temp = height * width
console.log('area: ', temp)
```
### 重构信号？

* temp不表意，不可读 {:&.fadeIn}
* 对同一个变量多次赋值（并且内容并无关联） ，职责不单一

[slide]
## Split Temoary Variable(分解临时变量)
```javascript
let temp = 2 * height * width
console.log('perimeter: ', temp)

temp = height * width
console.log('area: ', temp)
```

```javascript
const perimeter = 2 * height * width
console.log('perimeter: ', perimeter)

const area = height * width
console.log('area: ', area)
```

[slide]

```javascript
const users = [
  { user: 'barney', age: 36, active: true },
  { user: 'fred',   age: 40, active: false }
]

const activeUsers = []
users.each(user => {
  if (user.active) {
    activeUsers.push(user)
  }
})
```
### 遇到循环的时候，多斟酌一下

* 少告诉计算机**怎么做**，而应告诉它**做什么** {:&.fadeIn}
* 不利于`Parallelize`
* 尽量熟悉集合的API，如**filter**、**map**、**reduce**、**find**等


[slide]
## Substitue Algorithm(替换算法)

```javascript
const users = [
  { user: 'barney', age: 36, active: true },
  { user: 'fred',   age: 40, active: false }
]

const activeUsers = []
users.each(user => {
  if (user.active) {
    activeUsers.push(user)
  }
})
```

* {:&.fadeIn}

```javascript
const activeUsers = filter(users, user => user.active)
```

```javascript
const activeUsers = filter(users, 'active')
```

[slide]
## More Examples of Substitue Algorithm
```javascript
const users = [
  { user: 'barney', age: 36, active: true },
  { user: 'fred',   age: 40, active: false }
]
```

[magic data-transition="cover-circle"]

```javascript
let activeUser = null

users.each(user => {
  if (user.active) {
    activeUser = user
    break
 }
})

```
====

```javascript
let totalAge = 0
users.each(user => {
  if (user.active) {
    totalAge += user.age
  }
})
```

[/magic]

[slide]

```javascript
const discount = (inputVal, quantity) => {
  if (inputVal > 50) inputVal -= 2 
  // ...
  return inputVal  
}
```
### 重构信号？

* 对参数赋值，降低代码清晰度 {:&.fadeIn}
* 对参数赋值，参数为引用时，极易引入副作用
* 
```javascript
  const aMethod = (aObj) => {
    aObj.modifyInSomeWay()  // That's ok
    aObj = anotherObj       // Bad
  }
```


[slide]
## Remove Assignments to Parameters(移除对参数的赋值)

```javascript
const discount = (inputVal, quantity) => {
  if (inputVal > 50) inputVal -= 2 
  // ...
  return inputVal  
}
```

```javascript
const discount = (basePrice, quantity) => {
  const finalPrice = basePrice
  if (basePrice > 50) finalPrice -= 2 
  // ...
  return finalPrice  
}

```

[slide]

## Extract Method(提取方法)

```javascript
const printOwing = (orders) => {
  let outstanding = 0.0
  // print banner
  console.log('*********************************')
  console.log('********* Customer Owes *********')
  console.log('*********************************')
  for (let i = 0; i < orders.length; i++) {
    outstanding += orders[i].amount
  }
  // print details
  console.log('name: ', name)
  console.log('amount: ', outstanding)
}
```
### 重构信号？
* 职责不单一 {:&.fadeIn}
* 注释
* 抽象层次不协调，以致于主逻辑淹没其中

[slide]

## Extract Method(提取方法)

```javascript
const printOwing = (orders) => {
  printBanner()
  
  // print details
  console.log('name: ', name)
  console.log('amount: ', outstanding)
}
```
  
```javascript
const printOwing = (orders) => {
  printBanner()
  const outstanding = calculateOutstanding(orders)
  printDetails(outstanding)
}

const printBanner = () => {
  console.log('*********************************')
  console.log('********* Customer Owes *********')
  console.log('*********************************')
}

var calculateOutstanding = function (orders) {
  return sumBy(orders, 'amount')
}

const printDetails = function (outstanding) {
  console.log('name: ', name)
  console.log('amount: ', outstanding)
}
```

[slide]
## Inline Method(内联方法)

```javascript
const getRating = () => {
    return moreThanFiveNegativeFeedbacks() ? 1 : 2
}

const moreThanFiveNegativeFeedbacks = () => {
    return this.negativeFeedbacks.length > 5
}

```

* {:&.fadeIn}

```javascript
const getRating = () => {
    return this.negativeFeedbacks.length > 5 ? 1 : 2
}

```

[slide]
## Summary
![refactory_flow](/images/refactory_flow.png)


[slide]
## Thank You & QA
