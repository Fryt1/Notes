![[Pasted image 20251018165757.png]]

### 类职责

**渲染批次的核心容器，描述一组共享相同渲染状态的网格元素**

### 关键数据结构

```
struct FMeshBatch {
    // 网格元素数组 - 同一批次中的多个网格实例
    TArray<FMeshBatchElement, TInlineAllocator<1>> Elements;
    
    // 顶点数据工厂 - 定义顶点格式和缓冲
    const FVertexFactory* VertexFactory;
    
    // 材质渲染代理 - 定义表面外观和着色器
    const FMaterialRenderProxy* MaterialRenderProxy;
    
    // 渲染相关参数
    uint16 MeshIdInPrimitive;     // 图元内网格ID，用于稳定排序
    int8 LODIndex;                // LOD级别索引
    uint8 SegmentIndex;           // 子模型索引
    
    // 渲染状态标记
    uint32 ReverseCulling : 1;    // 是否反转剔除
    uint32 bDisableBackfaceCulling : 1; // 是否禁用背面剔除
    uint32 CastShadow : 1;        // 是否投射阴影
    uint32 bUseForMaterial : 1;   // 是否用于材质渲染
    uint32 bUseForDepthPass : 1;  // 是否用于深度通道
    uint32 bUseAsOccluder : 1;    // 是否作为遮挡物
    uint32 bWireframe : 1;        // 是否线框模式
    
    // 图元类型和深度优先级
    uint32 Type : PT_NumBits;     // 图元类型(三角形列表、线列表等)
    uint32 DepthPriorityGroup : SDPG_NumBits; // 深度优先级组
    
    // 其他渲染数据
    const FLightCacheInterface* LCI; // 光照缓存接口
    FHitProxyId BatchHitProxyId;  // 命中代理ID
};

```

### 在 MeshBatch 收集中的作用

- **容器作用**：将多个 FMeshBatchElement 组织在一起，共享相同的顶点工厂和材质

- **状态管理**：统一管理整个批次的渲染状态和参数

- **效率优化**：通过批次化减少状态切换，提高渲染效率

### 关键方法

```
bool IsTranslucent(ERHIFeatureLevel::Type InFeatureLevel) const;
bool IsDecal(ERHIFeatureLevel::Type InFeatureLevel) const;
bool IsMasked(ERHIFeatureLevel::Type InFeatureLevel) const;
int32 GetNumPrimitives() const;  // 获取批次中图元总数
bool HasAnyDrawCalls() const;    // 检查是否有实际绘制调用
```
