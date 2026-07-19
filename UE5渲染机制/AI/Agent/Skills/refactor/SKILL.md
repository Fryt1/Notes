---
name: refactor
description: 重构代码。识别代码坏味道，提供重构建议，优化代码结构和可读性
disable-model-invocation: false
allowed-tools: Read, Edit, Write, Grep, Glob
---

# 代码重构 Skill

识别代码问题并提供重构方案，在不改变功能的前提下优化代码。

## 重构原则

1. **保持功能不变**：重构不改变外部行为
2. **小步前进**：每次只做一个小改动
3. **持续测试**：每次改动后运行测试
4. **提高可读性**：让代码更易理解
5. **降低复杂度**：简化逻辑结构

## 代码坏味道识别

### 1. 重复代码（Duplicated Code）
**识别**：相同或相似的代码出现多次

**重构方法**：
- 提取函数（Extract Function）
- 提取类（Extract Class）
- 使用模板方法

```javascript
// ❌ 重复代码
function calculatePriceA(price) {
  const tax = price * 0.1;
  const total = price + tax;
  return total;
}
function calculatePriceB(price) {
  const tax = price * 0.1;
  const total = price + tax;
  return total;
}

// ✅ 提取公共函数
function calculatePrice(price) {
  const tax = price * 0.1;
  return price + tax;
}
```

### 2. 过长函数（Long Function）
**识别**：函数超过 20-30 行

**重构方法**：
- 提取函数
- 分解条件表达式
- 引入参数对象

```javascript
// ❌ 过长函数
function processOrder(order) {
  // 验证订单 (10行)
  // 计算价格 (10行)
  // 更新库存 (10行)
  // 发送通知 (10行)
}

// ✅ 拆分函数
function processOrder(order) {
  validateOrder(order);
  const price = calculatePrice(order);
  updateInventory(order);
  sendNotification(order);
}
```

### 3. 过大的类（Large Class）
**识别**：类有太多职责和方法

**重构方法**：
- 提取类
- 提取子类
- 提取接口

### 4. 过长参数列表（Long Parameter List）
**识别**：函数参数超过 3-4 个

**重构方法**：
- 引入参数对象
- 保持对象完整

```javascript
// ❌ 参数过多
function createUser(name, email, age, address, phone, role) {
  // ...
}

// ✅ 使用对象
function createUser(userInfo) {
  const { name, email, age, address, phone, role } = userInfo;
  // ...
}
```

### 5. 发散式变化（Divergent Change）
**识别**：一个类因不同原因被修改

**重构方法**：
- 提取类
- 分离关注点

### 6. 霰弹式修改（Shotgun Surgery）
**识别**：一个改动需要修改多个类

**重构方法**：
- 搬移函数
- 搬移字段
- 内联类

### 7. 依恋情结（Feature Envy）
**识别**：函数过度使用其他类的数据

**重构方法**：
- 搬移函数
- 提取函数

```javascript
// ❌ 依恋其他类
class Order {
  getPrice() {
    return this.product.basePrice * this.product.discount;
  }
}

// ✅ 搬移到合适的类
class Product {
  getDiscountedPrice() {
    return this.basePrice * this.discount;
  }
}
```

### 8. 数据泥团（Data Clumps）
**识别**：相同的数据项总是一起出现

**重构方法**：
- 提取类
- 引入参数对象

### 9. 基本类型偏执（Primitive Obsession）
**识别**：过度使用基本类型而非对象

**重构方法**：
- 以对象取代基本类型
- 提取类

```javascript
// ❌ 使用基本类型
function formatPhone(areaCode, prefix, number) {
  return `(${areaCode}) ${prefix}-${number}`;
}

// ✅ 使用对象
class PhoneNumber {
  constructor(areaCode, prefix, number) {
    this.areaCode = areaCode;
    this.prefix = prefix;
    this.number = number;
  }
  format() {
    return `(${this.areaCode}) ${this.prefix}-${this.number}`;
  }
}
```

