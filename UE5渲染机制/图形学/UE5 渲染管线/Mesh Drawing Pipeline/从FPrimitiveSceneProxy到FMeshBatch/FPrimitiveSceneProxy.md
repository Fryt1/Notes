### 类职责

**将场景图元数据转换为可渲染的网格批次**

### 关键成员

```
FPrimitiveSceneInfo* PrimitiveSceneInfo; // 对应的场景图元信息
```

### 成员在 MeshBatch 收集中的作用

- **PrimitiveSceneInfo**：连接渲染代理与场景图元数据的桥梁

### 关键方法

```
virtual void GetDynamicMeshElements(
    const TArray<const FSceneView*>& Views,
    const FSceneViewFamily& ViewFamily, 
    uint32 VisibilityMap,
    FMeshElementCollector& Collector);
// 核心方法：生成动态网格元素并填充到收集器
```
