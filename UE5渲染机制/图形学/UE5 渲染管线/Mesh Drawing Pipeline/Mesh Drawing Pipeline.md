# Mesh Drawing Pipeline

这组笔记解释 UE5 如何把 `FPrimitiveSceneProxy` 生成的动态网格元素，收集为 `FMeshBatch`，并分发到各个 Mesh Pass。

## 推荐阅读顺序

1. [[核心流程和数据流向]]
2. [[GatherDynamicMeshElements]]
3. [[FSceneRenderer]] → [[FVisibilityTaskData]] → [[FDynamicMeshElementContextContainer]] → [[DynamicMeshElementContext]]
4. [[FPrimitiveSceneProxy]] → [[FMeshElementCollector]] → [[struct FMeshBatch]] → [[struct FMeshBatchElement]]
5. [[FMaterialRenderProxy]] 与 [[FVertexFactory]]

![[图形学/UE5 渲染管线/Mesh Drawing Pipeline/资源/Pasted image 20251018165757.png]]