### 10. 过多的条件判断（Switch Statements）
**识别**：大量的 if-else 或 switch

**重构方法**：
- 以多态取代条件表达式
- 策略模式

```javascript
// ❌ 过多条件
function getPrice(type) {
  if (type === 'regular') return 10;
  if (type === 'premium') return 20;
  if (type === 'vip') return 30;
}

// ✅ 使用对象映射
const PRICES = {
  regular: 10,
  premium: 20,
  vip: 30
};
function getPrice(type) {
  return PRICES[type];
}
```

### 11. 临时字段（Temporary Field）
**识别**：对象中的某些字段只在特定情况下有值

**重构方法**：
- 提取类
- 引入特例对象

### 12. 过度耦合的消息链（Message Chains）
**识别**：a.b().c().d()

**重构方法**：
- 隐藏委托关系
- 提取函数

```javascript
// ❌ 消息链
const street = user.getAddress().getCity().getStreet();

// ✅ 隐藏委托
const street = user.getStreet();
```

### 13. 中间人（Middle Man）
**识别**：类的大部分方法都委托给其他类

**重构方法**：
- 移除中间人
- 内联函数

### 14. 过度设计（Speculative Generality）
**识别**：为未来可能的需求设计复杂结构

**重构方法**：
- 移除不必要的抽象
- 内联类/函数

## 重构技巧

### 提取函数
```javascript
// Before
function printOwing(invoice) {
  console.log('***********************');
  console.log('**** Customer Owes ****');
  console.log('***********************');

  let outstanding = 0;
  for (const o of invoice.orders) {
    outstanding += o.amount;
  }

  console.log(`name: ${invoice.customer}`);
  console.log(`amount: ${outstanding}`);
}

// After
function printOwing(invoice) {
  printBanner();
  const outstanding = calculateOutstanding(invoice);
  printDetails(invoice, outstanding);
}
```

### 内联函数
```javascript
// Before
function getRating(driver) {
  return moreThanFiveLateDeliveries(driver) ? 2 : 1;
}
function moreThanFiveLateDeliveries(driver) {
  return driver.numberOfLateDeliveries > 5;
}

// After
function getRating(driver) {
  return driver.numberOfLateDeliveries > 5 ? 2 : 1;
}
```

### 提取变量
```javascript
// Before
if (platform.toUpperCase().indexOf('MAC') > -1 &&
    browser.toUpperCase().indexOf('IE') > -1) {
  // ...
}

// After
const isMac = platform.toUpperCase().indexOf('MAC') > -1;
const isIE = browser.toUpperCase().indexOf('IE') > -1;
if (isMac && isIE) {
  // ...
}
```

### 重命名
```javascript
// Before
function calc(a, b) {
  return a * b * 0.1;
}

// After
function calculateTax(price, quantity) {
  const TAX_RATE = 0.1;
  return price * quantity * TAX_RATE;
}
```

## 重构流程

1. **识别问题**：找出代码坏味道
2. **编写测试**：确保有测试覆盖
3. **小步重构**：每次只做一个改动
4. **运行测试**：确保功能不变
5. **提交代码**：每个重构步骤提交一次
6. **重复**：继续下一个重构

## 输出格式

```
🔍 **代码分析**
发现以下问题：
- [问题1]: [位置]
- [问题2]: [位置]

💡 **重构建议**
1. [重构方法1]
   - 原因：[为什么]
   - 好处：[改进点]

2. [重构方法2]
   - 原因：[为什么]
   - 好处：[改进点]

✅ **重构后代码**
[展示重构后的代码]

📊 **改进指标**
- 代码行数：100 → 80 (-20%)
- 圈复杂度：15 → 8
- 可读性：提升
```

## 何时不应该重构

- ❌ 代码完全需要重写
- ❌ 接近发布日期
- ❌ 没有测试覆盖
- ❌ 性能关键代码（需要 profiling）
