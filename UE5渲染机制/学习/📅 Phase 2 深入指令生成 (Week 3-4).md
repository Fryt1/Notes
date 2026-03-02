目标： 介入渲染管线的“决策层”，控制“画什么”和“怎么画”。

# 学习

## 查看 BasePassRendering.cpp

- [x] **死磕核心：** 阅读 `Engine/Source/Runtime/Renderer/Private/BasePassRendering.cpp`。

	- [x] **搜索关键词：** `AddMeshBatch`。这是所有不透明物体渲染逻辑的入口。

## 第一层：材质解析
核心函数==AddMeshBatch==

```
void FBasePassMeshProcessor::AddMeshBatch(...)
{
    if (MeshBatch.bUseForMaterial)  // ① 检查 MeshBatch 是否用于材质渲染
    {
        const FMaterialRenderProxy* MaterialRenderProxy = MeshBatch.MaterialRenderProxy;
        while (MaterialRenderProxy)  // ② 材质降级循环 (Fallback)
        {
            const FMaterial* Material = MaterialRenderProxy->GetMaterialNoFallback(FeatureLevel);
            if (Material && Material->GetRenderingThreadShaderMap())  // ③ 材质合法 + Shader 已编译
            {
                if (TryAddMeshBatch(...))  // ④ 进入第二层
                    break;
            }
            MaterialRenderProxy = MaterialRenderProxy->GetFallback(FeatureLevel);  // ⑤ 拿不到就降级
        }
    }
}
```

**关键决策**：材质有 Fallback 链，如果你的自定义材质 Shader 没编译好，它会自动降级到默认材质。

## 第二层：混合模式过滤
核心函数==TryAddMeshBatch==
bool FBasePassMeshProcessor::TryAddMeshBatch(...)
{
    const EBlendMode BlendMode = Material.GetBlendMode();
    const bool bIsTranslucent = IsTranslucentBlendMode(BlendMode);

    // ShouldDraw 的核心逻辑:
    // - 如果是 bTranslucentBasePass → 只画对应半透明子 Pass 的
    // - 如果不是 → bShouldDraw = !bIsTranslucent (只画不透明的！)

    if (bShouldDraw
        && PrimitiveSceneProxy->ShouldRenderInMainPass()  // Proxy 是否参与主 Pass
        && ShouldIncludeDomainInMeshPass(Material)        // 材质 Domain 检查
        && ShouldIncludeMaterialInDefaultOpaquePass(Material))  // 默认 Opaque Pass 检查
    {
        // → 进入第三层
    }
}

## 第三层：光照策略选择 + 调用 Process

### 好的，第三层代码在 ==TryAddMeshBatch==，以下是带注释的完整逻辑：

```
// === 第三层：光照策略选择 + 调用 Process (L2217-2325) ===
// 前面两层过滤都通过了，现在决定"怎么画" —— 选光照策略

// ① 先判断：是不是 "有光照 + 半透明 + 能投体积自阴影" 的特殊情况
if (bIsLitMaterial
    && bIsTranslucent
    && PrimitiveSceneProxy
    && PrimitiveSceneProxy->CastsVolumetricTranslucentShadow())
{
    // 特殊路径：半透明自阴影，再细分三种 Policy
    
    if (bUseVolumetricLightmap && bAllowStaticLighting)
    {
        // 情况A: 用体积光照贴图 + 自阴影
        Process<FSelfShadowedVolumetricLightmapPolicy>(...);
    }
    else if (IsIndirectLightingCacheAllowed() && bAllowIndirectLightingCache)
    {
        // 情况B: 用间接光照缓存 + 自阴影
        Process<FSelfShadowedCachedPointIndirectLightingPolicy>(...);
    }
    else
    {
        // 情况C: 普通自阴影半透明
        Process<FSelfShadowedTranslucencyPolicy>(...);
    }
}
else
{
    // ② 常规路径（90%+ 的物体走这里）
    //    通过 GetUniformLightMapPolicyType 选择光照贴图策略
    ELightMapPolicyType UniformLightMapPolicyType = 
        GetUniformLightMapPolicyType(FeatureLevel, Scene, MeshBatch.LCI, 
                                      PrimitiveSceneProxy, Material);
    
    // 选出来的 Policy 可能是:
    //   LMP_HQ_LIGHTMAP                              → 高质量光照贴图
    //   LMP_LQ_LIGHTMAP                              → 低质量光照贴图
    //   LMP_DISTANCE_FIELD_SHADOWS_AND_HQ_LIGHTMAP   → 距离场阴影 + HQ
    //   LMP_PRECOMPUTED_IRRADIANCE_VOLUME            → 预计算辐照度体积
    //   LMP_CACHED_VOLUME_INDIRECT_LIGHTING          → 缓存体积间接光
    //   LMP_CACHED_POINT_INDIRECT_LIGHTING           → 缓存点间接光
    //   LMP_NO_LIGHTMAP                              → 无光照贴图
    
    Process<FUniformLightMapPolicy>(
        MeshBatch,
        BatchElementMask,
        StaticMeshId,
        PrimitiveSceneProxy,
        MaterialRenderProxy,
        Material,
        bIsMasked,
        bIsTranslucent,
        ShadingModels,
        FUniformLightMapPolicy(UniformLightMapPolicyType),  // ← 策略注入
        MeshBatch.LCI,        // ← 光照缓存接口
        MeshFillMode,
        MeshCullMode);
}
```

### ==Process== 内部做的事
调用==BuildMeshDrawCommands==

```
template<typename LightMapPolicyType>
bool FBasePassMeshProcessor::Process(...)
{
    // 1 根据 Material + VertexFactory + LightMapPolicy 获取 Shader
    TMeshProcessorShaders<VS, PS> BasePassShaders;
    if (!GetBasePassShaders<LightMapPolicyType>(
            MaterialResource,
            VertexFactory->GetType(),
            LightMapPolicy,          // ← 上一步选出的光照策略
            FeatureLevel,
            bRenderSkylight,
            &BasePassShaders.VertexShader,    // ← 输出: 选好的 VS
            &BasePassShaders.PixelShader))    // ← 输出: 选好的 PS
    {
        return false;  // Shader 编译失败 → 放弃
    }

    // 2 配置渲染状态
    FMeshPassProcessorRenderState DrawRenderState(PassDrawRenderState);
    SetDepthStencilStateForBasePass(DrawRenderState, ...);  // 深度/模板
    // 如果是半透明，还要设置半透明混合状态
    if (bTranslucentBasePass)
        SetTranslucentRenderState(DrawRenderState, ...);

    // 3 计算排序键（决定绘制顺序）
    FMeshDrawCommandSortKey SortKey;
    if (bTranslucentBasePass)
        SortKey = CalculateTranslucentMeshStaticSortKey(...);  // 半透明: 按距离排
    else
        SortKey = CalculateBasePassMeshStaticSortKey(          // 不透明: 按材质/Shader 排
            EarlyZPassMode, bIsMasked, VS, PS);

    // 4 最终调用！把所有决策结果交给 BuildMeshDrawCommands
    BuildMeshDrawCommands(
        MeshBatch,
        BatchElementMask,
        PrimitiveSceneProxy,
        MaterialRenderProxy,
        MaterialResource,
        DrawRenderState,        // ← PSO 状态 (深度/混合/模板)
        BasePassShaders,        // ← 选好的 VS + PS
        MeshFillMode,           // ← 实体/线框
        MeshCullMode,           // ← 背面剔除
        SortKey,                // ← 绘制排序键
        EMeshPassFeatures::Default,
        ShaderElementData);     // ← 光照贴图数据
    
    return true;
}
```

## **RenderDoc 逆向：**
- [x] **RenderDoc 逆向：** 截帧查看 Mesh 的 `Vertex Attributes`，回到源码寻找 `FMeshDrawCommand` 是在哪填充这些 Stream 的。
截帧查看 Mesh 的 **`Vertex Attributes`**，回到源码寻找 **`FMeshDrawCommand`** 是在哪填充这些 Stream 的。

## 将 MeshBatch 处理成 MeshDrawCommands 的流程

```
// 在你的 FRealityDistortionPassProcessor::AddMeshBatch 中
void FRealityDistortionPassProcessor::AddMeshBatch(...)
{
    // 1. "画什么" - 过滤逻辑（你已经做了）
    if (!ShouldRenderInDistortionField(PrimitiveSceneProxy))
        return;
    
    // 2. "怎么画" - 调用 BuildMeshDrawCommands
    //    现在你理解了这个函数内部：
    //    - 从 VertexFactory 拿 VertexStreams
    //    - 绑定你的自定义 Shader
    //    - 设置你的 Distortion 效果参数
    BuildMeshDrawCommands(
        MeshBatch,
        BatchElementMask,
        PrimitiveSceneProxy,
        MaterialRenderProxy,
        MaterialResource,
        PassDrawRenderState,
        DistortionPassShaders,  // 你的自定义 Shader
        MeshFillMode,
        MeshCullMode,
        SortKey,
        EMeshPassFeatures::Default,
        ShaderElementData  // 可以传自定义数据给 Shader
    );
}
```

