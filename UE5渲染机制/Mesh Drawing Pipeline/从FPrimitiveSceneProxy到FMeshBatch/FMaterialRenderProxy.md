**职责**：材质实例的运行时表示，管理材质状态和参数

```
class FMaterialRenderProxy {
    // 父级代理（用于材质继承）
    FMaterialRenderProxy* Parent;
    
    // 材质实例数据
    FMaterial* Material;
    
    // 渲染状态覆盖
    bool bSelected : 1;
    bool bHovered : 1;
    // ... 其他材质状态
};
```