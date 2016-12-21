---
layout: post
title:  "Common Reactor Skills Part Two"
date:   2016-12-18 12:55:19 +0800
categories: jekyll update
---

[slide]
# Common Reactor Skills Part Two
# 常见重构手法（二）
## qinfanpeng

[slide]

## Agenda
* Replace Magic number with Constant {:&.fadeIn}
* Decompose Conditional
* Replace Nested Conditional with Guard Clauses
* Consolidate Conditional Expression
* Consolidate Duplicate Conditional Fragments 
* Remove Control Flag
* Sparate Query from Modifer
* Introduce Named Parameter


[slide]
## Agenda
* **Replace Magic number with Constant**
* Decompose Conditional
* **Replace Nested Conditional with Guard Clauses**
* Consolidate Conditional Expression
* Consolidate Duplicate Conditional Fragments 
* Remove Control Flag
* **Sparate Query from Modifer**
* **Introduce Named Parameter**


[slide]

```javascript
const calculatePrice = (order) => {
    return basePrice(order) - quantityDiscount(order) + shipping(order)
}

const quantityDiscount = ({ itemPrice }) => {
    return Math.max(0, quantity - 500) * itemPrice * 0.05
}

const shipping = function (order) {
    return Math.min(100, basePrice(order) * 0.1)
}

```

### 改善空间？

* 魔法数不可读，妨碍理解 {:&.fadeIn}
* 魔法数容易重复，从而散落于代码各处，致使散弹式修改
* 不具备可搜索性


[slide]
## Replace Magic number with Constant(用常量替换魔法数)

```javascript
const QUANTITY_DISCOUNT_THRESHOLD = 500
const QUANTITY_DISCOUNT_RATE = 500
const SHIPPING_RATE = 0.1
const MIN_SHIPPING = 100

const quantityDiscount = ({ itemPrice }) => {
    return Math.max(0, quantity - QUANTITY_DISCOUNT_THRESHOLD) * itemPrice * QUANTITY_DISCOUNT_RATE
}

var shipping = function (order) {
    return Math.min(MIN_SHIPPING, basePrice(order) * SHIPPING_RATE)
}

```
[note]
* 命名良好的常量，就像文档一样，易于理解 {:&.fadeIn}
* 便于修改

[/note]

[slide]
## 思考 - 放置常量的位置？

* 变化频率 {:&.fadeIn}
* 使用范围

[note]
变化越频繁、使用范围越广，更应放到全局性的地方，甚至是配置文件；使用范围、变化越少，则可就近放置（比如当前文件顶部？）
[/note]

[slide]
## 字符串字面量与魔法数的异同？

```javascript
const shipping = function (order) {
    return Math.min(100, basePrice(order) * 0.1)
}
```

```javascript
determineLegendAlignment(model, ['top', 'topCenter'])
determineLegendAlignment(model, ['left', 'leftMiddle'])
determineLegendAlignment(model, ['right', 'rightMiddle', 'rightBottom'])
determineLegendAlignment(model, ['bottom', 'bottomCenter'])
```

### 和魔法数的异同？

* “魔法数不可读，妨碍理解”，字符串字面量无此问题 {:&.fadeIn}
* “魔法数容易重复，从而散落于代码各处，致使散弹式修改”，有此问题，但频率不高
* 字符串字面量容易出现拼写错误
* 要不要抽常量？**个人觉得尽量抽**


[slide]

```javascript
const calculateCharge = (date, quantity) => {
  if (SUMMER_START <= date && date <= SUMMER_END) {
    return quantity * SUMMER_RATE
  } else {
    return quantity * WINTER_RATE + WINTER_SERVICE_CHARGE 
  }
}
```

### 改善空间？

* 未能突出领域概念 {:&.fadeIn}
* 未能突出各个分支

[slide]
## Decompose Conditional(分解条件表达式)
```javascript
const calculateCharge = (date, quantity) => {
  if (SUMMER_START <= date && date <= SUMMER_END) {
    return quantity * SUMMER_RATE
  } else {
    return quantity * WINTER_RATE + WINTER_SERVICE_CHARGE 
  }
}
```