## 立方体的 drawcall
![[Pasted image 20260126205004.png]]
![[Pasted image 20260126204939.png]]

### input assembler
![[Pasted image 20260126204927.png]]

### Input Layout
![[Pasted image 20260126205624.png]]

#### PER_VERTEX 和 PER_INSTANCE
可以看到有两个槽位的数据

| 字段             | Slot 0 值        | Slot 10 值    | 含义                                                                                                      |
| -------------- | --------------- | ------------ | ------------------------------------------------------------------------------------------------------- |
| **Slot**       | 0               | 10           | 属性编号 (对应 Shader 中的 <br><br>```<br>ATTRIBUTE0<br>```<br><br>, <br><br>```<br>ATTRIBUTE10<br>```<br><br>) |
| **Semantic**   | ATTRIBUTE       | ATTRIBUTE    | 语义名 (D3D 用，Vulkan/OpenGL 通常都是 ATTRIBUTE)                                                                |
| **Index**      | 0               | 13           | 语义索引                                                                                                    |
| **Format**     | R32G32B32_FLOAT | R32_UINT     | 数据格式：float3 (12 bytes) / uint (4 bytes)                                                                 |
| **Input Slot** | 0               | 5            | 🔴 绑定到哪个 **Buffer Slot**                                                                                |
| **Offset**     | 0               | 0            | 在 Buffer 中的起始偏移                                                                                         |
| **Class**      | PER_VERTEX      | PER_INSTANCE | 🔵 每顶点读取 / 每实例读取                                                                                        |
| **Step Rate**  | -               | 1            | PER_INSTANCE 时，每 N 个实例更新一次                                                                              |
里面有语义，对应的 buffer 槽位，数据类型等等

### Buffer
![[Pasted image 20260126205637.png]]

从截图中看到 3 个 Buffer：

| Slot      | Buffer 名称                   | Stride | Offset  | Byte Length | 含义                          |
| --------- | --------------------------- | ------ | ------- | ----------- | --------------------------- |
| **Index** | Resource PoolAllocator...   | 2      | 9656579 | 90          | 索引缓冲区 (EBO), 45 个 uint16 索引 |
| **0**     | Resource PoolAllocator...   | 12     | 9657344 | 288         | 位置 VBO, 24 个 float3 顶点      |
| **5**     | InstanceCulling.Instance... | 0      | 28      | 65508       | 实例 ID 偏移缓冲区                 |

##### InstanceIdOffsetBuffer 实际读取 GPUScene 数据

```
InstanceIdOffsetBuffer = [0, 1, 2, 3, ..., 99]
                          │  │  │
                          │  │  └── 实例2 → 读取 GPU Scene[2]
                          │  └───── 实例1 → 读取 GPU Scene[1]
                          └──────── 实例0 → 读取 GPU Scene[0]
```

### **GPUScene**

GPU Scene Instance Data[N]:
├── LocalToWorld (float4x4)      ← Transform 矩阵
├── PrevLocalToWorld (float4x4)  ← 上一帧 Transform (用于 Motion Blur)
├── LocalBounds (float4)         ← 包围盒
├── CustomPrimitiveData          ← 自定义数据 (材质参数等)
├── LightmapUVBias               ← 光照贴图 UV 偏移
├── ... 更多属性

### Draw Call 完整流程
**场景**: 绘制 100 个椅子实例

```
for each Instance (0..99):// ← 实际上是 GPU 并行处理所有实例

① 从 InstanceIdOffsetBuffer[InstanceID] 获取 GPU Scene 偏移

uint offset = InstanceIdOffsetBuffer[InstanceID];

② 从 GPU Scene 读取这个实例的所有数据

InstanceData data = GPUScene[offset];

float4x4 WorldMatrix = data.LocalToWorld;

float4 CustomData = data.CustomPrimitiveData;

③ 遍历 EBO 中的索引，绘制每个三角形

for each triangle in EBO:

for each vertex (idx0, idx1, idx2):

④ 从 VBO 读取顶点的本地坐标

float3 LocalPos = PositionVBO[idx];

⑤ 用实例的 Transform 变换到世界空间

float3 WorldPos = LocalPos * WorldMatrix;

⑥ 输出到光栅化
```

## RenderDoc ↔ CPU 代码完整对应关系

本笔记记录 RenderDoc 截帧看到的 GPU 数据如何对应到 UE5 CPU 端源代码，并附带简要功能说明。

---

## 1. Vertex Input (VBO 绑定)

**核心功能**：告诉 GPU 顶点数据（位置、UV、法线等）存放在哪块显存，以及如何读取。

| RenderDoc 看到的       | CPU 代码位置                                | 功能简述                                  |
| ------------------- | --------------------------------------- | ------------------------------------- |
| VBO Slot 0,1,2,3... | FVertexFactory::GetStreams              | **配置数据流**：决定当前 Mesh 需要用到哪些顶点缓冲区（VBO）。 |
| VertexBuffer RHI 资源 | FVertexInputStream::SetOnRHICommandList | **绑定显存**：将具体的 VBO 资源句柄绑定到 GPU 的指定槽位。  |
| Stride / Offset     | FVertexStream 中的 `Stream.Offset`        | **数据偏移**：告诉 GPU 从 VBO 的哪个字节位置开始读取数据。  |

### 关键代码片段

```
// VertexFactory.cpp:256-288

void FVertexFactory::GetStreams(...) const

{

    // 遍历所有流，收集 VBO 和偏移信息，准备绑定

    for (int32 StreamIndex = 0; StreamIndex < Streams.Num(); StreamIndex++)

    {

        const FVertexStream& Stream = Streams[StreamIndex];

        OutVertexStreams.Add(FVertexInputStream(StreamIndex, Stream.Offset, 

                                                Stream.VertexBuffer->VertexBufferRHI));

    }

}
```

---

## 2. Pipeline State (PSO)

**核心功能**：配置光栅化管线状态，决定三角形如何被绘制、着色和混合。

|RenderDoc 看到的|CPU 代码位置|功能简述|
|---|---|---|
|Vertex/Pixel Shader|BuildMeshDrawCommands - SetupBoundShaderState|**绑定 Shader**：设置当前使用的顶点与像素着色器程序。|
|RasterizerState|BuildMeshDrawCommands - GetStaticRasterizerState|**光栅化设置**：控制背面剔除（Cull Mode）和填充模式（实体/线框）。|
|BlendState|BuildMeshDrawCommands - GetBlendState|**混合模式**：控制像素如何与背景混合（如透明度叠加）。|
|DepthStencilState|BuildMeshDrawCommands - GetDepthStencilState|**深度测试**：控制 Z-Buffer 读写，决定遮挡关系。|
|VertexDeclaration (Input Layout)|BuildMeshDrawCommands - GetDeclaration|**数据格式**：定义顶点数据的布局（如：Float3 位置 + Byte4 颜色）。|

### 关键代码片段

```
// MeshPassProcessor.inl:83-108

// 设置 PSO 状态

PipelineState.SetupBoundShaderState(VertexDeclaration, MeshProcessorShaders); // Shader + Input Layout

PipelineState.RasterizerState = GetStaticRasterizerState<true>(MeshFillMode, MeshCullMode); // 剔除

PipelineState.BlendState = DrawRenderState.GetBlendState();       // 混合

PipelineState.DepthStencilState = DrawRenderState.GetDepthStencilState(); // 深度

---
```

## 3. Shader Bindings (Uniform/Texture/Sampler)

**核心功能**：给 Shader 提供运行时参数（位置、颜色、纹理贴图）。

| RenderDoc 看到的              | CPU 代码位置                                  | 功能简述                                                            |
| -------------------------- | ----------------------------------------- | --------------------------------------------------------------- |
| CBV (Constant Buffer)      | BuildMeshDrawCommands - GetShaderBindings | **常量参数**：收集 Shader 需要的 Uniform 变量（如 ObjectPosition, BaseColor）。 |
| SRV (Shader Resource View) | 同上，在 `ShaderBindings` 中                   | **纹理资源**：收集 Shader 需要采样的纹理贴图。                                   |
| Sampler                    | 同上，在 `ShaderBindings` 中                   | **采样器**：定义纹理的过滤方式（如 Linear/Nearest）和寻址模式（Wrap/Clamp）。           |

