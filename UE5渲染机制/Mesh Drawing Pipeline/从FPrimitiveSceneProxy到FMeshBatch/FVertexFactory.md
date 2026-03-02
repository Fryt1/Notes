**职责**：定义顶点数据的格式、布局和获取方式

```
class FVertexFactory {
    // 顶点声明 - 描述顶点结构
    FRHIVertexDeclaration* VertexDeclaration;
    
    // 着色器参数 - 顶点着色器所需数据
    FVertexFactoryShaderParameters* ShaderParameters;
    
    // 数据类型标识
    EVertexElementType Type;
};
```
