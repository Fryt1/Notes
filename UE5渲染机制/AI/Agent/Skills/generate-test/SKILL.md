---
name: generate-test
description: 为函数或模块自动生成单元测试。当需要编写测试、提高测试覆盖率时使用
disable-model-invocation: false
allowed-tools: Read, Write, Grep, Glob
---

# 测试生成 Skill

自动为代码生成完整的单元测试。

## 测试生成策略

### 1. 分析目标代码
- 识别函数签名（参数、返回值）
- 理解函数功能
- 找出边界条件
- 识别依赖项

### 2. 设计测试用例
**测试类型**：
- ✅ 正常情况（Happy Path）
- ⚠️ 边界情况（Edge Cases）
- ❌ 异常情况（Error Cases）

**测试覆盖**：
- 所有分支路径
- 边界值（0, 1, -1, null, undefined, 空数组）
- 异常输入
- 异步操作

### 3. 选择测试框架
根据项目自动识别：
- **JavaScript**: Jest, Mocha, Vitest
- **Python**: pytest, unittest
- **Java**: JUnit
- **Go**: testing

### 4. 生成测试代码
遵循 AAA 模式：
- **Arrange**: 准备测试数据
- **Act**: 执行被测函数
- **Assert**: 验证结果

## 测试模板

### JavaScript (Jest)
```javascript
describe('functionName', () => {
  // 正常情况
  test('should return expected result with valid input', () => {
    const result = functionName(validInput);
    expect(result).toBe(expectedOutput);
  });

  // 边界情况
  test('should handle empty input', () => {
    const result = functionName('');
    expect(result).toBe(defaultValue);
  });

  test('should handle null/undefined', () => {
    expect(functionName(null)).toBe(null);
    expect(functionName(undefined)).toBe(undefined);
  });

  // 异常情况
  test('should throw error with invalid input', () => {
    expect(() => functionName(invalidInput)).toThrow();
  });

  // 异步测试
  test('should resolve with data', async () => {
    const result = await asyncFunction();
    expect(result).toEqual(expectedData);
  });
});
```

### Python (pytest)
```python
def test_function_name_valid_input():
    """测试正常输入"""
    result = function_name(valid_input)
    assert result == expected_output

def test_function_name_empty_input():
    """测试空输入"""
    result = function_name('')
    assert result == default_value

def test_function_name_invalid_input():
    """测试异常输入"""
    with pytest.raises(ValueError):
        function_name(invalid_input)
```

## 测试用例设计

### 数值函数
```javascript
// 测试: add(a, b)
- 正数相加: add(2, 3) → 5
- 负数相加: add(-2, -3) → -5
- 零值: add(0, 5) → 5
- 浮点数: add(0.1, 0.2) → 0.3
- 边界值: add(MAX_INT, 1) → ?
```

### 字符串函数
```javascript
// 测试: formatName(name)
- 正常名字: "john doe" → "John Doe"
- 空字符串: "" → ""
- null/undefined: null → ""
- 特殊字符: "o'brien" → "O'Brien"
- 多个空格: "john  doe" → "John Doe"
```

### 数组函数
```javascript
// 测试: filterActive(users)
- 正常数组: [{active:true}] → [{active:true}]
- 空数组: [] → []
- null: null → []
- 混合数据: [{active:true}, {active:false}]
```

### 异步函数
```javascript
// 测试: fetchUser(id)
- 成功获取: id=1 → {user data}
- 用户不存在: id=999 → null
- 网络错误: → throw Error
- 超时: → throw TimeoutError
```

## Mock 和 Stub

### Mock 外部依赖
```javascript
// Mock API 调用
jest.mock('./api', () => ({
  fetchData: jest.fn(() => Promise.resolve(mockData))
}));

// Mock 数据库
jest.mock('./db', () => ({
  query: jest.fn(() => mockResult)
}));
```

### Stub 时间相关
```javascript
// Mock 日期
jest.useFakeTimers();
jest.setSystemTime(new Date('2024-01-01'));
```

## 输出格式

生成测试后提供：
```
✅ 已生成测试文件：tests/function.test.js

📊 测试覆盖：
- 正常情况: 3 个测试
- 边界情况: 4 个测试
- 异常情况: 2 个测试

🧪 运行测试：
npm test tests/function.test.js

📝 测试说明：
- [测试1的目的]
- [测试2的目的]
```

## 最佳实践

- ✅ 测试名称清晰描述测试内容
- ✅ 每个测试只验证一个行为
- ✅ 使用有意义的测试数据
- ✅ Mock 外部依赖
- ✅ 测试应该快速运行
- ❌ 避免测试实现细节
- ❌ 避免测试之间相互依赖