### 关键代码片段

```
// MeshPassProcessor.inl:136-152

// 让 Shader 从 Scene、Proxy 和 Material 中获取它需要的具体数据绑定

PassShaders.VertexShader->GetShaderBindings(Scene, FeatureLevel, PrimitiveSceneProxy, 

                                             MaterialRenderProxy, MaterialResource, 

                                             ShaderElementData, ShaderBindings);
```

---

## 4. Draw Call 提交流程

**核心功能**：将准备好的命令发送给 GPU 执行绘制。

|RenderDoc 看到的|CPU 代码位置|功能简述|
|---|---|---|
|SetStreamSource|SubmitDrawBegin - SetStreamSource|**设置顶点源**：底层 API 调用，应用 VBO 绑定。|
|SetGraphicsPipelineState|SubmitDrawBegin - SetGraphicsPipelineStateCheckApply|**设置管线**：底层 API 调用，应用 PSO（Shader 和渲染状态）。|
|SetShaderBindings|SubmitDrawBegin - SetOnCommandList|**应用参数**：底层 API 调用，将参数和纹理表提交到 GPU。|
|DrawIndexedPrimitive|SubmitDrawEnd - DrawIndexedPrimitive|**绘制指令**：最终命令，让 GPU 根据索引缓冲区绘制三角形。|

### 关键代码片段

```
// MeshPassProcessor.cpp:1281-1297 (SubmitDrawBegin)

Stream.SetOnRHICommandList(RHICmdList);  // RHI: 绑定 VBO

// MeshPassProcessor.cpp:1320-1328 (SubmitDrawEnd)

RHICmdList.DrawIndexedPrimitive(MeshDrawCommand.IndexBuffer, ...); // RHI: 开始绘制
```

---

## 5. 完整调用链路

```mermaid
flowchart TD
    subgraph Phase1["Phase 1: 数据源"]
        A["UStaticMesh::GetRenderData()"] --> B["FStaticMeshLODResources"]
        B --> C["VertexBuffers\n(Position, Tangent, UV...)"]
        B --> D["IndexBuffer"]
        B --> E["FLocalVertexFactory"]
    end

    subgraph Phase2["Phase 2: FMeshBatch 收集"]
        F["GetDynamicMeshElements"] --> G["FMeshBatch"]
        G --> H["VertexFactory 指针"]
        G --> I["MaterialRenderProxy"]
        G --> J["IndexBuffer"]
    end

    subgraph Phase3["Phase 3: BuildMeshDrawCommands
    "]
        K["AddMeshBatch"] --> L["BuildMeshDrawCommands"]
        L --> M["GetStreams → VertexStreams"]
        L --> N["GetShaderBindings → ShaderBindings"]
        L --> O["PSO 状态配置"]
    end

    subgraph Phase4["Phase 4: 提交到 RHI"]
        P["SubmitMeshDrawCommandsRange"] --> Q["SubmitDrawBegin"]
        Q --> R["SetStreamSource (VBO)"]
        Q --> S["SetGraphicsPipelineState"]
        Q --> T["SetOnCommandList (Bindings)"]
        P --> U["SubmitDrawEnd"]
        U --> V["DrawIndexedPrimitive"]
    end

    Phase1 --> Phase2
    Phase2 --> Phase3
    Phase3 --> Phase4
```

---

## 6.  指令构建 (工作线程)
此阶段展示 **`MeshPassProcessor`** 如何将 **`FMeshBatch`** 转换为 GPU 可识别的

FMeshDrawCommand。

```mermaid
graph TD
    classDef rt fill:#fff9c4,stroke:#fbc02d,color:#000
    classDef build fill:#e8f5e9,stroke:#2e7d32,color:#000

    MeshBatch["FMeshBatch"]:::rt
    
    Processor["MeshPassProcessor"]:::build
    Func_BMDC("BuildMeshDrawCommands()"):::build
    
    PSO_Setup["PSO 设置<br/>Shader/光栅化/混合"]:::build
    Streams["顶点流<br/>VBO 绑定信息"]:::build
    Bindings["Shader 绑定<br/>Uniforms/纹理"]:::build
    
    MDC["FMeshDrawCommand<br/>(GPU 指令缓存)"]:::build

    Processor -- "调用" --> Func_BMDC
    Func_BMDC -- "输入" --> MeshBatch
    
    Func_BMDC -- "调用" --> PSO_Setup
    Func_BMDC -- "获取流" --> Streams
    Func_BMDC -- "获取绑定" --> Bindings
    
    PSO_Setup --> MDC
    Streams --> MDC
    Bindings --> MDC
```

## 7: 提交 (RHI 线程)
此阶段展示 RHI 线程如何执行最终的 GPU 指令

```mermaid
graph TD
    classDef build fill:#e8f5e9,stroke:#2e7d32,color:#000
    classDef rhi fill:#ffebee,stroke:#c62828,color:#000

    MDC["FMeshDrawCommand"]:::build

    Submit("SubmitDrawBegin"):::rhi
    CmdList["RHICmdList"]:::rhi
    
    Op_Streams["SetStreamSource (设置 VBO)"]:::rhi
    Op_PSO["SetGraphicsPipelineState (设置 PSO)"]:::rhi
    Op_Bind["SetShaderBindings (设置参数/纹理)"]:::rhi
    Op_Draw["DrawIndexedPrimitive (执行绘制)"]:::rhi

    MDC -- "读取者" --> Submit
    Submit -- "执行于" --> CmdList
    
    CmdList --> Op_Streams
    CmdList --> Op_PSO
    CmdList --> Op_Bind
    CmdList --> Op_Draw
```

## 8. RenderDoc 组件映射图
此图展示 RenderDoc 中观察到的 GPU 状态是由哪些 C++ 类负责管理的。

```mermaid
graph TB
    classDef gpu fill:#ffccbc,stroke:#bf360c,color:#000
    classDef cpu fill:#bbdefb,stroke:#0d47a1,color:#000

    subgraph CPU_Classes ["UE5 C++ 类"]
        VF["FVertexFactory<br/>(GetStreams)"]:::cpu
        MPP["FMeshPassProcessor<br/>(BuildMeshDrawCommands)"]:::cpu
        MMS["FMeshMaterialShader<br/>(GetShaderBindings)"]:::cpu
    end

    subgraph GPU_State ["RenderDoc 观察到的 GPU 状态"]
        VI["Vertex Input (VBOs 顶点输入)"]:::gpu
        VS["Vertex Shader (顶点着色器)"]:::gpu
        PS["Pixel Shader (像素着色器)"]:::gpu
        RS["Rasterizer State (光栅化状态)"]:::gpu
        BS["Blend State (混合状态)"]:::gpu
        CBV["Constant Buffers (CBV 常量缓冲)"]:::gpu
        SRV["Textures (SRV 纹理资源)"]:::gpu
    end
    
    VF -. "定义" .-> VI
    MPP -. "选择" .-> VS
    MPP -. "选择" .-> PS
    MPP -. "配置" .-> RS
    MPP -. "配置" .-> BS
    MMS -. "填充" .-> CBV
    MMS -. "填充" .-> SRV
```

## 关键函数解析

### 1. Vertex Input (VBO / 顶点输入)

- **负责人**: `FVertexFactory::GetStreams`
- **动作**: 遍历流并创建 `FVertexInputStream` 条目。
- **结果**: 告诉 GPU “数据流 0 位在这个缓冲区的 0 偏移处”。

### 2. Pipeline State (PSO / 管线状态)

- **负责人**: `FMeshPassProcessor::BuildMeshDrawCommands`
- **动作**: 调用 `SetupBoundShaderState` 并设置 `RasterizerState` (光栅化), `BlendState` (混合)。
- **结果**: 编译“如何绘制”的状态对象 (Shader 程序 + 固定管线设置)。

### 3. Shader Bindings (着色器绑定)

- **负责人**: `FMeshPassProcessor` -> `GetShaderBindings`
- **动作**: 从

    Scene (场景),

    View (视图), 和

    Material (材质) 中拉取数据来填充 Uniform Buffers (常量缓冲区)。
- **结果**: 提供“用什么数据绘制” (颜色, 变换矩阵, 纹理 ID)。

### 4. 补充深挖：FMeshBatch 如何进入 MeshPassProcessor？

很多开发者会好奇，`GetDynamicMeshElements` 只是生成了 `FMeshBatch`，是谁把它“喂”给 `MeshPassProcessor` 处理的？答案是由两个不同的路径汇聚到同一个入口函数 `AddMeshBatch`。

