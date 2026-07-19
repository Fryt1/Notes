---
name: fix-bug
description: 快速定位和修复 bug。根据错误信息、堆栈跟踪或问题描述自动分析并修复
disable-model-invocation: false
allowed-tools: Read, Edit, Write, Grep, Bash
---

# Bug 修复 Skill

快速定位和修复代码中的 bug。

## 修复流程

### 1. 理解问题
- 阅读错误信息
- 分析堆栈跟踪
- 理解预期行为 vs 实际行为

### 2. 定位问题
**根据错误类型定位**：
- `TypeError`: 类型使用错误
- `ReferenceError`: 变量未定义
- `SyntaxError`: 语法错误
- `RangeError`: 数值超出范围
- `null/undefined`: 空值访问

**定位工具**：
```bash
# 搜索错误相关代码
Grep 搜索关键字
# 查看相关文件
Read 读取文件
```

### 3. 分析原因
常见 bug 类型：
- **逻辑错误**：条件判断错误、循环边界
- **空值问题**：未检查 null/undefined
- **异步问题**：回调地狱、Promise 未处理
- **类型错误**：类型转换、参数类型
- **边界情况**：数组越界、除零

### 4. 实施修复
- 使用 Edit 工具精确修改
- 添加必要的错误处理
- 补充边界检查

### 5. 验证修复
- 运行测试
- 检查是否引入新问题
- 确认修复有效

## 修复模板

```
🐛 **问题分析**
错误类型：[TypeError/ReferenceError/...]
出错位置：[文件:行号]
错误原因：[根本原因]

🔍 **定位过程**
1. [步骤1]
2. [步骤2]

✅ **修复方案**
[具体修改内容]

📝 **修改文件**
- file1.js: [改动说明]
- file2.js: [改动说明]

🧪 **验证建议**
- [测试方法1]
- [测试方法2]
```

## 常见 Bug 修复示例

### 空值访问
```javascript
// ❌ 错误
const name = user.profile.name;

// ✅ 修复
const name = user?.profile?.name || 'Unknown';
```

### 异步处理
```javascript
// ❌ 错误
function getData() {
  fetch(url).then(res => data = res);
  return data; // undefined
}

// ✅ 修复
async function getData() {
  const res = await fetch(url);
  return res;
}
```

### 循环边界
```javascript
// ❌ 错误
for (let i = 0; i <= arr.length; i++) {
  console.log(arr[i]); // 最后一次越界
}

// ✅ 修复
for (let i = 0; i < arr.length; i++) {
  console.log(arr[i]);
}
```

## 预防措施

修复后建议添加：
- ✅ 输入验证
- ✅ 错误处理
- ✅ 边界检查
- ✅ 单元测试
