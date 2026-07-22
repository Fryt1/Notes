# Shader 绑定机制 - 完整数据流向

## 📊 完整流程图

```
编译时（Cook）:
┌─────────────────────────────────────────────────────────────┐
│ 1. 引擎启动                                                  │
│    └─> 扫描所有 IMPLEMENT_MATERIAL_SHADER_TYPE 宏           │
│        └─> FShaderTypeRegistration 收集所有 Shader 类型     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. Shader 编译阶段                                           │
│    └─> PrepareMaterialShaderCompileJob()                    │
│        ├─> ShaderType->GetShaderFilename()                  │
│        │   返回: "/Engine/Private/DepthOnlyVertexShader.usf"│
│        ├─> ShaderType->GetFunctionName()                    │
│        │   返回: "Main"                                      │
│        └─> GlobalBeginCompileShader()                       │
│            └─> 编译 USF → GPU 字节码                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. 保存到 ShaderCache                                        │
│    Key: ShaderType 的唯一 ID                                 │
│    Value: 编译好的 GPU 字节码                                │
└─────────────────────────────────────────────────────────────┘

运行时（游戏运行）:
┌─────────────────────────────────────────────────────────────┐
│ 1. PassProcessor 需要 Shader                                │
│    └─> MaterialResource.TryGetShaders(ShaderTypes, ...)     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. 查找全局 Shader Map                                       │
│    └─> 根据 ShaderType ID 找到对应的 GPU 字节码             │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. 创建 Shader 实例                                          │
│    └─> new TDepthOnlyVS(CompiledShaderInitializer)          │
│        └─> 绑定参数（Bind）                                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. 使用 Shader                                               │
│    └─> BuildMeshDrawCommands()                              │
│        └─> 设置参数（ShaderBindings.Add）                   │
│            └─> 提交到 GPU                                    │
└─────────────────────────────────────────────────────────────┘
```

## 🔑 关键代码位置总结

### 宏定义层
- **DECLARE_SHADER_TYPE**: 声明 Shader 类的静态类型信息
  - 位置：Shader.h
  - 作用：生成反射代码、序列化函数

- **IMPLEMENT_MATERIAL_SHADER_TYPE**: 注册 Shader 到全局系统
  - 位置：MaterialShaderType.h (line 13)
  - 转发到：IMPLEMENT_SHADER_TYPE (Shader.h line 1724)
  - 作用：创建 StaticType，注册到 FShaderTypeRegistration

### 注册层
- **FShaderTypeRegistration**: 延迟注册机制
  - 位置：Shader.h (line 1743)
  - 作用：引擎启动时统一构造所有 Shader 类型

### 编译层
- **PrepareMaterialShaderCompileJob**: 准备编译任务
  - 位置：MaterialShader.cpp (line 1865)
  - 关键调用：
    - `ShaderType->GetShaderFilename()` → 获取 USF 路径
    - `ShaderType->GetFunctionName()` → 获取入口函数
    - `GlobalBeginCompileShader()` → 启动编译

### 运行时层
- **MaterialResource.TryGetShaders**: 查找已编译的 Shader
  - 作用：从 ShaderCache 加载 GPU 字节码
  - 返回：FMaterialShaders 对象

## 💡 核心理解

### 为什么需要两个宏？

1. **DECLARE_SHADER_TYPE**（声明）
   - 在 .h 文件中
   - 告诉编译器："这个类是一个 Shader"
   - 生成必要的静态成员和函数

2. **IMPLEMENT_MATERIAL_SHADER_TYPE**（实现）
   - 在 .cpp 文件中
   - 告诉引擎："这个 Shader 对应哪个 USF 文件"
   - 注册到全局 Shader 系统

### 类比：身份证系统

```
DECLARE_SHADER_TYPE          → 申请身份证（声明我需要身份）
IMPLEMENT_MATERIAL_SHADER_TYPE → 办理身份证（填写详细信息）
FShaderTypeRegistration      → 户籍管理系统（统一登记）
ShaderCache                  → 档案库（存储编译结果）
TryGetShaders                → 查档案（运行时查找）
```

## 🎯 验证理解的三个问题

1. **Q: 如果我只写了 DECLARE_SHADER_TYPE，没写 IMPLEMENT_MATERIAL_SHADER_TYPE，会怎样？**
   - A: 链接错误！因为 StaticType 没有定义，GetStaticType() 找不到实现。

2. **Q: USF 文件路径写错了会怎样？**
   - A: 编译时报错："找不到 Shader 源文件"。

3. **Q: 入口函数名写错了会怎样？**
   - A: 编译时报错："找不到入口函数 Main"。

## 📝 你的笔记已经很好了！

你已经正确理解了：
- ✅ 宏的转发关系
- ✅ StaticType 的创建
- ✅ FShaderTypeRegistration 的延迟注册
- ✅ 编译时如何使用这些信息

建议补充：
- 运行时如何查找和使用 Shader（TryGetShaders）
- 完整的数据流向图（上面已提供）
