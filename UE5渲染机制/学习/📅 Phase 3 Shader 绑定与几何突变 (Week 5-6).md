# 📖 学习任务 (Study Tasks)

- [x] **Shader 绑定：** 搜索宏 `IMPLEMENT_MATERIAL_SHADER_TYPE`，理解 C++ 类与 USF 文件的映射关系。
结论：**C++ 类和 USF 的映射，是在 IMPLEMENT_MATERIAL_SHADER_TYPE 这一行“注册”确定的。**

- [ ] **参数传递：** 学习 `FMeshDrawShaderBindings`，理解如何把 C++ 的 `FVector` 变成 Shader 里的 `uniform float3`。

- [ ] **图元类型：** 查找 `FMeshDrawCommand.PrimitiveType`，了解 `PT_TriangleList`, `PT_LineList`, `PT_PointList` 的区别。

## Shader 绑定
搜索宏 `IMPLEMENT_MATERIAL_SHADER_TYPE`，理解 C++ 类与 USF 文件的映射关系。

### 1.宏入口在 MaterialShaderType.h (line 13)，它本质转发到 IMPLEMENT_SHADER_TYPE(...)。

```
#define IMPLEMENT_MATERIAL_SHADER_TYPE(TemplatePrefix,ShaderClass,SourceFilename,FunctionName,Frequency) \

    IMPLEMENT_SHADER_TYPE( \

        TemplatePrefix, \

        ShaderClass, \

        SourceFilename, \

        FunctionName, \

        Frequency \

        );
```

### 2.IMPLEMENT_SHADER_TYPE 在 Shader.h (line 1724)，会创建 StaticType，把 ShaderClass + SourceFilename + FunctionName + Frequency 固化进 FShaderType，并通过 FShaderTypeRegistration 注册（Shader.h (line 1743)）。

用 shader 数据（SourceFilename，FunctionName，Frequency）通过==ShaderClass::ShaderTypeRegistration==注册
==FShaderTypeRegistration== 会先把各个 shader 的 GetStaticType() 延迟登记起来，再在引擎启动时统一构造注册这些 shader 类型，**供后续初始化与编译流程**使用。

```
/** A macro to implement a shader type. */

#define IMPLEMENT_SHADER_TYPE(TemplatePrefix,ShaderClass,SourceFilename,FunctionName,Frequency) \

    IMPLEMENT_UNREGISTERED_TEMPLATE_TYPE_LAYOUT(TemplatePrefix, ShaderClass); \

    TemplatePrefix \

    ShaderClass::ShaderMetaType& ShaderClass::GetStaticType() \

    { \

        static ShaderClass::ShaderMetaType StaticType( \

            ShaderClass::StaticGetTypeLayout(), \

            TEXT(#ShaderClass), \

            SourceFilename, \

            FunctionName, \

            Frequency, \

            ShaderClass::FPermutationDomain::PermutationCount, \

            SHADER_TYPE_VTABLE(ShaderClass), \

            sizeof(ShaderClass), \

            ShaderClass::GetRootParametersMetadata() \

            SHADER_TYPE_EDITOR_PERMUTATION_METADATA(ShaderClass) \

        ); \

        return StaticType; \

    } \

    TemplatePrefix FShaderTypeRegistration ShaderClass::ShaderTypeRegistration{TFunctionRef<::FShaderType&()>{ShaderClass::GetStaticType}};
```

### 3.实际编译时从类型里取回这两个关键字符串并喂给编译器：ShaderType->GetShaderFilename()、ShaderType->GetFunctionName()，见 MaterialShader.cpp (line 1865) 和 MeshMaterialShader.cpp (line 59)。

```
static void PrepareMaterialShaderCompileJob(EShaderPlatform Platform,

    EShaderPermutationFlags PermutationFlags,

    const FMaterial* Material,

    const FMaterialShaderMapId& ShaderMapId,

    FSharedShaderCompilerEnvironment* MaterialEnvironment,

    const FShaderPipelineType* ShaderPipeline,

    const FString& DebugGroupName,

    const FString& DebugDescription,

    const FString& DebugExtension,

    FShaderCompileJob* NewJob)

{

    const FShaderCompileJobKey& Key = NewJob->Key;

    const FMaterialShaderType* ShaderType = Key.ShaderType->AsMaterialShaderType();

  

    NewJob->bBypassCache = Material->IsPreview() || !Material->IsPersistent();

    NewJob->Input.SharedEnvironment = MaterialEnvironment;

    FShaderCompilerEnvironment& ShaderEnvironment = NewJob->Input.Environment;

  

    UE_LOG(LogShaders, Verbose, TEXT("          %s"), ShaderType->GetName());

  

    //update material shader stats

    UpdateMaterialShaderCompilingStats(Material);

  

    Material->SetupExtraCompilationSettings(NewJob->Input.ExtraSettings);

  

    // Allow the shader type to modify the compile environment.

    ShaderType->SetupCompileEnvironment(Platform, Material, Key.PermutationId, PermutationFlags, ShaderEnvironment);

  

    // Compile the shader environment passed in with the shader type's source code.

    ::GlobalBeginCompileShader(

        DebugGroupName,

        nullptr,

        ShaderType,

        ShaderPipeline,

        Key.PermutationId,

        ShaderType->GetShaderFilename(),

        ShaderType->GetFunctionName(),

        FShaderTarget(ShaderType->GetFrequency(), Platform),

        NewJob->Input,

        true,

        DebugDescription,

        DebugExtension

    );

}
```
