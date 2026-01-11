### 类职责

**渲染流程入口和场景数据持有者**

### 关键成员


```
FVisibilityTaskData* VisibilityTaskData;  // 可见性任务数据协调器
TArray<FViewInfo> Views;                  // 渲染视图集合
FScene* Scene;                           // 场景数据容器

```
### 成员在MeshBatch收集中的作用

- **VisibilityTaskData**：协调整个动态网格元素的并行收集流程，管理任务分发和结果合并
    
- **Views**：定义需要收集动态网格元素的摄像机视角，每个视图对应不同的可见性要求和渲染目标
    
- **Scene**：提供场景中所有图元数据(FPrimitiveSceneInfo)，这些是生成FMeshBatch的源数据
    

### 关键方法


```
void BeginInitViews(FRDGBuilder& GraphBuilder, ...); 
// 启动视图初始化，开始网格收集流程
```