#### 核心机制

无论静态还是动态物体，最终都会调用 `FMeshPassProcessor::AddMeshBatch` 函数。

1. **静态路径 (Static Path)**

    - **时机**: 场景加载或物体变动时 (Cache Update)。
    - **函数**: `FPrimitiveSceneInfo::CacheMeshDrawCommands` (在

        PrimitiveSceneInfo.cpp)。
    - **流程**: 它遍历所有

        StaticMeshes，创建对应的 Processor，然后循环调用 `AddMeshBatch`。
2. **动态路径 (Dynamic Path)**

    - **时机**: 每一帧 (Per Frame)。
    - **函数**:

        GenerateDynamicMeshDrawCommands (在

        MeshDrawCommands.cpp)。
    - **流程**: 它遍历每一帧收集到的 `DynamicMeshElements`，然后循环调用 `AddMeshBatch`。

#### 流程图解

```mermaid
graph TD
    classDef static fill:#e3f2fd,stroke:#1565c0,color:#000
    classDef dynamic fill:#ffebee,stroke:#c62828,color:#000
    classDef core fill:#e8f5e9,stroke:#2e7d32,color:#000

    subgraph StaticPath ["静态路径 (一次性/缓存)"]
        direction TB
        S1["场景加载/物体变动"]:::static
        S2["FPrimitiveSceneInfo::CacheMeshDrawCommands"]:::static
        S3["遍历 StaticMeshes 数组"]:::static
        S1 --> S2 --> S3
    end

    subgraph DynamicPath ["动态路径 (每帧)"]
        direction TB
        D1["每一帧渲染循环"]:::dynamic
        D2["GenerateDynamicMeshDrawCommands"]:::dynamic
        D3["遍历 DynamicMeshElements 数组"]:::dynamic
        D1 --> D2 --> D3
    end

    subgraph Core ["MeshPassProcessor 核心"]
        Entry["AddMeshBatch (入口)"]:::core
        Build["BuildMeshDrawCommands"]:::core
        Entry --> Build
    end

    S3 -- "调用" --> Entry
    D3 -- "调用" --> Entry
```

# 实际操作

**加一个自定义 MeshPass 需要回答三个问题：**

**这个 Pass 叫什么？ →** **MeshPassProcessor.h** **加枚举**

**哪些 Mesh 进这个 Pass？ →** **SceneVisibility.cpp** **标记 Mesh**

**进来的 Mesh 怎么画？ →  MeshPassProcessor 子类决定 Shader 和渲染状态**

**什么时候画？ → DeferredShadingRenderer::Render 里提交 DrawCommand**

## 💻 实战任务：自定义 Processor (Coding Tasks)

### 注册 Pass

#### **注册 Pass：** 在 `MeshPassProcessor.h` (或你的模块) 中注册一个新的 `EMeshPass::Type`，命名为 `DistortionPass`。

##### **改动 1**：添加 EMeshPass 枚举==RealityDistortion==

```
namespace EMeshPass

{

    enum Type : uint8

    {

        DepthPass,

        SecondStageDepthPass,

        BasePass,

        AnisotropyPass,

        SkyPass,

        SingleLayerWaterPass,

        SingleLayerWaterDepthPrepass,

        CSMShadowDepth,

        VSMShadowDepth,

        OnePassPointLightShadowDepth,

        Distortion,

        Velocity,

        TranslucentVelocity,

        TranslucentVelocityClippedDepth, /**  A pass to handle pixels with depth below the opaque threshold*/

        TranslucencyStandard,

        TranslucencyStandardModulate,

        TranslucencyAfterDOF,

        TranslucencyAfterDOFModulate,

        TranslucencyAfterMotionBlur,

        TranslucencyHoldout, /** A standalone pass to render all translucency for holdout, inferring the background visibility*/

        TranslucencyAll, /** Drawing all translucency, regardless of separate or standard.  Used when drawing translucency outside of the main renderer, eg FRendererModule::DrawTile. */

        LightmapDensity,

        DebugViewMode, /** Any of EDebugViewShaderMode */

        CustomDepth,

        MobileBasePassCSM,  /** Mobile base pass with CSM shading enabled */

        VirtualTexture,

        LumenCardCapture,

        LumenCardNanite,

        LumenTranslucencyRadianceCacheMark,

        LumenFrontLayerTranslucencyGBuffer,

        DitheredLODFadingOutMaskPass, /** A mini depth pass used to mark pixels with dithered LOD fading out. Currently only used by ray tracing shadows. */

        NaniteMeshPass,

        MeshDecal_DBuffer,

        MeshDecal_SceneColorAndGBuffer,

        MeshDecal_SceneColorAndGBufferNoNormal,

        MeshDecal_SceneColor,

        MeshDecal_AmbientOcclusion,

        WaterInfoTextureDepthPass,

        WaterInfoTexturePass,

        RealityDistortion,

  

#if WITH_EDITOR

        HitProxy,

        HitProxyOpaqueOnly,

        EditorLevelInstance,

        EditorSelection,

#endif

  

        Num,

        NumBits = 6,

    };

}
```

##### **改动 2** — 在 ==GetMeshPassName== 的 switch 中添加 case：
添加 case EMeshPass::RealityDistortion: return TEXT("RealityDistortion");

```
inline const TCHAR* GetMeshPassName(EMeshPass::Type MeshPass)

{

    switch (MeshPass)

    {

    case EMeshPass::DepthPass: return TEXT("DepthPass");

    case EMeshPass::SecondStageDepthPass: return TEXT("SecondStageDepthPass");

    case EMeshPass::BasePass: return TEXT("BasePass");

    case EMeshPass::AnisotropyPass: return TEXT("AnisotropyPass");

    case EMeshPass::SkyPass: return TEXT("SkyPass");

    case EMeshPass::SingleLayerWaterPass: return TEXT("SingleLayerWaterPass");

    case EMeshPass::SingleLayerWaterDepthPrepass: return TEXT("SingleLayerWaterDepthPrepass");

    case EMeshPass::CSMShadowDepth: return TEXT("CSMShadowDepth");

    case EMeshPass::VSMShadowDepth: return TEXT("VSMShadowDepth");

    case EMeshPass::OnePassPointLightShadowDepth: return TEXT("OnePassPointLightShadowDepth");

    case EMeshPass::Distortion: return TEXT("Distortion");

    case EMeshPass::Velocity: return TEXT("Velocity");

    case EMeshPass::TranslucentVelocity: return TEXT("TranslucentVelocity");

    case EMeshPass::TranslucentVelocityClippedDepth: return TEXT("TranslucentVelocityClippedDepth");

    case EMeshPass::TranslucencyStandard: return TEXT("TranslucencyStandard");

    case EMeshPass::TranslucencyStandardModulate: return TEXT("TranslucencyStandardModulate");

    case EMeshPass::TranslucencyAfterDOF: return TEXT("TranslucencyAfterDOF");

    case EMeshPass::TranslucencyAfterDOFModulate: return TEXT("TranslucencyAfterDOFModulate");

    case EMeshPass::TranslucencyAfterMotionBlur: return TEXT("TranslucencyAfterMotionBlur");

    case EMeshPass::TranslucencyHoldout: return TEXT("TranslucencyHoldout");

    case EMeshPass::TranslucencyAll: return TEXT("TranslucencyAll");

    case EMeshPass::LightmapDensity: return TEXT("LightmapDensity");

    case EMeshPass::DebugViewMode: return TEXT("DebugViewMode");

    case EMeshPass::CustomDepth: return TEXT("CustomDepth");

    case EMeshPass::MobileBasePassCSM: return TEXT("MobileBasePassCSM");

    case EMeshPass::VirtualTexture: return TEXT("VirtualTexture");

    case EMeshPass::LumenCardCapture: return TEXT("LumenCardCapture");

    case EMeshPass::LumenCardNanite: return TEXT("LumenCardNanite");

    case EMeshPass::LumenTranslucencyRadianceCacheMark: return TEXT("LumenTranslucencyRadianceCacheMark");

    case EMeshPass::LumenFrontLayerTranslucencyGBuffer: return TEXT("LumenFrontLayerTranslucencyGBuffer");

    case EMeshPass::DitheredLODFadingOutMaskPass: return TEXT("DitheredLODFadingOutMaskPass");

    case EMeshPass::NaniteMeshPass: return TEXT("NaniteMeshPass");

    case EMeshPass::MeshDecal_DBuffer: return TEXT("MeshDecal_DBuffer");

    case EMeshPass::MeshDecal_SceneColorAndGBuffer: return TEXT("MeshDecal_SceneColorAndGBuffer");

    case EMeshPass::MeshDecal_SceneColorAndGBufferNoNormal: return TEXT("MeshDecal_SceneColorAndGBufferNoNormal");

    case EMeshPass::MeshDecal_SceneColor: return TEXT("MeshDecal_SceneColor");

    case EMeshPass::MeshDecal_AmbientOcclusion: return TEXT("MeshDecal_AmbientOcclusion");

    case EMeshPass::WaterInfoTextureDepthPass: return TEXT("WaterInfoTextureDepthPass");

    case EMeshPass::WaterInfoTexturePass: return TEXT("WaterInfoTexturePass");

    case EMeshPass::RealityDistortion: return TEXT("RealityDistortion");

#if WITH_EDITOR

    case EMeshPass::HitProxy: return TEXT("HitProxy");

    case EMeshPass::HitProxyOpaqueOnly: return TEXT("HitProxyOpaqueOnly");

    case EMeshPass::EditorLevelInstance: return TEXT("EditorLevelInstance");

    case EMeshPass::EditorSelection: return TEXT("EditorSelection");

#endif

    }

  

#if WITH_EDITOR

    static_assert(EMeshPass::Num == 40 + 4, "Need to update switch(MeshPass) after changing EMeshPass"); // GUID to prevent incorrect auto-resolves, please change when changing the expression: {674D7D62-CFD8-4971-9A8D-CD91E5612CD8}

#else

    static_assert(EMeshPass::Num == 40, "Need to update switch(MeshPass) after changing EMeshPass"); // GUID to prevent incorrect auto-resolves, please change when changing the expression: {674D7D62-CFD8-4971-9A8D-CD91E5612CD8}

#endif

  

    checkf(0, TEXT("Missing case for EMeshPass %u"), (uint32)MeshPass);

    return nullptr;

}
```

