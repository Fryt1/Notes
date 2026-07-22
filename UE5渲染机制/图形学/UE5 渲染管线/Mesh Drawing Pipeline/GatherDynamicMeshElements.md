在 Engine/Source/Runtime/Renderer/Private/SceneVisibility.cpp

UE5.54 版本该方法是改成了
FVisibilityTaskData::GatherDynamicMeshElements

在 void FDeferredShadingSceneRenderer::Render(FRDGBuilder& GraphBuilder)中的

```
//其中会调用FVisibilityTaskData::GatherDynamicMeshElements
RDG_EVENT_SCOPE_STAT(GraphBuilder, VisibilityCommands, "VisibilityCommands");  
RDG_GPU_STAT_SCOPE(GraphBuilder, VisibilityCommands);  
BeginInitViews(GraphBuilder, SceneTexturesConfig, InstanceCullingManager, ExternalAccessQueue, InitViewTaskDatas);
```

然后 FVisibilityTaskData::GatherDynamicMeshElements 会通过任务系统来调用
FDynamicMeshElementContext::GatherDynamicMeshElementsForPrimitive

```
//使用任务系统给DynamicMeshElements添加依赖，触发DynamicMeshElements.Trigger()就会开始依赖任务和DynamicMeshElements任务本身（但这里DynamicMeshElements任务是空实现）
if (!DynamicPrimitiveIndexList.IsEmpty())  
{  
    FDynamicPrimitiveIndexQueue* Queue = Allocator.Create<FDynamicPrimitiveIndexQueue>(MoveTemp(DynamicPrimitiveIndexList));  
  
    for (int32 Index = 0; Index < DynamicMeshElements.ContextContainer.GetNumAsyncContexts(); ++Index)  
    {       Tasks.DynamicMeshElements.AddPrerequisites(DynamicMeshElements.ContextContainer.LaunchAsyncTask(Queue, Index, TaskConfig.TaskPriority));  
    }}
```

**DynamicMeshElements.ContextContainer.LaunchAsyncTask(Queue, Index, TaskConfig.TaskPriority)**这个依赖任务的具体实现

这步重要代码是 **Primitive->Proxy->GetDynamicMeshElements(FirstViewFamily.AllViews, *Group.Family, MaskedViewMask, MeshCollector);**

```
void FDynamicMeshElementContext::GatherDynamicMeshElementsForPrimitive
(FPrimitiveSceneInfo* Primitive, uint8 ViewMask)  
{  
    SCOPED_NAMED_EVENT(DynamicPrimitive, FColor::Magenta);  
  
    TArray<int32, TInlineAllocator<4>> MeshBatchCountBefore;  
    MeshBatchCountBefore.SetNumUninitialized(Views.Num());  
    for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)  
    {       MeshBatchCountBefore[ViewIndex] = MeshCollector.GetMeshBatchCount(ViewIndex);  
    }  
    MeshCollector.SetPrimitive(Primitive->Proxy, Primitive->DefaultDynamicHitProxyId);  
  
    // If Custom Render Passes aren't in use, there will be only one group, which is the common case.  
    if (ViewFamilyGroups.Num() == 1 || Primitive->Proxy->SinglePassGDME())  
    {       Primitive->Proxy->GetDynamicMeshElements(FirstViewFamily.AllViews, FirstViewFamily, ViewMask, MeshCollector);  
    }    else  
    {  
       for (FViewFamilyGroup& Group : ViewFamilyGroups)  
       {          if (uint8 MaskedViewMask = ViewMask & Group.ViewSubsetMask)  
          {             Primitive->Proxy->GetDynamicMeshElements(FirstViewFamily.AllViews, *Group.Family, MaskedViewMask, MeshCollector);  
          }       }    }  
    for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)  
    {       FViewInfo& View = *Views[ViewIndex];  
  
       if (ViewMask & (1 << ViewIndex))  
       {          FDynamicPrimitive& DynamicPrimitive = DynamicPrimitives.Emplace_GetRef();  
          DynamicPrimitive.PrimitiveIndex = Primitive->GetIndex();  
          DynamicPrimitive.ViewIndex = ViewIndex;  
          DynamicPrimitive.StartElementIndex = MeshBatchCountBefore[ViewIndex];  
          DynamicPrimitive.EndElementIndex = MeshCollector.GetMeshBatchCount(ViewIndex);  
       }    }}
```

## 处理动态网格元素的相关性
这部分被改到了 FVisibilityTaskData::SetupMeshPasses 中

```
void FVisibilityTaskData::SetupMeshPasses(...)
{
    // 处理MeshPass相关的数据和标记.
    for (FDynamicPrimitive DynamicPrimitive : DynamicMeshElements.DynamicPrimitives)
    {
        for (int32 ElementIndex = DynamicPrimitive.StartElementIndex; 
             ElementIndex < DynamicPrimitive.EndElementIndex; ++ElementIndex)
        {// 这里会计算当前的MeshBatch会被哪些MeshPass引用,
         // 从而加到view的对应MeshPass的数组中.
            ComputeDynamicMeshRelevance(ShadingPath, bAddLightmapDensityCommands, 
                                      ViewRelevance, MeshBatch, View, 
                                      PassRelevance, PrimitiveSceneInfo, Bounds);
        }
    }
}
```
