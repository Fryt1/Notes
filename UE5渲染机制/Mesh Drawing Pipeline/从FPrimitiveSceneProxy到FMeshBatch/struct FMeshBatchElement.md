### 类职责

**单个网格元素的渲染数据描述，包含具体的几何和实例信息**

### 关键数据结构

```
struct FMeshBatchElement {
    // Uniform Buffer数据
    FRHIUniformBuffer* PrimitiveUniformBuffer;        // 图元UniformBuffer
    const TUniformBuffer<FPrimitiveUniformShaderParameters>* PrimitiveUniformBufferResource; // CPU侧Uniform数据
    
    // 几何数据
    const FIndexBuffer* IndexBuffer;                  // 索引缓冲区
    uint32 FirstIndex;                                // 起始索引位置
    uint32 NumPrimitives;                             // 图元数量
    uint32 BaseVertexIndex;                           // 基础顶点索引
    uint32 MinVertexIndex;                            // 最小顶点索引
    uint32 MaxVertexIndex;                            // 最大顶点索引
    
    // 实例化数据
    uint32 NumInstances;                              // 实例数量
    uint32* InstanceRuns;                             // 实例运行数据(用于层次实例化)
    
    // 图元ID模式
    EPrimitiveIdMode PrimitiveIdMode : PrimID_NumBits + 1; // 图元ID模式
    uint32 DynamicPrimitiveShaderDataIndex : 24;      // 动态图元着色器数据索引
    
    // 用户数据
    const void* UserData;                             // 用户自定义数据
    void* VertexFactoryUserData;                      // 顶点工厂用户数据
    int32 UserIndex;                                  // 用户索引
    
    // LOD和屏幕尺寸
    uint32 InstancedLODIndex : 4;                     // 实例化LOD索引
    uint32 InstancedLODRange : 4;                     // 实例化LOD范围
    float MinScreenSize;                              // 最小屏幕尺寸
    float MaxScreenSize;                              // 最大屏幕尺寸
    
    // 标记位
    uint32 bUserDataIsColorVertexBuffer : 1;          // 用户数据是否为颜色顶点缓冲
    uint32 bIsSplineProxy : 1;                        // 是否为样条代理
    uint32 bIsInstanceRuns : 1;                       // 是否为实例运行模式
};
```

### 在MeshBatch收集中的作用

- **几何数据承载**：存储具体的顶点、索引、实例等渲染数据
    
- **实例化支持**：支持各种实例化渲染技术
    
- **灵活数据扩展**：通过用户数据支持自定义渲染需求
    

### 关键方法

```
int32 GetNumPrimitives() const;
// 计算此元素的总图元数量，考虑实例化因素
```