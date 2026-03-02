# 📖 学习任务 (Study Tasks)

- [x] **Shader 绑定：** 搜索宏 `IMPLEMENT_MATERIAL_SHADER_TYPE`，理解 C++ 类与 USF 文件的映射关系。
- [x] **参数传递：** 学习 `FMeshDrawShaderBindings`，理解如何把 C++ 的 `FVector` 变成 Shader 里的 `uniform float3`。
- [ ] **图元类型：** 查找 `FMeshDrawCommand.PrimitiveType`，了解 `PT_TriangleList`, `PT_LineList`, `PT_PointList` 的区别。

---

## 1️⃣ Shader 绑定机制

### 核心问题：为什么需要这两个宏？

UE5 的渲染系统有两部分代码：
- **C++ 代码**：运行在 CPU 上，负责逻辑控制
- **Shader 代码（.usf 文件）**：运行在 GPU 上，负责实际渲染

**核心问题：C++ 怎么知道去哪里找对应的 Shader 文件？**

比如：
```cpp
// C++ 类
class TDepthOnlyVS : public FMeshMaterialShader
{
    // ...
};
```

这个 C++ 类对应的 Shader 代码在哪个文件？入口函数叫什么名字？

**答案：通过宏来"注册"这个映射关系！**

---

### 两个宏的分工

#### DECLARE_SHADER_TYPE（在 .h 文件）
**作用：为 Shader 类声明必需的"基础设施"**

```cpp
class TDepthOnlyVS : public FMeshMaterialShader
{
    DECLARE_SHADER_TYPE(TDepthOnlyVS, MeshMaterial);
    // ↑ 告诉编译器：这个类是 Shader，需要特殊处理
};
```

这个宏会自动生成：
- ✅ 静态类型信息（让引擎知道"这是个 Shader 类"）
- ✅ 序列化函数（保存/加载 Shader）
- ✅ 反射函数（运行时查询 Shader 信息）

**类比：** 就像给类办"营业执照"，有了这个才能在引擎的 Shader 系统里"合法经营"。

#### IMPLEMENT_MATERIAL_SHADER_TYPE（在 .cpp 文件）
**作用：绑定 C++ 类和 USF 文件的具体信息**

```cpp
IMPLEMENT_MATERIAL_SHADER_TYPE(, TDepthOnlyVS,
    TEXT("/Engine/Private/DepthOnlyVertexShader.usf"),  // USF 文件路径
    TEXT("Main"),                                        // 入口函数名
    SF_Vertex);                                          // Shader 频率（类型）
```

这个宏告诉引擎：
- ✅ **USF 在哪**：`/Engine/Private/DepthOnlyVertexShader.usf`
- ✅ **入口函数**：`Main`（USF 文件里必须有这个函数）
- ✅ **Shader 频率**：`SF_Vertex`（顶点着色器）

**Shader 频率（Frequency）类型：**
- `SF_Vertex` → Vertex Shader（顶点着色器）
- `SF_Pixel` → Pixel Shader（像素着色器）
- `SF_Compute` → Compute Shader（计算着色器）

---

### 完整对应关系

```
C++ 类                          USF 文件
┌─────────────────────┐        ┌──────────────────────────┐
│ TDepthOnlyVS        │   ←→   │ DepthOnlyVertexShader.usf│
│                     │        │                          │
│ DECLARE_SHADER_TYPE │        │ void Main(...)           │
│ (声明基础设施)       │        │ {                        │
│                     │        │     // Shader 代码        │
│ IMPLEMENT_xxx       │        │ }                        │
│ (绑定文件和入口)     │        │                          │
└─────────────────────┘        └──────────────────────────┘
         ↓                              ↑
    告诉引擎去这里找 ──────────────────┘
```

---

### 🤔 思考：如果入口函数名写错了？

**问题：如果我把入口函数名写成 `TEXT("MainVS")`，但 USF 文件里函数叫 `Main`，会怎样？**

**答案：编译时报错！**

