# 04 排查、踩坑与 Shadow_VSM 收束

> 这一篇不是简单记“修掉了哪些 bug”，而是把这轮最值钱的调试方法固定下来。  
> 它现在要回答的是：  
> **当主视图、ShadowDepth、VSM、可见性、HZB、渲染线程等待和 GPU 成本缠在一起时，怎样把问题一层层剥开，而不是继续在错误层面上乱调。**

旧版本里最需要纠正的两件事是：

- 不能再把“`DrawStaticElements()` 置空”当成最终答案
- 不能再把“`ShadowCacheInvalidationBehavior = Always`”讲成无条件全量重绘的最终工程解

---

## 一、这轮排查里最重要的方法，不是调参数，而是先切层

你这次遇到的问题表面上看很乱：

- 主视图洞有时候已经对了
- 阴影可能不对
- draw call 很高
- `LaunchVisibilityTasks / VisibilityCommands` 很高
- 开 `HZB` 以后症状还会换桶
- 有时候渲染线程像在等 GPU

这类问题最容易掉进一个坑：

> 看到哪个桶高，就直接假设根因一定在那里。

但这次真正有效的方法，是先把它切成 4 个层次：

1. 主视图 correctness
2. 阴影 / VSM correctness
3. 渲染线程提交与可见性成本
4. GPU 真实 pass 成本

只要这一步做对，后面很多“看起来互相打架”的 profiler 现象都会变得好解释。

---

## 二、最稳的排查顺序是什么

### 第 1 步：先看主视图是否已经真的成立

重点看：

- `SceneDepth`
- `SceneColor`
- `RealityDistortionPass_PostLighting`

这一层只回答一个问题：

**主视图里 receiver 到底有没有真的被裁掉。**

如果这里都还没对，后面再看阴影没有意义。

### 第 2 步：再看 `ShadowDepth`

这一层回答的是：

**从光源视角看，第一个挡光表面是不是也变了。**

如果 `ShadowDepth` 都还没有开始正确 clip，那后面的 VSM 只是继续处理一份旧输入。

### 第 3 步：如果项目走的是 VSM，再继续看 invalidation 和 projection

重点看：

- `PrimitiveSceneProxy` 侧 invalidation 语义
- `ShadowScene`
- `VirtualShadowMapCacheManager`
- 最终 `ShadowMaskTexture`

这一步回答的是：

**是不是前面已经 clip 对了，但 VSM 后段还在复用旧页或旧语义。**

### 第 4 步：再看 RT 和 GPU 到底谁是真的瓶颈

这一步一定要结合着看：

- `stat unit`
- `stat gpu`
- `ProfileGPU`
- `stat initviews`

原因是：

- 渲染线程高，不一定只是 CPU 自己算得慢
- `VisibilityCommands` 高，也不一定说明纯粹是剔除算法本身有 bug
- `HZB / occlusion` 高，经常和上一帧 GPU 是否及时给出结果有关

---

## 三、这次最关键的 8 个坑

## 坑 1：看到“没效果”，第一反应就怀疑 shader 公式

这次一路排下来，最清楚的一条经验就是：

**“没效果”先怀疑路径，再怀疑公式。**

优先怀疑这些问题：

- primitive 根本没进这条 pass
- 当前项目没跑你以为的 feature path
- 缓存还在复用旧答案
- 当前高桶只是别的层在等待

---

## 坑 2：把 `RealityDistortionPass_PostLighting` 当成阴影主战场

它不是。

它负责的是：

- `HoleInfoTexture` 写完之后的最终主视图合成

它不直接负责：

- 光源视角挡光关系
- VSM 页缓存语义

所以如果地面阴影还是完整的，只盯着这一层往往会浪费很多时间。

---

## 坑 3：看到 `ShadowDepth` 已经 clip，就以为阴影一定对了

不够。

正确说法是：

- `ShadowDepth` 对了，说明前段开始成立
- 但 VSM 后面还有 cache、projection、final shadow factor

所以 `ShadowDepth` 正确只是必要条件，不是充分条件。

---

## 坑 4：把 `ShadowCacheInvalidationBehavior = Always` 理解成最终策略

旧版本这样理解还能勉强解释阶段性结果，但现在已经不对了。

