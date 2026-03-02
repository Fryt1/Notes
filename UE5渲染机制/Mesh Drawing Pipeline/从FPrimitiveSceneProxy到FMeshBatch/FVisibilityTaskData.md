### 类职责

**动态网格元素收集的任务调度和结果管理**

### 关键成员

```
FDynamicMeshElements DynamicMeshElements;    // 最终动态网格元素容器
TArray<FVisibilityViewPacket> ViewPackets;   // 按视图分组的处理单元
FTaskEventArray Tasks;                       // 并行任务事件管理

```

### 成员在 MeshBatch 收集中的作用

- **DynamicMeshElements**：存储所有视图合并后的最终动态网格元素数据

- **ViewPackets**：将收集任务按视图进行分组，支持不同视图的并行处理

- **Tasks**：管理并行收集任务的同步和依赖关系

### 关键方法

```

void LaunchVisibilityTasks();    // 启动所有可见性相关任务
void SetupMeshPasses(...);       // 设置网格通道，处理收集结果
void Finish();                   // 完成所有任务处理
```