```
引擎启动
  ↓
扫描所有 IMPLEMENT_MATERIAL_SHADER_TYPE
  ↓
准备编译 Shader
  ↓
读取 /Engine/Private/DepthOnlyVertexShader.usf
  ↓
查找入口函数 "MainVS"  ← 找不到！
  ↓
❌ 编译错误：Entry point 'MainVS' not found
```

---

### 序列化和反序列化的时机

#### 什么是序列化？

**序列化**：把 Shader 的编译结果（GPU 字节码）保存到硬盘
**反序列化**：从硬盘加载已编译的 Shader

#### ⏰ 时机详解

**序列化（保存）- Cook 时**

```
【开发阶段 - Cook 项目】
1. 引擎扫描所有 Shader 类
   └─> 发现 TDepthOnlyVS

2. 编译 Shader
   └─> 读取 DepthOnlyVertexShader.usf
   └─> 编译成 GPU 字节码

3. 序列化（保存）
   └─> 把 GPU 字节码写入 .ushaderbytecode 文件
   └─> 保存到 Saved/Cooked/Platform/ShaderCache/
```

**反序列化（加载）- 游戏运行时**

```
【游戏运行时】
1. 引擎启动
   └─> 加载 ShaderCache

2. PassProcessor 需要 Shader
   └─> MaterialResource.TryGetShaders(TDepthOnlyVS)

3. 反序列化（加载）
   └─> 从 ShaderCache 读取 GPU 字节码
   └─> 调用 TDepthOnlyVS 的反序列化函数
   └─> 创建 Shader 实例

4. 使用 Shader
   └─> 绑定到渲染管线
```

---

### 🤔 思考：为什么需要序列化？

**问题：为什么需要序列化？直接每次运行时编译 Shader 不行吗？**

**答案：太慢了！**

- 编译一个 Shader 可能需要几秒甚至几十秒
- 一个游戏可能有几千个 Shader
- 如果每次启动都编译，游戏要等很久才能运行

所以：
- **开发时**：编译一次，保存结果（序列化）
- **运行时**：直接加载结果（反序列化），秒开！

---

### 游戏第一次启动的"编译着色器"是什么？

#### 💡 真相：不是在"编译"，而是在"准备"

你在正式版游戏第一次启动时看到的：

```
显示：      "Compiling Shaders... 45%"
实际在做：   创建 PSO（Pipeline State Object）
```

#### 🔧 PSO 是什么？

**PSO = Pipeline State Object（渲染管线状态对象）**

```
Shader 字节码（已经编译好）
    ↓
需要和 GPU 驱动"握手"
    ↓
创建 PSO（GPU 可以直接使用的对象）
    ↓
保存到 PSO Cache
```

#### 📊 完整流程

```
【Cook 时】
USF 源码 → 编译 → Shader 字节码
                    ↓
                保存到 ShaderCache
                    ↓
                打包到游戏

【第一次运行游戏】
加载 Shader 字节码
    ↓
创建 PSO（和显卡驱动交互）← 这里显示"编译着色器"
    ↓
保存到 PSO Cache
    ↓
下次启动直接加载 PSO Cache（秒开）

【第二次及以后运行】
直接加载 PSO Cache
    ↓
不显示"编译着色器"（很快）
```

#### 🎯 为什么第一次慢？

1. **Shader 字节码是通用的**
   - 已经在 Cook 时编译好
   - 打包在游戏里

2. **PSO 是显卡特定的**
   - 不同显卡需要不同的 PSO
   - 不同驱动版本需要不同的 PSO
   - **无法预先打包**，必须在玩家电脑上创建

3. **第一次运行**
   - 为你的显卡创建 PSO
   - 保存到本地缓存
   - 显示"编译着色器"进度条

4. **第二次运行**
   - 直接加载 PSO Cache
   - 秒开，不显示进度条

#### 🤔 思考：更新显卡驱动后为什么又慢了？

**问题：为什么游戏更新显卡驱动后，又会显示"编译着色器"？**

**答案：因为 GPU 驱动改了，原本的 PSO 需要重新和 GPU 建立连接**

