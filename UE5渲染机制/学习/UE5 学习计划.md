
# 🚀 UE5 渲染管线特种兵突击方案：现实扭曲力场 (Reality Distortion Field)

项目代号： RealityDistortion

时间周期： 8 周 (2 个月)

核心目标： 脱离材质编辑器，深入 C++ 底层，通过修改 FMeshPassProcessor 和 FPrimitiveSceneProxy，实现一套自定义的几何变换渲染管线。

---

## ⚔️ 核心心法：尸检式学习 (The Autopsy Method)

不要试图通读几百万行源码。像法医一样去解剖引擎：

1. **定位 (Trace):** 用 RenderDoc 抓帧，找到你感兴趣的 DrawCall，反向推导它在 C++ 里是哪行代码提交的。
    
2. **破坏 (Break):** 找到代码后，打断点或注释掉它，看画面哪里坏了。“坏了”是你理解代码起作用的最好证明。
    
3. **模仿 (Mimic):** 复制引擎现有的类（如 `BasePassRendering`），改个名字，尝试跑通最简单的逻辑。
    

---

## 🛠️ Day 0: 前置准备 (必须完成)

- [x] **获取源码：** 从 GitHub 下载 UE5 源码 (推荐 5.3 或 5.4)，编译 `Development Editor` 模式。
    
- [x] **IDE 环境：** 安装 IDE，配置好 UE 开发插件。
    
- [x] **调试神器：** 安装 **RenderDoc** 并集成到 UE 插件中。
    
    
- [x] **项目创建：**
    
    - 新建 C++ 项目，命名为 `RealityDistortion`。
        
    - 勾选 `Starter Content` (初学者内容包)。
        
- [x] **调试技巧验证：** 确保 IDE 能够 `Attach to Process` 到 UE 编辑器，并在 C++ 代码中命中断点。
    

---

 ## 📅 Phase 1: 建立数据流认知 (Week 1-2)[[📅 Phase 1 建立数据流认知 (Week 1-2)]]

目标： 劫持从 Actor 到 Renderer 的数据传输通道。理解 GameThread 与 RenderThread 的隔离机制。

参考文档： 3.1.2 (渲染机制基础), 3.2.2 (从Proxy到MeshBatch)

### 📖 学习任务 (Study Tasks)
- [x] **源码尸检：** 打开 `Runtime/Engine/Private/StaticMeshSceneProxy.cpp`。
    
- [x] **关键函数：** 阅读 `FStaticMeshSceneProxy` 的构造函数。
    
- [x] **核心逻辑：** 深入阅读 `GetDynamicMeshElements`。思考它是如何把 LOD 资源塞进 `FMeshBatch` 的。
    
- [x] **断点追踪：** 在 `FScene::AddPrimitive` 打断点，拖入一个 Cube，观察 Call Stack，理解 Proxy 是何时被创建的。
    

### 💻 实战任务：数据劫持 (Coding Tasks)

- [x] **创建组件：** 新建 C++ 类 `UDistortionMeshComponent`，继承自 `UStaticMeshComponent`。
    
- [x] **创建代理：** 在组件内部新建类 `FDistortionSceneProxy`，继承自 `FStaticMeshSceneProxy`。
    
- [x] **重写接口：** 在组件中重写 `CreateSceneProxy()`，返回你的自定义 Proxy。
    
- [x] **实施劫持：**
    
    - 在 Proxy 中重写 `GetDynamicMeshElements`。
        
    - 复制父类代码，但在提交 `MeshBatch` 前，强行修改 `MeshBatch.MaterialRenderProxy`。
        
    - **Hack：** 在 C++ 中获取一个纯红色的材质，强制赋值给它。
        
- [x] **✅ 验收标准：** 场景中摆放石头模型的 `DistortionMeshComponent`，运行游戏后，模型变成了纯红色（完全绕过原本的材质设置）。
    

---

## 📅 Phase 2: 深入指令生成 (Week 3-4)[[📅 Phase 2 深入指令生成 (Week 3-4)]]

目标： 介入渲染管线的“决策层”，控制“画什么”和“怎么画”。

参考文档： 3.2.3 (从MeshBatch到MeshDrawCommand), 3.3.3 (静态绘制路径)

### 📖 学习任务 (Study Tasks)

- [x] **死磕核心：** 阅读 `Engine/Source/Runtime/Renderer/Private/BasePassRendering.cpp`。
    
	- [x] **搜索关键词：** `AddMeshBatch`。这是所有不透明物体渲染逻辑的入口。
    