```javascript
const calculateCharge = (date, quantity) => {
  return isInSummer(date) ? summerCharge(date) : winterCharge(date)
}

const isInSummer = (date) => SUMMER_START <= date && date <= SUMMER_END

const summerCharge = (quantity) => quantity * SUMMER_RATE

const winterCharge = (quantity) => quantity * WINTER_RATE + WINTER_SERVICE_CHARGE
```

[slide]

[magic data-transition="cover-circle"]
## 最容易从以下代码结构联想到什么？

```javascript
const calculatePayAmount = () => {
  let result = 0.0

  if (isDead()) {
    result = deadAmount()
  } else {
    if (isSeparated()) {
      result = separatedAmount()
    } else {
      if (isRetired()) {
        result = retiredAmount()
      } else {
        result = normalPayAmount()
      }
    }
  }

  return result
}
```

====

![迷宫](/images/maze.jpg)
====
![鹿角](/images/hartshorn.jpg)
====
## 改善空间？
* 条件嵌套，致使阅读负担陡增, 不容易找出主要逻辑
* 摆脱`单一出口原则`的束缚
* 乱用 if-else 结构

[/magic]

[slide]

## if-else 结构意味着什么？

* 两个分支都是正常情况 {:&.zoomIn}
* 两个分支重要程度相当，该受到相同的重视程度
* 两个分支发生的概率相当
* 两个分支确实属于<em>**非此即彼**</em>的关系

[slide]
## Guard Clauses（卫语句）

```javascript
const calculatePayAmount = () => {
  if (isDead()) return deadAmount()
  if (isSeparated()) return separatedAmount()
  
  // ...
  // ...
  // ...
```

### 卫语句意味着什么？
  
* 这种情况很罕见，如果它真地发生了，稍作处理，退出即可 {:&.zoomIn}


[slide]
## 何时使才用 if-else？

* 两个分支都是正常情况 {:&.zoomIn}
* 两个分支重要程度相当，该受到相同的重视程度
* 两个分支发生的概率相当
* 两个分支确实属于<em>**非此即彼**</em>的关系
* **两个分支代码量相当**


[slide]
[magic data-transition="cover-circle"]
## 影响选用 if-else/卫语句的因数
### -- 主要观察点：两个分支是否都是正常情况 

```javascript
const computeDisabilityAmount = () => {
  if (isNotEligibleForDisability()) {
    return 0.0
  } else {
  	// compute disability amount
  	// ...
  	// ...
  }
}

const isNotEligibleForDisability = () => {
  return seniority < 0 || monthsDisabled > 12 || isPartTime()
}
```

====

```javascript
const computeDisabilityAmount = () => {
  if (isNotEligibleForDisability()) return 0.0
  
  // compute disability amount
  // compute disability amount
  // ...
  // ...
}

const isNotEligibleForDisability = () => {
  return seniority < 0 || monthsDisabled > 12 || isPartTime()
}
```
[/magic]

[slide]
[magic data-transition="cover-circle"]
## 影响选用 if-else/卫语句的因数
### -- 主要观察点：两个分支代码量是否相当 

```javascript
handleCellWithTooltip(fieldValue, field) {
    if (this.props.needTooltipFieldNames.includes(field)) {
      const tooltipId = uniqueId('fieldValue_')
      return (
        <span key={tooltipId}>
          <div data-tip data-for={tooltipId}>{fieldValue}</div>
          <Tooltip id={tooltipId} place='right'>
            <div>{fieldValue}</div>
          </Tooltip>
        </span>
      )
    } else {
      return <span key={uniqueId('cell_')}>{fieldValue}</span>
    }
  }
```
====

```javascript
handleCellWithTooltip(fieldValue, field) {
    if (!this.props.needTooltipFieldNames.includes(field)) return <span key={uniqueId('cell_')}>{fieldValue}</span>
    
   const tooltipId = uniqueId('fieldValue_')
   return (
     <span key={tooltipId}>
       <div data-tip data-for={tooltipId}>{fieldValue}</div>
       <Tooltip id={tooltipId} place='right'>
         <div>{fieldValue}</div>
       </Tooltip>
     </span>
   )
}
```
[/magic]

[slide]