```
更新显卡驱动
    ↓
旧的 PSO Cache 失效（驱动接口变了）
    ↓
需要重新创建 PSO（重新"握手"）
    ↓
又显示"编译着色器"进度条
    ↓
创建新的 PSO Cache
```

---

### 📊 三个阶段总结

| 阶段 | 时机 | 做什么 | 结果 |
|------|------|--------|------|
| **1. 编译 Shader** | Cook 时 | USF → GPU 字节码 | ShaderCache（打包到游戏） |
| **2. 创建 PSO** | 第一次运行 | 字节码 + 显卡驱动 → PSO | PSO Cache（本地缓存） |
| **3. 使用 PSO** | 后续运行 | 直接加载 PSO | 秒开 |

---

### ✅ 第一步核心理解

1. ✅ **两个宏的作用**
   - DECLARE_SHADER_TYPE：声明基础设施（营业执照）
   - IMPLEMENT_MATERIAL_SHADER_TYPE：绑定 USF 文件（具体信息）

2. ✅ **序列化和反序列化**
   - 序列化：Cook 时保存编译结果
   - 反序列化：运行时加载编译结果
   - 目的：避免每次运行都重新编译

3. ✅ **"编译着色器"进度条**
   - 不是在编译 Shader 源码
   - 是在创建 PSO（和显卡驱动握手）
   - 第一次慢，后续快
   - 更新驱动后需要重新创建

---

### 核心结论
**C++ 类和 USF 的映射，是在 IMPLEMENT_MATERIAL_SHADER_TYPE 这一行"注册"确定的。**

### 🔄 完整流程

#### 宏的转发链
```
IMPLEMENT_MATERIAL_SHADER_TYPE (MaterialShaderType.h line 13)
    ↓ 转发到
IMPLEMENT_SHADER_TYPE (Shader.h line 1724)
    ↓ 创建
StaticType (包含 SourceFilename + FunctionName + Frequency)
    ↓ 通过
FShaderTypeRegistration (Shader.h line 1743) 注册到全局系统
```

#### 1. 宏入口：MaterialShaderType.h (line 13)

```cpp
#define IMPLEMENT_MATERIAL_SHADER_TYPE(TemplatePrefix,ShaderClass,SourceFilename,FunctionName,Frequency) \
    IMPLEMENT_SHADER_TYPE( \
        TemplatePrefix, \
        ShaderClass, \
        SourceFilename, \
        FunctionName, \
        Frequency \
    );
```

#### 2. 核心实现：IMPLEMENT_SHADER_TYPE (Shader.h line 1724)

这个宏做了两件关键的事：

**① 创建 StaticType**：把 Shader 元数据固化
```cpp
static ShaderClass::ShaderMetaType StaticType(
    ShaderClass::StaticGetTypeLayout(),
    TEXT(#ShaderClass),           // 类名
    SourceFilename,                // USF 文件路径 ← 关键！
    FunctionName,                  // 入口函数名 ← 关键！
    Frequency,                     // SF_Vertex/SF_Pixel
    // ... 其他元数据
);
```

**② 注册到全局系统**：
```cpp
FShaderTypeRegistration ShaderClass::ShaderTypeRegistration{
    TFunctionRef<::FShaderType&()>{ShaderClass::GetStaticType}
};
```

**FShaderTypeRegistration 的作用：**
- 延迟登记：先把各个 Shader 的 `GetStaticType()` 收集起来
- 统一注册：引擎启动时统一构造所有 Shader 类型
- 供后续使用：编译流程和运行时查找都依赖这个注册表

#### 3. 编译时使用：MaterialShader.cpp (line 1865)

```cpp
static void PrepareMaterialShaderCompileJob(/* ... */)
{
    const FMaterialShaderType* ShaderType = Key.ShaderType->AsMaterialShaderType();

    // 从 StaticType 取回注册时保存的信息
    ::GlobalBeginCompileShader(
        // ...
        ShaderType->GetShaderFilename(),  // 取回 USF 路径
        ShaderType->GetFunctionName(),    // 取回入口函数
        FShaderTarget(ShaderType->GetFrequency(), Platform),
        // ...
    );
}
```

### 📊 完整数据流向

