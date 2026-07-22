# 14 黑块问题收束：Receiver 分类、高风险资产过滤与编辑器工具

> 这一篇不是在继续追问“是不是某一行 shader 又写错了”，而是把这次真正收束问题的关键认知单独固定下来。  
> 这次最值钱的结论不是“又修了一个黑块 bug”，而是：  
> **不是所有被转换出来的 receiver 都适合继续充当通用 receiver。某些资产从内容结构上就属于高风险对象，应该被分类、跳过或手动排除。**

---

## 一、这次问题最后是怎么收束下来的

最开始表面症状是：

- 一部分物体能正常出现 RealityDistortion 的半透明裂隙效果
- 另一部分物体在裂隙区域会变黑、变条纹，或者透不过去
- 问题在性能优化之后变得更明显，但不是所有物体都一起坏

真正把问题范围收紧的，不是继续盲改 composite，而是几组非常关键的 A/B 证据：

### 1. `r.RealityDistortion.BasePassDisableFields 1`

这个开关打开后：

- 原来发黑的物体不再发黑
- 但同时也直接没有了 RealityDistortion 效果

这说明：

- 问题不是普通材质本身坏了
- 而是 receiver 参与 RD 字段裁切之后，某条后续链路拿到了错误的“后景”

### 2. `r.RealityDistortion.CompositeDebugMode 1`

这个模式下问题区域是纯绿的。

这说明：

- `HoleInfo` 本身是有写进去的
- 裂隙命中判断不是主因
- 问题不在“洞有没有被标出来”，而在“洞里最后采到了什么”

### 3. `SceneDepth` 观察

问题物体上，`SceneDepth` 里仍然能看到伞面、地面这类物体自己的轮廓；而那些正常透过去的物体，在被剔除后能看到后面的深度。

这说明：

- 这些异常物体并不是简单的“前面一层被裁掉，后面自然就是背景”
- 对它们来说，前层后面仍然可能是它自己的内层、背层、相邻层，或者另一个 receiver 层
- 所以 composite 虽然想“透过去”，但取到的并不是真正的远处背景

### 4. `r.RealityDistortion.SkipReceiverDepthOnly 1`

这个测试最有价值的地方在于：

- 大部分物体开了之后，确实能透过去
- 只有少数物体依然黑、依然异常

这一步直接把问题性质改写成了：

> 这不是“整个系统都错了”，而是“有一小批资产不适合继续当通用 receiver”。

---

## 二、最后的结论为什么不是“继续修 shader”，而是“做 receiver 分类”

到这一步以后，问题已经不再像：

- `Composite` 数学公式写错
- 半透明混合错
- 某个 pass 根本没跑

更准确的说法应该是：

**当前这套 receiver 语义默认假设：当前像素前层被裁掉以后，后面会是合理背景。**  
但有些资产天然不满足这个假设。

典型高风险对象包括：

- 大型地面、路面、地砖、超大平面
- 伞面、棚布、遮阳结构
- 叶片卡片、树叶、草、灌木、植被类
- `Masked + TwoSided` 的双面卡片结构
- 多层壳体、近距离自遮挡明显的特殊结构

这些对象的问题不是“不能画”，而是：

- 它们的前后层关系太复杂
- 它们并不一定存在一个稳定、单义的“真正背景”
- 被 generic receiver 逻辑一刀切进去之后，就很容易在裂隙区出现黑块、条纹、假透视

所以这次收束后的正确方向不是：

- 继续把所有资产都强行塞进 receiver
- 再去用更多 shader patch 兜底

而是：

- **先承认 receiver 需要内容分类**
- **先把明显高风险对象从通用 receiver 体系里排除出去**

---

## 三、这次“自动高风险资产”到底做了什么

这里一定要讲清楚，避免以后自己误会。

### 它不是“自动帮你选中场景里的高风险对象”

当前实现不是一个“扫描全场景然后帮你自动高亮选中”的工具。  
它做的是另一件更保守、也更稳的事：

> **当你执行“转换成 receiver”时，系统会先对候选对象做一轮高风险判定；命中规则的对象会被自动跳过，不参与转换。**

也就是说，它是：

- 自动过滤
- 自动跳过
- 自动记录日志

而不是：

- 自动选中
- 自动改材质
- 自动强行修复

### 过滤逻辑入口

核心逻辑放在：

- `Source/RealityDistortion/Rendering/DistortionEditorConversion.cpp`

其中最关键的函数是：

- `EvaluateReceiverConversionFilter(...)`

这一步会对每个候选 `StaticMeshActor / DistortionReceiverActor` 做启发式判断。

---

## 四、当前高风险过滤具体按什么规则判

这套规则现在是“保守优先”的。

### 1. 手动覆盖标签优先级最高

支持两个显式标签：

- `RD_SkipReceiverConversion`
- `RD_ForceReceiverConversion`

语义是：

- 只要打了 `RD_SkipReceiverConversion`，就直接跳过
- 只要打了 `RD_ForceReceiverConversion`，就强制允许，不再走后面的自动过滤

这一步是为了防止自动规则过严或过松时，没有人工兜底口子。

### 2. 材质类型过滤

现在会优先排除：

- 任何非 `Opaque / Masked` 的材质
- `Masked + TwoSided` 的材质

这样做的原因很明确：

- 非不透明材质本来就不适合再套这套“前层裁掉、取后景”的 receiver 语义
- `Masked + TwoSided` 往往就是树叶卡片、草、灌木、薄片结构的高风险来源

这一步其实就是把“最容易黑、最容易出现层间歧义”的内容先挡掉。

### 3. 几何外形过滤