##### **改动 C** — 更新 static_assert

// 从 39 改为 40（非 Editor），从 39+4 改为 40+4（Editor）

static_assert(EMeshPass::Num == 40, ...);

```
inline const TCHAR* GetMeshPassName(EMeshPass::Type MeshPass)

{

	......

    case EMeshPass::RealityDistortion: return TEXT("RealityDistortion");
	......

    }

  

#if WITH_EDITOR

    static_assert(EMeshPass::Num == 40 + 4, "Need to update switch(MeshPass) after changing EMeshPass"); // GUID to prevent incorrect auto-resolves, please change when changing the expression: {674D7D62-CFD8-4971-9A8D-CD91E5612CD8}

#else

    static_assert(EMeshPass::Num == 40, "Need to update switch(MeshPass) after changing EMeshPass"); // GUID to prevent incorrect auto-resolves, please change when changing the expression: {674D7D62-CFD8-4971-9A8D-CD91E5612CD8}

#endif

  

    checkf(0, TEXT("Missing case for EMeshPass %u"), (uint32)MeshPass);

    return nullptr;

}
```

#### 渲染 Pass 的"前置条件"— 可见性注册
引擎的可见性系统（ ==SceneVisibility.cpp== ）在每帧/每次缓存更新时，根据各种条件（材质类型、ViewRelevance 等）决定把每个 StaticMesh/DynamicMesh 分配到哪些 EMeshPass 中。如果你不在这里注册，你的 Processor 的 AddMeshBatch 永远不会被调用。 改动位置： Engine/Source/Runtime/Renderer/Private/SceneVisibility.cpp

##### 改动 A — 静态网格路径

```
// 原有代码（约 L1717）
DrawCommandPacket.AddCommandsForMesh(PrimitiveIndex, PrimitiveSceneInfo, StaticMeshRelevance, StaticMesh, CullingPayloadFlags, Scene, bCanCache, EMeshPass::BasePass);

// ★ 追加：所有进入 BasePass 的静态网格也进入 RealityDistortion
DrawCommandPacket.AddCommandsForMesh(PrimitiveIndex, PrimitiveSceneInfo, StaticMeshRelevance, StaticMesh, CullingPayloadFlags, Scene, false, EMeshPass::RealityDistortion);

// 原有代码继续
MarkMask |= EMarkMaskBits::StaticMeshVisibilityMapMask;
```

##### 改动 B — 动态网格路径
在同文件的 ==ComputeDynamicMeshRelevance== 函数中，找到动态网格的 BasePass 注册（if (ViewRelevance.bRenderInMainPass || ViewRelevance.bRenderCustomDepth)），在 BasePass 设置后面追加：

```
// 原有代码（约 L2407-2410）
if (ViewRelevance.bRenderInMainPass || ViewRelevance.bRenderCustomDepth)
{
    PassMask.Set(EMeshPass::BasePass);
    View.NumVisibleDynamicMeshElements[EMeshPass::BasePass] += NumElements;

    // ★ 追加：所有进入 BasePass 的动态网格也进入 RealityDistortion
    PassMask.Set(EMeshPass::RealityDistortion);
    View.NumVisibleDynamicMeshElements[EMeshPass::RealityDistortion] += NumElements;

    // 原有代码继续（SkyPass 等）
    if (ViewRelevance.bUsesSkyMaterial)
    {
        ...
    }
}
```

原理：PassMask.Set 会让引擎在后续的 GenerateDynamicMeshDrawCommands 中，对这些动态 Mesh 调用你的 FRealityDistortionPassProcessor::AddMeshBatch。NumVisibleDynamicMeshElements 的计数用于预分配内存。

### 编写特效场组件
具体效果就是通过效果场去影响物体的渲染流程
**声明力场的参数化变量**

```
// Copyright Epic Games, Inc. All Rights Reserved.

  

#pragma once

  

#include "Containers/ArrayView.h"

#include "CoreMinimal.h"

  

// 每个 Distortion 力场的一份参数快照（RT 读取）。

struct FRealityDistortionFieldSettings

{

    // 力场中心（世界空间）。

    FVector Center = FVector::ZeroVector;

  

    // 力场半径（世界空间）。

    float Radius = 500.0f;

  

    // 开关位：RT 过滤时只处理启用项。

    bool bEnabled = false;

  

    // 力场匹配接收体的 Tag。

    // None 表示不做 Tag 限制（命中半径即可）。

    FName ReceiverTagFilter = NAME_None;

};

  

// 句柄语义：

// - GT 持有 Handle；RT 用 Handle 维护对应条目。

// - 0 为无效句柄，不参与任何更新。

using FRealityDistortionFieldHandle = uint32;

static constexpr FRealityDistortionFieldHandle RealityDistortionInvalidFieldHandle = 0;

  

// 在 GT 上创建/销毁一个 Field 实例句柄。

RENDERER_API FRealityDistortionFieldHandle CreateRealityDistortionFieldHandle_GameThread();

RENDERER_API void DestroyRealityDistortionFieldHandle_GameThread(FRealityDistortionFieldHandle FieldHandle);

  

// 在 GT 上更新指定 Field；真正写入通过 ENQUEUE_RENDER_COMMAND 在 RT 执行。

RENDERER_API void SetRealityDistortionFieldSettings_GameThread(

    FRealityDistortionFieldHandle FieldHandle,

    const FRealityDistortionFieldSettings& InSettings);

  

// 兼容旧单实例调用：内部使用保留 Handle。

RENDERER_API void SetRealityDistortionFieldSettings_GameThread(const FRealityDistortionFieldSettings& InSettings);

  

// 在 RT 上读取当前所有 Field 快照。

// 返回值是只读视图，不应在外部缓存到跨帧。

RENDERER_API TConstArrayView<FRealityDistortionFieldSettings> GetRealityDistortionFieldSettings_RenderThread();
```

**将设置参数命令放入渲染线程执 RT 行队列**

