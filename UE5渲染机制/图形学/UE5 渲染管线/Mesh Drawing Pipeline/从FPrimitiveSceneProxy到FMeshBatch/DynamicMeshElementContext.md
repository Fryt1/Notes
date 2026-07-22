### 类职责

**单个并行单元的动态网格元素收集和临时数据管理**

### 关键成员

```
FMeshElementCollector MeshCollector;         // 网格元素收集器
FMeshElementCollector EditorMeshCollector;   // 编辑器专用收集器
TArray<FDynamicPrimitive> DynamicPrimitives; // 本地收集结果
FRHICommandList* RHICmdList;                 // RHI命令列表
```

### 成员在 MeshBatch 收集中的作用

- **MeshCollector**：实际的网格批次收集工具，存储 FMeshBatch 对象

- **EditorMeshCollector**：编辑器模式下专用的网格收集器

- **DynamicPrimitives**：记录本收集器处理的动态图元及其 MeshBatch 范围

- **RHICmdList**：管理本收集器的渲染命令

### 关键方法

```
void GatherDynamicMeshElementsForPrimitive(FPrimitiveSceneInfo* Primitive, uint8 ViewMask);
// 核心方法：为单个图元收集动态网格元素

UE::Tasks::FTask LaunchAsyncTask(FDynamicPrimitiveIndexQueue* PrimitiveIndexQueue, ...);
// 启动异步收集任务
```