```
【编译时 - Cook 阶段】
┌─────────────────────────────────────────┐
│ 1. 引擎启动                              │
│    └─> 扫描所有 IMPLEMENT_xxx 宏        │
│        └─> FShaderTypeRegistration      │
│            收集所有 Shader 类型          │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│ 2. Shader 编译                           │
│    └─> PrepareMaterialShaderCompileJob  │
│        ├─> GetShaderFilename()          │
│        │   → "/Engine/Private/xxx.usf"  │
│        ├─> GetFunctionName()            │
│        │   → "Main"                      │
│        └─> GlobalBeginCompileShader     │
│            └─> 编译 USF → GPU 字节码    │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│ 3. 保存到 ShaderCache                    │
│    Key: ShaderType 唯一 ID               │
│    Value: GPU 字节码                     │
└─────────────────────────────────────────┘

【运行时 - 游戏运行】
┌─────────────────────────────────────────┐
│ 1. PassProcessor 需要 Shader            │
│    └─> MaterialResource.TryGetShaders() │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│ 2. 查找全局 Shader Map                   │
│    └─> 根据 ShaderType ID 找字节码      │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│ 3. 创建 Shader 实例                      │
│    └─> new TDepthOnlyVS(Initializer)    │
│        └─> 构造函数中 Bind 参数          │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│ 4. 使用 Shader                           │
│    └─> BuildMeshDrawCommands()          │
│        └─> ShaderBindings.Add(参数)     │
│            └─> 提交到 GPU                │
└─────────────────────────────────────────┘
```

### 🎯 自测三问

1. **Q: 如果只写 DECLARE_SHADER_TYPE，不写 IMPLEMENT_MATERIAL_SHADER_TYPE？**
   - A: **链接错误**！StaticType 没有定义，GetStaticType() 找不到实现。

2. **Q: USF 文件路径写错了？**
   - A: **编译时报错**："找不到 Shader 源文件"。

3. **Q: 入口函数名写错了？**
   - A: **编译时报错**："找不到入口函数 Main"。

### 💡 类比理解

把这两个宏想象成"身份证系统"：

- **DECLARE_SHADER_TYPE**：申请身份证（声明"我需要身份"）
- **IMPLEMENT_MATERIAL_SHADER_TYPE**：办理身份证（填写详细信息）
  - 姓名：TDepthOnlyVS
  - 住址：/Engine/Private/DepthOnlyVertexShader.usf
  - 职业：Vertex Shader (SF_Vertex)
  - 工作入口：Main 函数
- **FShaderTypeRegistration**：户籍管理系统（统一登记）
- **ShaderCache**：档案库（存储编译结果）
- **TryGetShaders**：查档案（运行时查找）

---

## 2️⃣ 参数传递机制

### 核心流程
```
C++ 变量 → Bind() 绑定 → ShaderBindings.Add() 设置 → GPU Constant Buffer → Shader 参数
```

### 关键文件
- [DebugViewModeRendering.h](/d/UnrealEngine/ue5.7.1/UnrealEngine/Engine/Source/Runtime/Renderer/Private/DebugViewModeRendering.h)
- [DebugViewModeRendering.cpp](/d/UnrealEngine/ue5.7.1/UnrealEngine/Engine/Source/Runtime/Renderer/Private/DebugViewModeRendering.cpp) (第 580-739 行)

### 三步走

#### 步骤 1: 声明参数（.h 文件）

```cpp
class FDebugViewModePS : public FMeshMaterialShader
{
    DECLARE_SHADER_TYPE(FDebugViewModePS, MeshMaterial);

private:
    // 声明 Shader 参数
    LAYOUT_FIELD(FShaderParameter, OneOverCPUTexCoordScalesParameter)
    LAYOUT_FIELD(FShaderParameter, CPUTexelFactorParameter)
    LAYOUT_FIELD(FShaderParameter, NormalizedComplexity)
};
```

#### 步骤 2: 绑定参数（构造函数）