```
// Copyright Epic Games, Inc. All Rights Reserved.

  

#include "RealityDistortionField.h"

  

#include "RenderCore.h"

#include "RenderingThread.h"

#include "RHICommandList.h"

  

namespace

{

    // 句柄生成器（从 1 开始，避开 0 无效值）。

    TAtomic<uint32> GNextRealityDistortionFieldHandle(1);

  

    // RT 持有真正参与过滤的数据表：Handle 数组 + Settings 数组一一对应。

    // Add/Update/Remove 都在 RT 命令中执行，保证线程时序一致。

    TArray<FRealityDistortionFieldHandle> GRealityDistortionFieldHandles_RT;

    TArray<FRealityDistortionFieldSettings> GRealityDistortionFields_RT;

  

    // 兼容旧接口使用的保留 Handle。

    constexpr FRealityDistortionFieldHandle GLegacyRealityDistortionFieldHandle = 1;

  

    // 输入收敛：防止负半径等非法值进入 RT 数据表。

    FRealityDistortionFieldSettings SanitizeFieldSettings(const FRealityDistortionFieldSettings& InSettings)

    {

        FRealityDistortionFieldSettings OutSettings = InSettings;

        OutSettings.Radius = FMath::Max(0.0f, OutSettings.Radius);

        OutSettings.bEnabled = OutSettings.bEnabled && OutSettings.Radius > 0.0f;

        return OutSettings;

    }

  

    int32 FindFieldIndex_RT(FRealityDistortionFieldHandle FieldHandle)

    {

        return GRealityDistortionFieldHandles_RT.IndexOfByKey(FieldHandle);

    }

  

    // Upsert 语义：存在则覆盖，不存在则新增。

    void UpsertField_RT(FRealityDistortionFieldHandle FieldHandle, const FRealityDistortionFieldSettings& InSettings)

    {

        const int32 ExistingIndex = FindFieldIndex_RT(FieldHandle);

        if (ExistingIndex == INDEX_NONE)

        {

            GRealityDistortionFieldHandles_RT.Add(FieldHandle);

            GRealityDistortionFields_RT.Add(InSettings);

        }

        else

        {

            GRealityDistortionFields_RT[ExistingIndex] = InSettings;

        }

    }

}

  

FRealityDistortionFieldHandle CreateRealityDistortionFieldHandle_GameThread()

{

    check(IsInGameThread());

  

    FRealityDistortionFieldHandle NewHandle = GNextRealityDistortionFieldHandle.IncrementExchange();

    if (NewHandle == RealityDistortionInvalidFieldHandle)

    {

        NewHandle = GNextRealityDistortionFieldHandle.IncrementExchange();

    }

  

    return NewHandle;

}

  

void DestroyRealityDistortionFieldHandle_GameThread(FRealityDistortionFieldHandle FieldHandle)

{

    if (FieldHandle == RealityDistortionInvalidFieldHandle)

    {

        return;

    }

  

    // 在 GT 只入队；真正删除在 RT 命令队列消费时执行。

    // 这样可以和 RT 同帧内的读取保持严格的队列顺序。

    ENQUEUE_RENDER_COMMAND(DestroyRealityDistortionField)(

        [FieldHandle](FRHICommandListImmediate&)

        {

            const int32 ExistingIndex = FindFieldIndex_RT(FieldHandle);

            if (ExistingIndex != INDEX_NONE)

            {

                GRealityDistortionFieldHandles_RT.RemoveAtSwap(ExistingIndex);

                GRealityDistortionFields_RT.RemoveAtSwap(ExistingIndex);

            }

        });

}

  

void SetRealityDistortionFieldSettings_GameThread(

    FRealityDistortionFieldHandle FieldHandle,

    const FRealityDistortionFieldSettings& InSettings)

{

    if (FieldHandle == RealityDistortionInvalidFieldHandle)

    {

        return;

    }

  

    const FRealityDistortionFieldSettings SanitizedSettings = SanitizeFieldSettings(InSettings);

  

    // 入队到 RT，确保修改与渲染读取遵循同一命令队列顺序。

    ENQUEUE_RENDER_COMMAND(SetRealityDistortionFieldSettings)(

        [FieldHandle, SanitizedSettings](FRHICommandListImmediate&)

        {

            UpsertField_RT(FieldHandle, SanitizedSettings);

        });

}

  

void SetRealityDistortionFieldSettings_GameThread(const FRealityDistortionFieldSettings& InSettings)

{

    SetRealityDistortionFieldSettings_GameThread(GLegacyRealityDistortionFieldHandle, InSettings);

}

  

TConstArrayView<FRealityDistortionFieldSettings> GetRealityDistortionFieldSettings_RenderThread()

{

    check(IsInRenderingThread() || IsInParallelRenderingThread());

    return MakeArrayView(GRealityDistortionFields_RT);

}
```

### 受影响物体组件
 组件有受场影响开关，和劫持材质

```
// DistortionMeshComponent.h

//

// UDistortionMeshComponent（Receiver）

// ------------------------------------

// 职责：

// 1) 继承 UStaticMeshComponent，保持普通静态网格的编辑体验。

// 2) 重写 CreateSceneProxy()，把渲染代理切换为 FDistortionSceneProxy。

// 3) 提供接收体主开关与覆盖材质。

//

// 注意：

// - 这个组件不负责“发射力场”，发射职责在 UDistortionFieldComponent。

// - 这个组件不直接做渲染决策，真正决策在 FRealityDistortionPassProcessor::AddMeshBatch。

  

#pragma once

  

#include "CoreMinimal.h"

#include "Components/StaticMeshComponent.h"

#include "DistortionMeshComponent.generated.h"

  

UCLASS(ClassGroup=(Rendering), meta=(BlueprintSpawnableComponent))

class REALITYDISTORTION_API UDistortionMeshComponent : public UStaticMeshComponent

{

    GENERATED_BODY()

  

public:

    UDistortionMeshComponent(const FObjectInitializer& ObjectInitializer);

  

    // 通过自定义 SceneProxy 接入渲染管线。

    // 之后该 Primitive 在 RT 会以 FDistortionSceneProxy 的形态参与收集与过滤。

    virtual FPrimitiveSceneProxy* CreateSceneProxy() override;

  

    // Phase 1 材质劫持入口。

    // DistortionSceneProxy::GetDynamicMeshElements 会把 MeshBatch.MaterialRenderProxy

    // 替换为此材质的 RenderProxy。

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Distortion")

    TObjectPtr<UMaterialInterface> OverrideMaterial;

  

    // Receiver 主开关：false 表示该组件永远不作为 Distortion 接收体。

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Distortion|Receiver")

    bool bEnableDistortionReceiver = true;

};
```

实现==CreateSceneProxy()==

```
// DistortionMeshComponent.cpp

  

#include "Rendering/DistortionMeshComponent.h"

  

#include "Materials/Material.h"

#include "Rendering/DistortionSceneProxy.h"

#include "UObject/ConstructorHelpers.h"

  

UDistortionMeshComponent::UDistortionMeshComponent(const FObjectInitializer& ObjectInitializer)

    : Super(ObjectInitializer)

{

    // 默认给一个可见材质，便于未配置时直接看到“材质劫持是否生效”。

    // 后续你可以在蓝图/细节面板替换为自己的 Distortion 材质。

    static ConstructorHelpers::FObjectFinder<UMaterial> DefaultMaterialFinder(

        TEXT("/Engine/EngineMaterials/WorldGridMaterial"));

    if (DefaultMaterialFinder.Succeeded())

    {

        OverrideMaterial = DefaultMaterialFinder.Object;

    }

}

  

FPrimitiveSceneProxy* UDistortionMeshComponent::CreateSceneProxy()

{

    // ------------------------------

    // 关键入口：切换到自定义 SceneProxy

    // ------------------------------

    // 这里做足前置检查，避免把无效资源带到 RT。

    if (GetStaticMesh() == nullptr)

    {

        return nullptr;

    }

  

    if (GetStaticMesh()->GetRenderData() == nullptr)

    {

        return nullptr;

    }

  

    if (!GetStaticMesh()->GetRenderData()->IsInitialized())

    {

        return nullptr;

    }

  

    // 交给 FDistortionSceneProxy，后续 MeshBatch 会在其 GetDynamicMeshElements 中被“劫持”。

    return new FDistortionSceneProxy(this);

}
```

### 编写处理器

#### **编写处理器：** 复制 `FDepthPassMeshProcessor` (因为它最简单) 的代码，改名为 `FRealityDistortionPassProcessor`。

###### RealityDistortionPassProcessor.h 定义
整个文件是新建的。定义了 ==FRealityDistortionPassProcessor== 类： 继承自 FMeshPassProcessor  重写 AddMeshBatch （空间过滤入口） 私有函数 TryAddMeshBatch （BlendMode 过滤）和 Process （Shader 查找 + BuildMeshDrawCommands）