## Replace Nested Conditional with Guard Clauses (用卫语句替换嵌套条件表达式)

<div class="columns-2"><pre><code class="javascript">
const calculatePayAmount = () => {
  let result = 0.0

  if (isDead()) {
    result = deadAmount()
  } else {
    if (isSeparated()) {
      result = separatedAmount()
    } else {
      if (isRetired()) {
        result = retiredAmount()
      } else {
        result = normalPayAmount()
      }
    }
  }

  return result
}

</code></pre><pre><code class="javascript">

const calculatePayAmount = () => {
  if (isDead()) return deadAmount()
  if (isSeparated()) return separatedAmount()
  if (isRetired()) return retiredAmount()
  
  return normalPayAmount()
}

</code></pre></div>

[slide]
[magic data-transition="cover-circle"]
## 消除条件嵌套
```javascript
const getAdjustedCapital = () => {
  const result = 0

  if (capital > 0) {
    if (intRate > 0 && duration > 0) {
      result = (_income / _duration) * ADJ_FACTOR
    }
  }

  return result
}
```
====
## 条件取反 -- 识别异常情况

```javascript
const getAdjustedCapital = () => {
  if (capital <= 0) return 0
  if (!(intRate > 0 && duration > 0)) return 0
  
  return (_income / _duration) * ADJ_FACTOR
}
```

====
## 简化取反后的条件

```javascript
const getAdjustedCapital = () => {
  if (capital <= 0) return 0
  if (intRate <= 0 || duration <= 0)) return 0
  
  return (_income / _duration) * ADJ_FACTOR
}

```

[/magic]


[slide]
## 使用 if-else 的注意事项

* **尽量避免条件表达式嵌套** {:&.zoomIn}
* 个人建议**别**优先使用 if-else


[slide]

```javascript
const computeDisabilityAmount = () => {
  if (seniority < 0) return 0
  if (monthsDisabled > 12) return 0
  if (isPartTime()) return 0

  // compute disability amount
}
```

### 改善空间？

* 喧宾夺主，函数很大一部分空间被验证逻辑霸占 {:&.fadeIn}
* 可以合并的条件检测

[slide]
## Consolidate Conditional Expression(合并条件表达式)

```javascript
const computeDisabilityAmount = () => {
  if (seniority < 0) return 0
  if (monthsDisabled > 12) return 0
  if (isPartTime()) return 0

  // compute disability amount
}
```

* {:&.fadeIn}

```javascript
const computeDisabilityAmount = () => {
  if (seniority < 0 || monthsDisabled > 12 || isPartTime()) return 0
  // compute disability amount
}
```

```javascript
const computeDisabilityAmount = () => {
  isNotEligibleForDisability() return 0
  // compute disability amount
}

const isNotEligibleForDisability = () => {
  return seniority < 0 || monthsDisabled > 12 || isPartTime()
}

```

[slide]

```javascript
if (isSpecialDeal()) {
  totalPrice = price * SPECIAL_DEAL_DISCOUNT_RATE
  send()
} else {
  totalPrice = price * NORMAL_DISCOUNT_RATE
  send()
}
```

### 改善空间？

* 并没严格把**变**和**不变**区分开 {:&.fadeIn}


[slide]
## Consolidate Duplicate Conditional Fragments (合并重复条件代码片段)

```javascript
if (isSpecialDeal()) {
  totalPrice = price * SPECIAL_DEAL_DISCOUNT_RATE
  send()
} else {
  totalPrice = price * NORMAL_DISCOUNT_RATE
  send()
}
```

```javascript
if (isSpecialDeal()) {
  totalPrice = price * SPECIAL_DEAL_DISCOUNT_RATE
} else {
  totalPrice = price * NORMAL_DISCOUNT_RATE
}
send()
```

[slide]
## Demo of Consolidate Duplicate Conditional Fragments 

```javascript
handleMenuOnChange(selectedItem) {
  if (this.props.length > 1) {
    this.setState({
      selectedMeasure:{
        fieldId: selectedItem.value
      }
    })
  }
  else {
    this.setState({
      selectedMeasure: selectedItem.measure
    })
  }
}
```

* {:&.fadeIn}