现在会检查包围盒：

- `Max(X, Y) >= 1500`
- `Min(X, Y, Z) <= 20`

如果同时成立，就把它判成：

- `Large flat surface`

这类对象本质上就是：

- 大而薄
- 覆盖范围广
- 很容易同时承担前景、地面、承载面等多重语义

它们拿来做 generic receiver，风险本来就很高。

### 4. 名字 / 路径关键词过滤

当前还会把下面这些文本拼起来做关键字搜索：

- Actor Label
- Actor Path
- Component Path
- StaticMesh Path

命中这些关键词时会直接判成高风险：

- `ombrellone`
- `umbrella`
- `strada`
- `road`
- `street`
- `marciapiede`
- `sidewalk`
- `ground`
- `floor`
- `tree`
- `leaf`
- `leaves`
- `foliage`
- `grass`
- `bush`
- `plant`
- `hedge`

这一步的本质不是“名字里带这个词就一定坏”，而是：

- 把这次已经被验证过高风险的资产类型先做成保守拦截
- 避免以后批量转换时又把整批明显危险的内容一次性塞进去

---

## 五、为什么这套过滤不是“投机取巧”，而是正确工程化方向

因为这次已经验证过了：

- 问题不是所有 receiver 都黑
- 问题不是所有材质都黑
- 问题不是整个 composite 链路统一失效

而是：

- 某些特殊资产的内容结构不符合当前 receiver 语义假设

这就意味着：

**receiver 不是一个“对任何静态网格都无差别适用”的通用标签。**  
它更像一种需要内容准入条件的渲染身份。

所以工程上最合理的做法就是：

1. 默认允许普通稳定实体网格进入 receiver
2. 默认把高风险结构排除在外
3. 给人工保留强制允许 / 强制跳过的覆盖入口

这不是偷懒，而是在做真正的内容分类。

---

## 六、这次补进去的编辑器工具有什么用

除了“转换时自动跳过高风险资产”，这次还补了两组非常实用的排查工具。

### 1. 批量禁用选中 receiver

现在可以直接把选中的 receiver 关掉：

- 右键菜单：`Disable Distortion Receiver`
- `DistortionFieldActor` 的 `CallInEditor` 按钮
- 控制台命令：`RealityDistortion.DisableSelectedReceivers`

它做的事情很直接：

- 把 `bEnableDistortionReceiver` 设为 `false`
- 刷新 CPD 标记
- 刷新渲染状态

它不会：

- 把 actor 转回普通 `StaticMeshActor`
- 删除对象
- 破坏当前内容结构

这非常适合做“先把问题对象排除出 receiver，验证症状是否消失”的诊断。

### 2. 批量重新启用选中 receiver

同样补了反向操作：

- 右键菜单：`Enable Distortion Receiver`
- `DistortionFieldActor` 的 `CallInEditor` 按钮
- 控制台命令：`RealityDistortion.EnableSelectedReceivers`

这意味着：

- 你现在可以非常低成本地来回 A/B
- 不需要每次重新转换 actor
- 也不需要手工改回普通组件

### 3. 工具代码入口

这部分主要落在：

- `Source/RealityDistortion/Rendering/DistortionEditorConversion.h`
- `Source/RealityDistortion/Rendering/DistortionEditorConversion.cpp`
- `Source/RealityDistortion/Rendering/DistortionFieldActor.h`
- `Source/RealityDistortion/Rendering/DistortionFieldActor.cpp`
- `Source/RealityDistortion/RealityDistortion.cpp`

---

## 七、现在推荐的实际工作流

如果以后继续扩这套系统，当前最稳的工作流应该是：

### 1. 批量转换时先走保守过滤

让普通稳定实体先进入 receiver；  
高风险资产默认跳过。

### 2. 如果某个对象看起来应该能进，但被过滤了

给它打：

- `RD_ForceReceiverConversion`

然后再转换。

### 3. 如果某个对象已经转了，但在效果里发黑、出条纹、透不过去

先不要急着改 shader。  
先直接：

- `Disable Distortion Receiver`

如果症状立刻消失，就说明它大概率属于不适合做 generic receiver 的内容。

### 4. 如果某类对象长期都不适合

那就应该把它明确归类成：

- 永久不进 receiver 的内容类型

而不是每次都再试一次、再踩一次同样的坑。

---

## 八、这一轮真正留下来的认知升级是什么

这次最值钱的收获不是又加了两个菜单项，而是下面这句话：

> `RealityDistortion` 的 receiver 不是一个“只要是 StaticMesh 就都能挂上去”的通用身份，而是一种需要内容语义约束的渲染角色。

这句话背后对应的工程现实是：

- 性能优化会把更多路径变稳定，也会把原来被遮住的内容问题暴露出来
- 当系统逐渐从“能跑”进入“稳定可用”阶段时，内容分类就会自然成为下一层瓶颈
- 真正成熟的系统，不是无限兜底所有资产，而是明确哪些资产类型属于准入范围

也就是说，这一轮不是在“承认修不动”，而是在把系统从：

- 单纯效果实现

往：

- **带内容约束、带准入规则、带排查工具的可用系统**

推进了一步。

---

## 九、一句话总结

这次黑块问题最后能收束，不是因为继续对着 `Composite` 或 `BasePass` 猛改，而是因为我们最终确认了一件更重要的事：

**问题不在于所有 receiver 都错了，而在于一小批特殊资产根本不应该被当成通用 receiver。**  
自动高风险过滤、手动启停 receiver、显式准入标签，这三件事一起，才让这套系统真正开始从“能做效果”走向“能稳定落地”。