```
// RealityDistortionPassProcessor.h

//

// FRealityDistortionPassProcessor

// ------------------------------

// 这是 RealityDistortion Pass 的决策层：

// 1) AddMeshBatch: 决定“画什么”（Receiver/空间/材质三层过滤）

// 2) TryAddMeshBatch: 决定“是否可用当前材质 + 是否需要 DefaultMaterial 回退”

// 3) Process: 决定“怎么画”（Shader/Pipeline/RenderState/DrawCommand）

  

#pragma once

  

#include "CoreMinimal.h"

#include "MeshPassProcessor.h"

  

class FRealityDistortionPassProcessor

    : public FSceneRenderingAllocatorObject<FRealityDistortionPassProcessor>

    , public FMeshPassProcessor

{

public:

    FRealityDistortionPassProcessor(

        const FScene* Scene,

        ERHIFeatureLevel::Type FeatureLevel,

        const FSceneView* InViewIfDynamicMeshCommand,

        FMeshPassDrawListContext* InDrawListContext);

  

    // MeshPass 入口：每个候选 MeshBatch 都会走这里。

    virtual void AddMeshBatch(

        const FMeshBatch& RESTRICT MeshBatch,

        uint64 BatchElementMask,

        const FPrimitiveSceneProxy* RESTRICT PrimitiveSceneProxy,

        int32 StaticMeshId = -1) override final;

  

private:

    // 第二阶段过滤：处理 BlendMode/Domain，并做必要的材质回退。

    bool TryAddMeshBatch(

        const FMeshBatch& RESTRICT MeshBatch,

        uint64 BatchElementMask,

        const FPrimitiveSceneProxy* RESTRICT PrimitiveSceneProxy,

        int32 StaticMeshId,

        const FMaterialRenderProxy& MaterialRenderProxy,

        const FMaterial& Material);

  

    // 最终构建 DrawCommand：查找 Shader、组装 RenderState、调用 BuildMeshDrawCommands。

    bool Process(

        const FMeshBatch& RESTRICT MeshBatch,

        uint64 BatchElementMask,

        int32 StaticMeshId,

        const FPrimitiveSceneProxy* RESTRICT PrimitiveSceneProxy,

        const FMaterialRenderProxy& RESTRICT MaterialRenderProxy,

        const FMaterial& RESTRICT MaterialResource,

        ERasterizerFillMode MeshFillMode,

        ERasterizerCullMode MeshCullMode);

  

    FMeshPassProcessorRenderState PassDrawRenderState;

};
```

##### RealityDistortionPassProcessor.cpp 实现

###### **逻辑注入：**

    **- 在 `AddMeshBatch` 函数中加入空间判断逻辑。**

    **- 定义一个全局变量（或从 Scene 传入）`FieldCenter`。**

    **- 计算 `PrimitiveSceneProxy->GetBounds().Origin` 到 `FieldCenter` 的距离。**

    **- **If (Distance < 500)**: 调用 `BuildMeshDrawCommands`；**Else**: return。**

```

// RealityDistortionPassProcessor.cpp

  

#include "Rendering/RealityDistortionPassProcessor.h"

  

#include "MaterialShaderType.h"

#include "Materials/Material.h"

#include "MeshPassProcessor.inl"

#include "RealityDistortionField.h"

#include "Rendering/DistortionSceneProxy.h"

  

FRealityDistortionPassProcessor::FRealityDistortionPassProcessor(

    const FScene* Scene,

    ERHIFeatureLevel::Type FeatureLevel,

    const FSceneView* InViewIfDynamicMeshCommand,

    FMeshPassDrawListContext* InDrawListContext)

    : FMeshPassProcessor(EMeshPass::RealityDistortion, Scene, FeatureLevel, InViewIfDynamicMeshCommand, InDrawListContext)

{

    // 本 Pass 采用“类似深度 Pass”的基础状态：不混合，深度可写。

    PassDrawRenderState.SetBlendState(TStaticBlendState<>::GetRHI());

    PassDrawRenderState.SetDepthStencilState(TStaticDepthStencilState<true, CF_DepthNearOrEqual>::GetRHI());

}

  

void FRealityDistortionPassProcessor::AddMeshBatch(

    const FMeshBatch& RESTRICT MeshBatch,

    uint64 BatchElementMask,

    const FPrimitiveSceneProxy* RESTRICT PrimitiveSceneProxy,

    int32 StaticMeshId)

{

    if (PrimitiveSceneProxy == nullptr)

    {

        return;

    }

  

    // ==================================================

    // 第一层：Receiver 类型过滤

    // ==================================================

    // 只有自定义 FDistortionSceneProxy 才参与本 Pass。

    // 这样可以把“受影响物体”与普通 BasePass 物体分开。

    if (PrimitiveSceneProxy->GetTypeHash() != FDistortionSceneProxy::GetStaticTypeHash())

    {

        return;

    }

  

    const FDistortionSceneProxy* DistortionProxy = static_cast<const FDistortionSceneProxy*>(PrimitiveSceneProxy);

    if (!DistortionProxy->ShouldRenderInRealityDistortionPass())

    {

        return;

    }

  

    // ==================================================

    // 第二层：Field 过滤（Tag + 空间）

    // ==================================================

    // 读取 RT 当前所有 Field，命中任意一个启用 Field 才继续。

    const TConstArrayView<FRealityDistortionFieldSettings> Fields = GetRealityDistortionFieldSettings_RenderThread();

    if (Fields.IsEmpty())

    {

        return;

    }

  

    const FVector PrimitiveOrigin = PrimitiveSceneProxy->GetBounds().Origin;

    bool bInsideAnyField = false;

    for (const FRealityDistortionFieldSettings& Field : Fields)

    {

        if (!Field.bEnabled || Field.Radius <= 0.0f)

        {

            continue;

        }

  

        // Field 指定了 Tag 时，只影响标签命中的接收体。

        if (!DistortionProxy->HasReceiverTag(Field.ReceiverTagFilter))

        {

            continue;

        }

  

        const float DistSq = FVector::DistSquared(PrimitiveOrigin, Field.Center);

        if (DistSq <= FMath::Square(Field.Radius))

        {

            bInsideAnyField = true;

            break;

        }

    }

  

    if (!bInsideAnyField)

    {

        return;

    }

  

    if (!MeshBatch.bUseForMaterial)

    {

        return;

    }

  

    // ==================================================

    // 第三层：材质 fallback 链

    // ==================================================

    // 同 BasePass 思路：沿 MaterialRenderProxy->Fallback 链找可用 ShaderMap。

    const FMaterialRenderProxy* MaterialRenderProxy = MeshBatch.MaterialRenderProxy;

    while (MaterialRenderProxy)

    {

        const FMaterial* Material = MaterialRenderProxy->GetMaterialNoFallback(FeatureLevel);

        if (Material && Material->GetRenderingThreadShaderMap())

        {

            if (TryAddMeshBatch(MeshBatch, BatchElementMask, PrimitiveSceneProxy, StaticMeshId, *MaterialRenderProxy, *Material))

            {

                break;

            }

        }

  

        MaterialRenderProxy = MaterialRenderProxy->GetFallback(FeatureLevel);

    }

}

  

bool FRealityDistortionPassProcessor::TryAddMeshBatch(

    const FMeshBatch& RESTRICT MeshBatch,

    uint64 BatchElementMask,

    const FPrimitiveSceneProxy* RESTRICT PrimitiveSceneProxy,

    int32 StaticMeshId,

    const FMaterialRenderProxy& MaterialRenderProxy,

    const FMaterial& Material)

{

    // 只处理不透明/Masked；半透明直接跳过。

    const EBlendMode BlendMode = Material.GetBlendMode();

    if (IsTranslucentBlendMode(BlendMode))

    {

        return true;

    }

  

    // Volume 材质域不在本 Pass 渲染。

    if (Material.GetMaterialDomain() == MD_Volume)

    {

        return true;

    }

  

    const FMaterialRenderProxy* FinalMaterialProxy = &MaterialRenderProxy;

    const FMaterial* FinalMaterial = &Material;

  

    // DepthOnly shader 对“普通不透明材质”常常没有可用 permutation。

    // 这里沿用引擎深度 Pass 的做法：必要时回退到 DefaultMaterial。

    const bool bNeedsDefaultMaterial = Material.WritesEveryPixel(false, false)

        && !Material.MaterialUsesPixelDepthOffset_RenderThread()

        && !Material.MaterialMayModifyMeshPosition();

  

    if (bNeedsDefaultMaterial)

    {

        const FMaterialRenderProxy* DefaultProxy = UMaterial::GetDefaultMaterial(MD_Surface)->GetRenderProxy();

        const FMaterial* DefaultMaterial = DefaultProxy->GetMaterialNoFallback(FeatureLevel);

        if (DefaultMaterial && DefaultMaterial->GetRenderingThreadShaderMap())

        {

            FinalMaterialProxy = DefaultProxy;

            FinalMaterial = DefaultMaterial;

        }

    }

  

    // 计算 Fill/Cull 状态，准备进入 Process 构建 DrawCommand。

    const FMeshDrawingPolicyOverrideSettings OverrideSettings = ComputeMeshOverrideSettings(MeshBatch);

    const ERasterizerFillMode MeshFillMode = ComputeMeshFillMode(*FinalMaterial, OverrideSettings);

    const ERasterizerCullMode MeshCullMode = ComputeMeshCullMode(*FinalMaterial, OverrideSettings);

  

    return Process(

        MeshBatch,

        BatchElementMask,

        StaticMeshId,

        PrimitiveSceneProxy,

        *FinalMaterialProxy,

        *FinalMaterial,

        MeshFillMode,

        MeshCullMode);

}

  

bool FRealityDistortionPassProcessor::Process(

    const FMeshBatch& RESTRICT MeshBatch,

    uint64 BatchElementMask,

    int32 StaticMeshId,

    const FPrimitiveSceneProxy* RESTRICT PrimitiveSceneProxy,

    const FMaterialRenderProxy& RESTRICT MaterialRenderProxy,

    const FMaterial& RESTRICT MaterialResource,

    ERasterizerFillMode MeshFillMode,

    ERasterizerCullMode MeshCullMode)

{

    const FVertexFactory* VertexFactory = MeshBatch.VertexFactory;

  

    // 运行时按名称查询 Depth Pass 的 shader/pipeline。

    // 这样可以避免直接依赖 Renderer 私有模板类型定义。

    static FShaderType* DepthVSType = FShaderType::GetShaderTypeByName(TEXT("TDepthOnlyVS<false>"));

    static FShaderType* DepthPSType = FShaderType::GetShaderTypeByName(TEXT("FDepthOnlyPS"));

    static const FShaderPipelineType* DepthNoPixelPipeline =

        FShaderPipelineType::GetShaderPipelineTypeByName(FHashedName(TEXT("DepthNoPixelPipeline")));

    static const FShaderPipelineType* DepthWithPixelPipeline =

        FShaderPipelineType::GetShaderPipelineTypeByName(FHashedName(TEXT("DepthPipeline")));

  

    if (!DepthVSType)

    {

        return false;

    }

  

    // 只有“不写满像素”或使用 PixelDepthOffset 的材质才需要 PS。

    const bool bVFTypeSupportsNullPixelShader = VertexFactory->GetType()->SupportsNullPixelShader();

    const bool bNeedsPixelShader = !MaterialResource.WritesEveryPixel(false, bVFTypeSupportsNullPixelShader)

        || MaterialResource.MaterialUsesPixelDepthOffset_RenderThread();

  

    FMaterialShaderTypes ShaderTypes;

    ShaderTypes.AddShaderType(DepthVSType);

    if (bNeedsPixelShader && DepthPSType)

    {

        ShaderTypes.AddShaderType(DepthPSType);

        ShaderTypes.PipelineType = DepthWithPixelPipeline;

    }

    else

    {

        ShaderTypes.PipelineType = DepthNoPixelPipeline;

    }

  

    FMaterialShaders Shaders;

    if (!MaterialResource.TryGetShaders(ShaderTypes, VertexFactory->GetType(), Shaders))

    {

        return false;

    }

  

    TMeshProcessorShaders<FMeshMaterialShader, FMeshMaterialShader> PassShaders;

    Shaders.TryGetShader(SF_Vertex, PassShaders.VertexShader);

    if (bNeedsPixelShader)

    {

        Shaders.TryGetShader(SF_Pixel, PassShaders.PixelShader);

    }

  

    // ShaderElementData 会把 Primitive/Material 相关绑定数据带到 DrawCommand。

    FMeshMaterialShaderElementData ShaderElementData;

    ShaderElementData.InitializeMeshMaterialData(ViewIfDynamicMeshCommand, PrimitiveSceneProxy, MeshBatch, StaticMeshId, false);

  

    const FMeshDrawCommandSortKey SortKey = CalculateMeshStaticSortKey(PassShaders.VertexShader, PassShaders.PixelShader);

  

    // 最终生成 FMeshDrawCommand（包含 PSO、Shader、VertexStreams、Bindings）。

    BuildMeshDrawCommands(

        MeshBatch,

        BatchElementMask,

        PrimitiveSceneProxy,

        MaterialRenderProxy,

        MaterialResource,

        PassDrawRenderState,

        PassShaders,

        MeshFillMode,

        MeshCullMode,

        SortKey,

        EMeshPassFeatures::Default,

        ShaderElementData);

  

    return true;

}

  

static FMeshPassProcessor* CreateRealityDistortionPassProcessor(

    ERHIFeatureLevel::Type FeatureLevel,

    const FScene* Scene,

    const FSceneView* InViewIfDynamicMeshCommand,

    FMeshPassDrawListContext* InDrawListContext)

{

    return new FRealityDistortionPassProcessor(Scene, FeatureLevel, InViewIfDynamicMeshCommand, InDrawListContext);

}

  

// 注册到 Deferred + MainView。

// 是否缓存静态命令由 SceneVisibility.cpp 的 AddCommandsForMesh(bCanCache) 决定。

REGISTER_MESHPASSPROCESSOR_AND_PSOCOLLECTOR(

    RealityDistortionPass,

    CreateRealityDistortionPassProcessor,

    EShadingPath::Deferred,

    EMeshPass::RealityDistortion,

    EMeshPassFlags::CachedMeshCommands | EMeshPassFlags::MainView);
```

