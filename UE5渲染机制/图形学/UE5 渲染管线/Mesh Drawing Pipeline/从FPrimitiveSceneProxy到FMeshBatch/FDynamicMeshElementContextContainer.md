### 类职责

**并行动态网格元素收集器的管理和结果合并**

### 关键成员

```
TArray<FDynamicMeshElementContext*> Contexts; // 并行收集器实例数组
TArray<FRHICommandList*> CommandLists;       // 关联的RHI命令列表

```

### 成员在 MeshBatch 收集中的作用

- **Contexts**：多个并行工作的收集器实例，每个在独立线程中处理图元子集

- **CommandLists**：管理与每个收集器关联的 RHI 命令列表，用于后续提交

### 关键方法

```
void MergeContexts(TArray<FDynamicPrimitive, SceneRenderingAllocator>& OutDynamicPrimitives);
// 核心方法：合并所有并行收集器的结果到统一输出

void Submit(FRHICommandListImmediate& RHICmdList);
// 提交所有收集器的RHI命令到渲染线程
```