```cpp
FDebugViewModePS(const ShaderMetaType::CompiledShaderInitializerType& Initializer)
    : FMeshMaterialShader(Initializer)
{
    // 将 C++ 变量绑定到 Shader 参数名
    OneOverCPUTexCoordScalesParameter.Bind(Initializer.ParameterMap, TEXT("OneOverCPUTexCoordScales"));
    CPUTexelFactorParameter.Bind(Initializer.ParameterMap, TEXT("CPUTexelFactor"));
    NormalizedComplexity.Bind(Initializer.ParameterMap, TEXT("NormalizedComplexity"));
}
```

**关键点：**
- `ParameterMap`：Shader 编译后生成的参数映射表
- `TEXT("参数名")`：**必须与 USF 文件中的参数名完全一致**

#### 步骤 3: 设置参数值（GetShaderBindings）

**位置：** [DebugViewModeRendering.cpp](/d/UnrealEngine/ue5.7.1/UnrealEngine/Engine/Source/Runtime/Renderer/Private/DebugViewModeRendering.cpp) (第 727-739 行)

```cpp
void GetDebugViewModeShaderBindings(
    const FDebugViewModePS& Shader,
    FMeshDrawSingleShaderBindings& ShaderBindings) const
{
    // 1. 准备 C++ 数据
    FVector4f OneOverCPUTexCoordScales[4];
    FVector4f NormalizedComplexityValue = FVector4f(1.0f, 2.0f, 3.0f, 4.0f);
    FVector4 WorldUVDensities = FVector4(10.0, 20.0, 30.0, 40.0);

    // 2. 设置参数 - 关键代码！
    ShaderBindings.Add(Shader.OneOverCPUTexCoordScalesParameter, OneOverCPUTexCoordScales);
    ShaderBindings.Add(Shader.CPUTexelFactorParameter, FVector4f(WorldUVDensities));
    ShaderBindings.Add(Shader.NormalizedComplexity, NormalizedComplexityValue);
}
```

### 🎯 ShaderBindings.Add() 用法

```cpp
// 语法
ShaderBindings.Add(Shader参数变量, C++数据值);

// 支持的数据类型
ShaderBindings.Add(Shader.VectorParam, FVector3f(1.0f, 2.0f, 3.0f));  // → float3
ShaderBindings.Add(Shader.FloatParam, 1.5f);                           // → float
ShaderBindings.Add(Shader.IntParam, 42);                               // → int
ShaderBindings.Add(Shader.Vector4Param, FVector4f(1,2,3,4));          // → float4
ShaderBindings.Add(Shader.ArrayParam, MyArray);                        // → 数组
```

### 📊 数据流向

```
┌──────────────┐
│ C++ 变量      │  FVector3f MyPos(1, 2, 3);
└──────┬───────┘
       ↓
┌──────────────┐
│ Bind()       │  MyPosParam.Bind(ParameterMap, TEXT("MyPos"));
└──────┬───────┘  建立 C++ 变量 ↔ Shader 参数名的映射
       ↓
┌──────────────┐
│ Add()        │  ShaderBindings.Add(Shader.MyPosParam, MyPos);
└──────┬───────┘  设置实际数据值
       ↓
┌──────────────┐
│ GPU CB       │  上传到 GPU Constant Buffer
└──────┬───────┘
       ↓
┌──────────────┐
│ Shader 参数  │  float3 MyPos; // 在 USF 中使用
└──────────────┘
```

### 💡 理解要点

1. **Bind() 是"建立映射"**：告诉系统"C++ 的这个变量对应 Shader 的那个参数"
2. **Add() 是"设置数据"**：把实际的数值传给 GPU
3. **参数名必须一致**：C++ 的 `TEXT("MyPos")` 必须和 USF 中的 `float3 MyPos` 名字相同

---

## 3️⃣ 图元类型

查找 `FMeshDrawCommand.PrimitiveType`，了解 `PT_TriangleList`, `PT_LineList`, `PT_PointList` 的区别。

### 关键文件
- [RHIDefinitions.h](/d/UnrealEngine/ue5.7.1/UnrealEngine/Engine/Source/Runtime/RHI/Public/RHIDefinitions.h) - EPrimitiveType 定义

（待补充...）
