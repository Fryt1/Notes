### 类职责

**网格批次的分配、存储和视图组织**

### 关键成员


```
TChunkedArray<FMeshBatch> MeshBatchStorage;  // 网格批次存储池
TArray<TArray<FMeshBatchAndRelevance>*> MeshBatches; // 按视图组织的批次
FGlobalDynamicIndexBuffer& DynamicIndexBuffer; // 动态索引缓冲区
FGlobalDynamicVertexBuffer& DynamicVertexBuffer; // 动态顶点缓冲区
```

### 成员在MeshBatch收集中的作用

- **MeshBatchStorage**：预分配的FMeshBatch对象池，避免频繁内存分配
    
- **MeshBatches**：将收集的网格批次按目标视图进行分类组织
    
- **DynamicIndex/VertexBuffer**：为动态图元提供顶点和索引数据存储
    

### 关键方法

```
FMeshBatch& AllocateMesh();  // 从存储池分配新的FMeshBatch
void AddMesh(int32 ViewIndex, FMeshBatch& Mesh); // 添加网格到指定视图
```