```javascript
handleMenuOnChange(selectedItem) {
  const selectedMeasure = this.props.measures.length > 1 ? {fieldId: selectedItem.value} : selectedItem.measure 
  this.setState({ selectedMeasure })
}
```

[slide]

```javascript
const checkSecurity = (people) => {
  const miscreant = findMiscreant(people)
  // doSomethingElse(miscreant)
}

const findMiscreant = (people) => {
  let found = false
  people.forEach(person => {
    if (!found) {
      if (person === 'Bob') {
        sendAlert()
        found = true
      }
      if (person === 'Jack') {
        sendAlert()
        found = true
      }
    }
  })
}
```

### 改善空间？
* 丑陋的控制位（found） {:&.fadeIn}
* 查询函数中暗含副作用，且无法从函数名中看出
* 臃肿的 for 循环


[slide]

[magic data-transition="cover-circle"]
## Remove Control Flag (移除控制位)
### -- 移除前

```javascript
const findMiscreant = (people) => {
  let found = false
  people.forEach(person => {
    if (!found) {
      if (person === 'Bob') {
        sendAlert()
        found = true
      }
      if (person === 'Jack') {
        sendAlert()
        found = true
      }
    }
  })
}
```
====

## Remove Control Flag (移除控制位)
### -- 移除后

```javascript
const findMiscreant = (people) => {
  people.forEach(person => {
      if (person === 'Bob') {
        sendAlert()
        break;
      }
      if (person === 'Jack') {
        sendAlert()
        break;
      }
    }
  })
}
```
[/magic]

[slide]
[magic data-transition="cover-circle"]

## Sparate Query from Modifer (将查询函数和修改函数分开)
### -- 分离前

```javascript
const findMiscreant = (people) => {
  people.forEach(person => {
      if (person === 'Bob') {
        sendAlert()
        break;
      }
      if (person === 'Jack') {
        sendAlert()
        break;
      }
    }
  })
}
```

====
## Sparate Query from Modifer (将查询函数和修改函数分开)
### -- 分离后

```javascript
const checkSecurity = (people) => {
  const miscreant = findMiscreant(people)
  if (!isUndefined(miscreant)) sendAlert(miscreant)
  // doSomethingElse(miscreant)
}

const findMiscreant = (people) => {
  const miscreants = ['Bob', 'Jack']
  return find(people, person => miscreants.includes(person))
}
```
====

## Sparate Query from Modifer (将查询函数和修改函数分开)
### -- 万不得已

```javascript
const findMiscreantAndSendAlert = (people) => {
  people.forEach(person => {
    if (!found) {
      if (person === 'Bob') {
        sendAlert(person)
        return person
      }
      if (person === 'Jack') {
        sendAlert(person)
        return person
      }
    }
  })
}
```
[/magic]


[slide]

```javascript
const SearchCriteria = buildSearchCriteria(5, "0201485672")
```

* {:&.fadeIn}

```javascript
const SearchCriteria = buildSearchCriteria(5, "0201485672", false)
```

```javascript
buildSearchCriteria(5, "0201485672", false, 'JavaScript : The Good Parts')
```

```javascript
const buildSearchCriteria = (authorId, ISBN, includeSoldOut, title) => {
  // ...
}
```


### 改善空间？
* 实参为数字、布尔值时不可读 {:&.fadeIn}
* 参数较多时，逻辑混乱，不易理解，且容易弄错参数顺序


[slide]

[magic data-transition="cover-circle"]
## Introduce Named Parameter (引入具名参数)

```javascript
const buildSearchCriteria = ({ authorId, ISBN, includeSoldOut, title }) => {
  // ...
}
```

```javascript
const SearchCriteria = buildSearchCriteria({
  authorId: 5, 
  ISBN: "0201485672", 
  includeSoldOut: false, 
  title: 'JavaScript : The Good Parts'
})
```
====

## 至少提取解释变量

```javascript
const **authorId** = 5
const ISBN = '0201485672'
const includeSoldOut = false
const title = 'JavaScript The Good Parts'

const SearchCriteria = buildSearchCriteria(authorId, ISBN, includeSoldOut, title)
```

[/magic]


[slide]
## Thank You & QA