- [x] **RenderDoc 逆向：** 截帧查看 Mesh 的 `Vertex Attributes`，回到源码寻找 `FMeshDrawCommand` 是在哪填充这些 Stream 的。
    

### 💻 实战任务：自定义 Processor (Coding Tasks)

- [ ] **注册 Pass：** 在 `MeshPassProcessor.h` (或你的模块) 中注册一个新的 `EMeshPass::Type`，命名为 `DistortionPass`。
    
- [ ] **编写处理器：** 复制 `FDepthPassMeshProcessor` (因为它最简单) 的代码，改名为 `FDistortionPassMeshProcessor`。
    
- [ ] **逻辑注入：**
    
    - 在 `AddMeshBatch` 函数中加入空间判断逻辑。
        
    - 定义一个全局变量（或从 Scene 传入）`FieldCenter`。
        
    - 计算 `PrimitiveSceneProxy->GetBounds().Origin` 到 `FieldCenter` 的距离。
        
    - **If (Distance < 500)**: 调用 `BuildMeshDrawCommands`；**Else**: return。
        
- [ ] **渲染插入：** 找到 `FSceneRenderer::Render`，模仿 `RenderCustomDepthPass`，在合适的位置调用你的 `DistortionPass`。
    
- [ ] **✅ 验收标准：** 用 RenderDoc 截帧，发现多了一个 Pass。当你移动“力场”位置时，该 Pass 内包含的 DrawCall 数量在动态变化。
    

---

## 📅 Phase 3: Shader 绑定与几何突变 (Week 5-6)

目标： 这里的代码将直接决定显卡如何绘制像素，实现“现实扭曲”的视觉效果。

参考文档： 3.2.3 (Shader绑定相关代码)

### 📖 学习任务 (Study Tasks)

- [ ] **Shader 绑定：** 搜索宏 `IMPLEMENT_MATERIAL_SHADER_TYPE`，理解 C++ 类与 USF 文件的映射关系。
    
- [ ] **参数传递：** 学习 `FMeshDrawShaderBindings`，理解如何把 C++ 的 `FVector` 变成 Shader 里的 `uniform float3`。
    
- [ ] **图元类型：** 查找 `FMeshDrawCommand.PrimitiveType`，了解 `PT_TriangleList`, `PT_LineList`, `PT_PointList` 的区别。
    

### 💻 实战任务：几何突变 (Coding Tasks)

- [ ] **编写 Shader：** 创建 `.usf` 文件。
    
    - **VS:** 简单的顶点变换 + 故障抖动 (`sin(Time * Speed)`)。
        
    - **PS:** 输出高亮青色 (`float4(0, 1, 1, 1)`).
        
- [ ] **C++ 绑定：** 在你的 `FDistortionPassMeshProcessor` 中，将这个 Shader 绑定到 Command 上。
    
- [ ] **参数传递：** 创建 Uniform Buffer，将“力场中心”和“当前时间”传给 Shader。
    
- [ ] **💥 高光时刻：** 在 `AddMeshBatch` 中，强行修改 `MeshDrawCommand.PrimitiveType` 为 `PT_LineList` (线框) 或 `PT_PointList` (点云)。
    
- [ ] **✅ 验收标准：** 游戏运行中，被“力场”扫过的物体瞬间瓦解，变成了抖动的线框或发光的点云，且带有 Bloom 辉光效果。
    

---

## 📅 Phase 4: 优化、合批与作品集 (Week 7-8)

目标： 从 Demo 走向工程，证明你的代码具备高性能特性。

参考文档： 3.4.1 (绘制管线优化技术)

### 📖 学习任务 (Study Tasks)

- [ ] **合批机制：** 深入理解 Instancing。阅读文档中关于 `MatchesForDynamicInstancing` 的条件（为什么 Uniform Buffer 不同会打断合批）。
    
- [ ] **性能分析：** 学习 RenderDoc 的 `Duration` 面板和 Overdraw 分析。
    

### 💻 实战任务：压力测试 (Coding Tasks)

- [ ] **压力场景：** 在场景中复制 5000 个 `DistortionMeshComponent` 球体。
    
- [ ] **合批调试：** 开启控制台 `r.MeshDrawCommands.LogDynamicInstancingStats 1`。
    
- [ ] **数据分析：** 检查 Log，确认你的 `DistortionPass` 是否将 5000 个球合并成了少量的 DrawCall。
    