#### 渲染插入

##### **渲染插入：** 找到 `FSceneRenderer::Render`，模仿 `RenderCustomDepthPass`，在合适的位置调用你的 `DistortionPass`。

###### **改动 1** — 在文件顶部添加参数结构体定义：

```
// === RealityDistortion Pass 参数结构体 ===

BEGIN_SHADER_PARAMETER_STRUCT(FRealityDistortionPassParameters, )

    SHADER_PARAMETER_STRUCT_INCLUDE(FViewShaderParameters, View)

    SHADER_PARAMETER_STRUCT_INCLUDE(FInstanceCullingDrawParams, InstanceCullingDrawParams)

    SHADER_PARAMETER_STRUCT_INCLUDE(FSceneTextureShaderParameters, SceneTextures)

    RENDER_TARGET_BINDING_SLOTS()

END_SHADER_PARAMETER_STRUCT()
```

##### **改动 2** — 在 Render 函数中，CustomDepth AfterBasePass 之后、Velocity 之前，插入调度代码）：

```
// === RealityDistortion Pass ===

        // 调度自定义 MeshPass：遍历 View，执行 EMeshPass::RealityDistortion 的 MeshDrawCommands

        {

            RDG_EVENT_SCOPE(GraphBuilder, "RealityDistortionPass");

  

            for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ++ViewIndex)

            {

                FViewInfo& View = Views[ViewIndex];

                if (auto* Pass = View.ParallelMeshDrawCommandPasses[EMeshPass::RealityDistortion];

                    Pass && View.ShouldRenderView())

                {

                    RDG_EVENT_SCOPE_CONDITIONAL(GraphBuilder, Views.Num() > 1, "View%d", ViewIndex);

  

                    View.BeginRenderView();

  

					auto* PassParameters = GraphBuilder.AllocParameters<FRealityDistortionPassParameters>();
					
					PassParameters->View = View.GetShaderParameters(); // ★ 新增
					
					PassParameters->SceneTextures = SceneTextures.GetSceneTextureShaderParameters(FeatureLevel); // ★ 新增
					
					PassParameters->RenderTargets.DepthStencil = FDepthStencilBinding(
					
					SceneTextures.Depth.Target,
					
					ERenderTargetLoadAction::ELoad,
					
					ERenderTargetLoadAction::ELoad,
					
					FExclusiveDepthStencil::DepthWrite_StencilWrite);
  

                    Pass->BuildRenderingCommands(GraphBuilder, Scene->GPUScene, PassParameters->InstanceCullingDrawParams);

  

                    GraphBuilder.AddPass(

                        RDG_EVENT_NAME("RealityDistortion"),

                        PassParameters,

                        ERDGPassFlags::Raster,

                        [&View, Pass, PassParameters](FRDGAsyncTask, FRHICommandList& RHICmdList)

                    {

                        SetStereoViewport(RHICmdList, View, 1.0f);

                        Pass->Draw(RHICmdList, &PassParameters->InstanceCullingDrawParams);

                    });

                }

            }

        }
```

### 验收

-**✅ 验收标准：** 用 RenderDoc 截帧，发现多了一个 Pass。当你移动“力场”位置时，该 Pass 内包含的 DrawCall 数量在动态变化。
有深度的都说受到影响的 mesh
![[Pasted image 20260218224346.png]]

ue_log 主要看 BuiltDrawCommands，HasAnyDraw 只作参考