当前正确理解是：

- component 侧用 `Always` 让 receiver 进入那条每帧会检查的集合
- proxy 侧在运行时根据 field 是否相交，动态返回：
  - 相交：`Always`
  - 不相交：`Static`

所以真正成立的不是：

> “所有 receiver 永远全量 invalidation”

而是：

> “通过引擎侧 override，把 invalidation 收束成条件式。”

---

## 坑 5：以为 invalidation 判定对了，draw call 自然就会掉

这次性能排查里，这是一个非常关键的认知转折点。

即使 invalidation 已经开始区分：

- 相交
- 不相交

如果 receiver 仍然走错误的动态提交结构，CPU 端的 draw call 和 mesh 收集成本还是会先炸。

这就是为什么：

- 只改 invalidation，不够
- 只改 `GetShadowCacheInvalidationBehavior()`，也不够

真正打中问题的一刀，是：

**receiver 回到静态提交路径。**

---

## 坑 6：继续把 `DrawStaticElements()` 置空

这一条在当前版本已经从“阶段性技巧”变成“错误结论”了。

原因很简单：

- 置空后，静态 shadow mesh 注册直接没了
- 不相交 receiver 失去静态缓存收益
- 阴影性能和 correctness 都更难稳定

现在正确做法是：

- `DrawStaticElements()` 调父类
- 让静态 shadow mesh 正常注册
- 再通过条件式 invalidation 决定哪些 receiver 真正重绘

---

## 坑 7：看到 `LaunchVisibilityTasks / VisibilityCommands` 高，就立刻断言是纯 CPU 问题

这次实际排查已经证明，这个桶不能这么粗暴看。

它确实经常和：

- 可见性任务调度
- HZB / occlusion
- 视图相关并行任务

有关，但还要继续结合看：

- `RHICmdList_Submit`
- `ProfileGPU`
- 开关 `HZB` 前后的瓶颈迁移

如果开关 `HZB` 后，瓶颈明显往 GPU 侧挪，那说明之前那部分“看起来像 CPU 的高桶”里，本来就夹着等待上一帧渲染结果的链式成本。

换句话说：

**它既可能是 CPU 真忙，也可能是 CPU 在等 GPU 相关结果；不能只看桶名下结论。**

---

## 坑 8：把 HZB 当成“开了就一定更快”

HZB / occlusion 的意义是：

- 用更多前期可见性工作，换更少后面的无意义渲染

所以它非常依赖当前场景的瓶颈位置。

如果原来是：

- GPU 很重
- 可见性工作还能帮你挡掉很多后续成本

那它会明显有收益。

但如果开了之后变成：

- Frame 还是高
- 只是高桶从 `VisibilityCommands` 换到 GPU 侧

那说明它不是白拿收益，而是在做：

**CPU / GPU 负担重新分配。**

---

## 四、Shadow / VSM 这轮最后到底是怎么收束的

如果把这轮收束压成最核心的几条，可以总结成下面这样。

### 1. 先确保 `ShadowDepth` 真的 clip 对了

也就是：

- receiver 进入阴影收集路径
- shader 真执行
- field 内片元真被裁掉

### 2. 再确保 VSM 后段不会继续使用旧答案

这一步靠的是：

- 引擎支持运行时 shadow cache invalidation override
- proxy 根据 field 相交关系动态返回 invalidation 行为

### 3. 同时必须让 receiver 回到静态提交路径

这是这一轮非常值钱的新结论。  
如果不做这一步，就算 correctness 对了，性能仍然可能卡死在：

- draw call
- shadow 收集
- 渲染线程提交

### 4. 当前正确概括已经变成

> receiver 常规渲染与阴影走静态提交路径；  
> `DrawStaticElements()` 负责保住静态 shadow mesh；  
> component 用 `Always` 作为进入检查集合的“入场券”；  
> proxy 再根据 field 相交关系把真正 invalidation 收束成条件式。

这才是现在这套 shadow / VSM 方案最准确的表达。

---

## 五、这轮性能排查和 shadow 排查是怎么互相咬合的

这次一个特别值钱的认知，是 finally 把两件事分开了：

### 1. correctness 问题

它问的是：