- [ ] **Fix Bug:** 如果没合批，检查 Uniform Buffer 的更新频率，将其改为全局 Uniform 或 Scene Uniform。
    
- [ ] **作品集录制：**
    
    - 录制视频：力场扫过古代遗迹场景，建筑结构瓦解重组。
        
    - 技术截图：RenderDoc 抓帧证明 Pass 存在；C++ 核心代码高亮。
        

---

## ⏱️ 每日执行 Routine (高强度版)

_建议时间：周一至周五每晚 19:00 - 23:00_

1. **🔍 19:00 - 20:00 [定位]：**
    
    - 不做新功能。打开 AgentRansack 搜源码，对照文档看注释。
        
    - 目标：弄懂今天要写的那个类（比如 `FMeshPassProcessor`）的所有成员变量是干嘛的。
        
2. **💻 20:00 - 22:00 [破坏 & 模仿]：**
    
    - 写代码。这是最痛苦的时间，会伴随大量的编译错误和编辑器崩溃。
        
    - **技巧：** 多用 `UE_LOG`，少用断点（渲染线程断点容易卡死机器）。
        
3. **📝 22:00 - 23:00 [复盘]：**
    
    - 不要把 Bug 带到梦里。记录今天的 Call Stack。
        
    - 画一张简单的 UML 图，理清今天学到的类继承关系。
        

_建议时间：周日 [RenderDoc 日]_

- 不写代码。找一个复杂的场景，只用 RenderDoc 分析。尝试遮住 DrawCall 列表，只看画面，猜它是哪个 Pass 画的。
    

---

## 🆘 遇到困难怎么办？(Troubleshooting)

1. **编译太慢：**
    
    - **解法：** 使用 `Adaptive Unity Build`，或者只编译你修改的模块（右键项目 -> Build，而不是重新生成解决方案）。
        
2. **一运行就 Crash，没有 Log：**
    
    - **解法：** 不要直接双击 `.uproject` 启动。在 VS 里按 `F5` 启动，这样 Crash 时会直接停在出错的 C++ 代码行。
        
3. **看不懂 Shader 绑定宏：**
    
    - **解法：** 询问 AI。
        
    - _Prompt:_ "在 UE5 中，`IMPLEMENT_MATERIAL_SHADER_TYPE` 宏的具体展开是什么？它如何将 C++ 类 `FMyShader` 注册到全局 Shader Map 中？"
        

---

## 🤖 AI 辅助指令集 (Copy & Paste)

_遇到卡顿时，使用以下 Prompt 提问：_

- **Week 1 (Proxy):** "UE5 中 `FPrimitiveSceneProxy` 的生命周期是怎样的？如果我在 `UComponent` 里修改了参数，如何触发 `SceneProxy` 的重建？请解释 `MarkRenderStateDirty` 的作用。"
    
- **Week 3 (Processor):** "我要在 UE5 C++ 中实现一个新的 `FMeshPassProcessor`。请基于 UE 5.3 的标准，为我生成头文件和源文件的骨架代码。包含 `AddMeshBatch` 的重写声明，但不要写具体逻辑。"
    
- **Week 5 (Shader):** "如何编写一个 UE5 Vertex Shader 让顶点沿法线偏移？请给出 `.usf` 代码示例以及 C++ 端 `FMeshMaterialShader` 的绑定代码。"
    

---

### 🛡️ 学习任务验收标准：三位一体 (The Trinity Protocol)

_对于任何一个关键类或函数（如 `GetDynamicMeshElements`），必须完成以下三项才算“任务完成”。拒绝无效阅读，只求实战掌握。_

#### 1. 画出地图 (The Map)

- **目标：** 理清数据流向。
    
- **执行：** 在白板/纸上画出草图，必须包含箭头。
    
- **标准：** 能清晰标出 **Input**（原材料，如 `RenderData`）和 **Output**（产成品，如 `MeshBatch` -> `Collector`）。
    

#### 2. 搞崩引擎 (The Crash)

- **目标：** 验证核心代码行的作用。
    
- **执行：** 找到函数中最关键的一行（例如 `Collector.AddMesh` 或材质赋值），将其注释掉或传入 `nullptr`。
    
- **标准：** 必须亲眼看到引擎 **Crash** 或 **画面出现预期异常**（如模型消失、变黑）。只有能破坏它，才算掌控它。
    

#### 3. 盲写骨架 (The Skeleton) —— _核心考核_

- **目标：** 将 C++ 翻译为逻辑语言，彻底内化。
    