- 洞对不对
- 阴影对不对
- VSM 有没有复用旧结果

### 2. 性能问题

它问的是：

- draw call 为什么这么高
- `LaunchVisibilityTasks / VisibilityCommands` 为什么高
- 开 `HZB` 为什么会迁移瓶颈
- GPU 大头到底落在 `ShadowDepths / DeferredLighting / PostProcessing / TSR / Lumen` 哪些桶

如果这两层不分开，就会很容易出现：

- correctness 刚修对，就以为性能也会自然好
- 性能刚有改善，又误判成 correctness 被修坏了

这也是为什么这轮最后新增了：

- [[13 性能优化完整复盘：有效与无效路径]]
- [[14 黑块问题收束：Receiver 分类、高风险资产过滤与编辑器工具]]

因为到了当前阶段，shadow / VSM、性能、黑块内容问题已经值得分成 3 个独立主题讲。

---

## 六、这次最该带走的 6 个结论

### 结论 1：先切层，再改代码

不要一看到结果不对，就直接 patch 某个 shader。  
先问：

- 错的是主视图吗
- 错的是阴影吗
- 错的是 VSM 后段吗
- 错的是提交结构还是 GPU 成本

### 结论 2：路径问题比公式问题更常见

很多“没效果”最后根因都是：

- 路径没走到
- 当前 feature 没开
- 旧缓存还在

### 结论 3：`ShadowDepth` 对了，阴影不一定最终对

因为 VSM 还有后段。

### 结论 4：invalidation 判定对了，性能也不一定自然好

因为提交结构可能先炸。

### 结论 5：`HZB / VisibilityCommands` 不是只看桶名就能下结论

必须和 GPU、RHI 提交、瓶颈迁移一起看。

### 结论 6：当前正确的 shadow 收束方案，是“静态提交 + 条件式 invalidation”

不是旧文档里的：

- 动态主路径
- `DrawStaticElements()` 置空
- 无条件全量 Always

---

## 七、自问自答：现在怎么讲这轮 Shadow/VSM 收束才准确

### 问题 1：为什么 `ShadowDepth` 已经 clip 了，画面还是可能不对？

**回答：**

因为 `ShadowDepth` 只是前段。  
VSM 后面还有 cache、projection 和最终 `ShadowFactor`，它们都可能继续复用旧语义。

### 问题 2：为什么 `GetShadowCacheInvalidationBehavior()` 对了，draw call 还是可能高？

**回答：**

因为 invalidation 控制的是“哪些页需要重绘”，不是“CPU 端是不是还在走昂贵的提交路径”。  
如果 receiver 提交结构本身还错，draw call 一样会很高。

### 问题 3：为什么现在不能再讲“`DrawStaticElements()` 置空是关键点”？

**回答：**

因为当前版本已经依赖静态 shadow mesh 注册来吃到缓存收益。  
继续置空会直接把收益断掉。

### 问题 4：为什么 `Always` 现在不是旧意义上的 `Always`？

**回答：**

因为 component 侧的 `Always` 现在只是让 receiver 进入“每帧检查”的集合；真正是否 invalidation，由 proxy 在运行时按 field 相交关系决定。

### 问题 5：为什么开 `HZB` 后 frame 还是高，不代表它无效？

**回答：**

因为它可能只是把瓶颈从 CPU 可见性任务迁到了 GPU 真实渲染。  
这说明它在重新分配成本，不代表逻辑没生效。

### 问题 6：当前这轮最值钱的新认知是什么？

**回答：**

不是某个 cvar，而是 finally 把下面几件事分清了：

- correctness
- VSM 缓存语义
- 静态 / 动态提交路径
- 可见性任务
- GPU 真实 pass 成本

---

## 结论

这轮 `Shadow / VSM` 排查最有价值的部分，已经不只是“修对了某条阴影链”，而是你现在有了一套更成熟的 UE5 渲染排查方法：

**先切层，先分清 correctness 和性能，再判断当前高桶到底是在做真实工作，还是在等待另一层。**

而在当前版本里，关于 shadow / VSM 最准确的最终结论就是：

**receiver 需要回到静态提交路径，再通过条件式 invalidation 只重绘真正受 field 影响的阴影页。**