- **规则：** 关闭所有源码，新建一个空文本文件，凭记忆写出伪代码。
    
- **不考什么：** 变量类型定义、分号、指针语法、API 确切拼写。
    
- **只考什么：** 关键对象的交互顺序、核心动词。
    

**✅ 盲写合格标准范例 (以 `GetDynamicMeshElements` 为例)：**

> _只要你能默写出下面这个结构，即便单词拼错，也算满分通过。_

C++

```
// 盲写目标：逻辑伪代码
void GetDynamicMeshElements(Collector) // 1. 知道参数里有收集器
{
    // 2. 遍历视野 (可能有多个相机)
    for (View in Views)
    {
        // 3. [动作A] 申请：向收集器要一张空白单子
        var Mesh = Collector.AllocateMesh();

        // 4. [动作B] 填空：填入形状 (LOD资源)
        Mesh.VertexFactory = RenderData.LODs[0].VertexFactory;

        // 5. [动作C] 填空：填入材质 (这是劫持点！)
        Mesh.Material = MyMaterial; 

        // 6. [动作D] 提交：把单子交上去
        Collector.AddMesh(Mesh);
    }
}
```

**🧠 盲写自测口诀 (能回答这4个问题即为通过)：**

1. **问：** 我向谁申请内存？ -> **答：** `Collector`
    
2. **问：** 申请的动作叫什么？ -> **答：** `Allocate`
    
3. **问：** 哪一步决定了模型长什么样和什么颜色？ -> **答：** 赋值 `VertexFactory` 和 `Material`
    
4. **问：** 最后怎么提交？ -> **答：** `AddMesh`
    

---

### 📋 构造函数验收标准：清单协议 (The Checklist Protocol)

_针对 `FStaticMeshSceneProxy` 构造函数等初始化逻辑，重点在于理解数据的来源与所有权转移。_

#### 1. 溯源 (Trace the Source)

- **目标：** 搞清楚渲染线程用的数据，最初是从哪里来的。
    
- **执行：** 找到关键成员变量（如 `RenderData`）的赋值语句。
    
- **标准：** 能够指出该数据是来自 **Component（组件本身）** 还是 **Resource（如 UStaticMesh 资源）**。
    
    - _为什么重要？_ 来自 Component 的数据容易变（如材质参数），来自 Resource 的数据一般不变（如顶点）。
        

#### 2. 划界 (Define the Boundary)

- **目标：** 搞清楚数据是 **“拷贝（Copy）”** 还是 **“引用（Reference）”**。
    
- **执行：** 观察赋值方式。是 `this->Val = InVal` (拷贝) 还是指针/引用？
    
- **标准：** 必须能回答：“如果我在游戏运行中修改了源数据，Proxy 里的这份会立马跟着变吗？”
    
    - _如果是拷贝：_ 不会变（它是快照）。
        
    - _如果是引用：_ 可能会变（共享内存）。
        

---

### 📝 任务记录模板 (Copy & Paste)

_在你的文档里，对于构造函数的阅读任务，填好这张表就算过关：_

> **任务：分析 `FStaticMeshSceneProxy` 构造函数**
> 
> 1. **[核心数据源]**
>     
>     - **RenderData (顶点):** 来源是 `UStaticMesh` 资源。_(类型：引用/共享指针)_
>         
>     - **Material (材质):** 来源是 `UStaticMeshComponent` 的覆写设置。_(类型：拷贝/快照)_
>         
> 2. **[线程安全性]**
>     
>     - **结论：** 这是一个**快照 (Snapshot)**。
>         
>     - **证据：** 构造函数是在 **GameThread** 执行的，执行完后 Proxy 就被甩给渲染线程了。之后 Component 怎么变，Proxy 都不知道，除非重建 Proxy。
>         
> 3. **[一句话总结]**
>     
>     - 它把游戏线程的 `Component` 数据“冻结”并打包，生成了一个独立的渲染代理。
>         

---

### 🧠 自测口诀 (能回答这 2 个问题即为通过)

1. **问：** 这个 Proxy 里的数据（比如顶点），是它自己独占一份内存，还是大家共用一份？
    
    - _（针对 StaticMeshProxy，答案通常是：LOD资源是共用的，但材质设置是自己独占的。）_
        
2. **问：** 如果我现在的 Actor 瞬间移动了，需要重新运行这个构造函数吗？
    
    - _（答案：不需要。移动只更新 Transform 矩阵，不需要重建 Proxy。但如果是换了模型或者材质，就需要重建。）_
        

---

