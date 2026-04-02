# Day 5 — RenderPass 与 Subpass

> 日期：2026-04-02  
> 目标：理解 VkRenderPass 结构，理解 Subpass 在 TBDR 上的优化意义，分析 Filament 的 RenderPass 实现

---

## 一、VkRenderPass 结构

```
VkRenderPassCreateInfo
 ├── VkAttachmentDescription[]   描述每个附件的格式、load/store op、初始/最终 layout
 ├── VkSubpassDescription[]      每个子通道引用哪些附件，如何读写
 └── VkSubpassDependency[]       子通道之间的执行与内存依赖
```

### 1.1 VkAttachmentDescription

| 字段 | 含义 | 移动端优化原则 |
|------|------|--------------|
| `format` | 图像格式 | |
| `samples` | MSAA 采样数 | |
| `loadOp` | RenderPass 开始时如何处理附件内容 | 优先 `CLEAR` 或 `DONT_CARE`，避免 `LOAD`（会从主存读回 tile） |
| `storeOp` | RenderPass 结束时是否写回主存 | 深度/模板通常用 `DONT_CARE`（On-Chip 数据不需要保留） |
| `initialLayout` | 开始时图像的 layout | `UNDEFINED` 配合 `DONT_CARE`，驱动知道不用读旧内容 |
| `finalLayout` | 结束时图像的 layout | 按后续用途：`SHADER_READ_ONLY_OPTIMAL`、`PRESENT_SRC_KHR` 等 |

### 1.2 VkSubpassDescription

```cpp
VkSubpassDescription subpass{};
subpass.pipelineBindPoint       = VK_PIPELINE_BIND_POINT_GRAPHICS;
subpass.colorAttachmentCount    = 1;
subpass.pColorAttachments       = &colorRef;      // 输出到哪些附件
subpass.pDepthStencilAttachment = &depthRef;      // 深度附件
subpass.inputAttachmentCount    = 1;
subpass.pInputAttachments       = &inputRef;      // 从上一个 subpass 读取的附件
```

`pInputAttachments` 中的附件在 Fragment Shader 中通过 `subpassLoad()` 读取，只能访问当前像素位置（不能偏移采样），但不经过主存。

### 1.3 VkSubpassDependency

```cpp
VkSubpassDependency dep{};
dep.srcSubpass      = ?;   // 生产者 subpass 索引
dep.dstSubpass      = ?;   // 消费者 subpass 索引
dep.srcStageMask    = ?;   // 生产者在哪个阶段完成写入
dep.dstStageMask    = ?;   // 消费者在哪个阶段开始读取
dep.srcAccessMask   = ?;   // 生产者的访问类型
dep.dstAccessMask   = ?;   // 消费者的访问类型
dep.dependencyFlags = ?;   // 依赖范围
```

> **规律：src 描述"谁写完了"，dst 描述"谁要开始读"，Stage 比 Access 更粗粒度，两者配合才能精确描述同步点。**

#### srcSubpass / dstSubpass

| 填什么 | 含义 |
|--------|------|
| `0, 1, 2...` | 具体的 Subpass 索引 |
| `VK_SUBPASS_EXTERNAL` | 表示 RenderPass 之外（外部依赖） |

```cpp
dep.srcSubpass = VK_SUBPASS_EXTERNAL;  dep.dstSubpass = 0;  // 外部 → Subpass 0
dep.srcSubpass = 1;  dep.dstSubpass = VK_SUBPASS_EXTERNAL;  // Subpass 1 → 外部
dep.srcSubpass = 0;  dep.dstSubpass = 1;                    // 内部传递（Filament 用的）
```

#### srcStageMask / dstStageMask 常用值

| 值 | 含义 |
|----|------|
| `VERTEX_SHADER_BIT` | 顶点着色器 |
| `FRAGMENT_SHADER_BIT` | 片元着色器 |
| `EARLY_FRAGMENT_TESTS_BIT` | 深度/模板测试（Early） |
| `LATE_FRAGMENT_TESTS_BIT` | 深度/模板测试（Late） |
| `COLOR_ATTACHMENT_OUTPUT_BIT` | 颜色附件写入 |
| `COMPUTE_SHADER_BIT` | Compute Shader |
| `TRANSFER_BIT` | 拷贝/Blit 操作 |
| `ALL_GRAPHICS_BIT` | 所有图形阶段（保守但慢） |
| `TOP_OF_PIPE_BIT / BOTTOM_OF_PIPE_BIT` | 流水线起点/末尾（常配合 EXTERNAL） |

#### srcAccessMask / dstAccessMask 常用值

| 值 | 含义 |
|----|------|
| `COLOR_ATTACHMENT_READ_BIT` | 读颜色附件 |
| `COLOR_ATTACHMENT_WRITE_BIT` | 写颜色附件 |
| `DEPTH_STENCIL_ATTACHMENT_READ_BIT` | 读深度/模板 |
| `DEPTH_STENCIL_ATTACHMENT_WRITE_BIT` | 写深度/模板 |
| `SHADER_READ_BIT` | Shader 读（纹理、SSBO、Input Attachment） |
| `SHADER_WRITE_BIT` | Shader 写（SSBO、Image Store） |
| `INPUT_ATTACHMENT_READ_BIT` | 专门用于 subpassLoad() 读取 |
| `TRANSFER_READ_BIT / TRANSFER_WRITE_BIT` | 拷贝操作读/写 |

#### dependencyFlags

| 值 | 含义 |
|----|------|
| `0` | 全局依赖，整帧所有 Tile 都等待 |
| `VK_DEPENDENCY_BY_REGION_BIT` | Tile 级别依赖，TBDR 优化关键 |
| `VK_DEPENDENCY_VIEW_LOCAL_BIT` | Multiview 渲染时使用 |
| `VK_DEPENDENCY_DEVICE_GROUP_BIT` | 多 GPU 时使用 |

`VK_DEPENDENCY_BY_REGION_BIT`：依赖只在同一 Tile 内成立，不同 Tile 可以并行处理。没有这个 flag，驱动会等整帧所有 Tile 的 Subpass 0 全部完成才开始 Subpass 1，TBDR 并行被完全打断。

#### 常见组合

```cpp
// G-Buffer → Deferred Lighting（Filament 的用法）
srcSubpass    = 0;
dstSubpass    = 1;
srcStageMask  = COLOR_ATTACHMENT_OUTPUT_BIT;
dstStageMask  = FRAGMENT_SHADER_BIT;
srcAccessMask = COLOR_ATTACHMENT_WRITE_BIT;
dstAccessMask = INPUT_ATTACHMENT_READ_BIT;   // subpassLoad() 专用
dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;

// 深度写入 → RenderPass 结束后作为纹理采样
srcSubpass    = 0;
dstSubpass    = VK_SUBPASS_EXTERNAL;
srcStageMask  = LATE_FRAGMENT_TESTS_BIT;
dstStageMask  = FRAGMENT_SHADER_BIT;
srcAccessMask = DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;
dstAccessMask = SHADER_READ_BIT;
dependencyFlags = 0;
```

---

## 二、Subpass 在 TBDR 上的价值

### 2.1 没有 Subpass 的 Deferred Rendering（主存往返）

```
Pass A (G-Buffer)
  → Fragment Shader 写入 G-Buffer 颜色附件
  → storeOp = STORE → 写回主存 DRAM   ← 带宽消耗

Pass B (Deferred Lighting)
  → loadOp = LOAD  → 从主存 DRAM 读取  ← 带宽消耗
  → Fragment Shader 读取 G-Buffer，计算光照
  → storeOp = STORE → 最终颜色写回主存
```

每个 Tile 需要两次主存往返，带宽消耗是 Deferred 在移动端的主要瓶颈。

### 2.2 使用 Subpass（On-Chip 传递）

```
RenderPass
  Subpass 0 (G-Buffer)
    → Fragment Shader 写入 G-Buffer，数据留在 On-Chip Tile Memory

  Subpass Dependency（by region）
    → 当前 Tile 的 Subpass 0 完成后即可进入 Subpass 1，不等待其他 Tile

  Subpass 1 (Deferred Lighting)
    → subpassLoad(inputAttachment) 直接读取当前 Tile 的 G-Buffer（On-Chip）
    → G-Buffer 的 storeOp = DONT_CARE（不写回主存）
    → 最终颜色 storeOp = STORE（只写一次主存）
```

**优化结果：**
- G-Buffer 数据从不落地主存（既不写也不读）
- 主存只承受最终颜色的一次写操作
- 对于 4 个 G-Buffer Attachment，节省 4 × 2（写+读）= 8 次主存带宽

### 2.3 Subpass 的限制

- `subpassLoad()` 只能读当前 fragment 对应的像素（不能偏移），适合 Deferred Lighting 但不适合 Blur、SSAO、TAA 等需要邻近像素的算法
- 多个 subpass 共享同一个 Framebuffer 尺寸，不能中途改变分辨率
- 多 subpass 增加 RenderPass 配置复杂度，调试和兼容性验证更繁琐

---

## 三、补充概念

### 3.0 Image Layout 是什么

同一张图在不同用途下，GPU 内部的数据排列方式不同：

| Layout | 适合的用途 |
|--------|-----------|
| `COLOR_ATTACHMENT_OPTIMAL` | 作为渲染目标写入 |
| `SHADER_READ_ONLY_OPTIMAL` | 作为纹理采样读取 |
| `DEPTH_ATTACHMENT_OPTIMAL` | 深度测试写入 |
| `TRANSFER_SRC_OPTIMAL` | blit/copy 的源 |
| `UNDEFINED` | 不关心内容（配合 DONT_CARE） |

用错 Layout 轻则性能下降，重则 Validation 报错或结果错误。

**典型的 layout 转换流程：**
```
上一帧结束：颜色附件被 Shader 采样 → SHADER_READ_ONLY_OPTIMAL
               ↓
本帧开始：要把这张图作为渲染目标 → 必须 transition → COLOR_ATTACHMENT_OPTIMAL
               ↓
RenderPass 渲染完成
               ↓
结束后再 transition → SHADER_READ_ONLY_OPTIMAL（供下一帧采样）
```

`emitBarriersBeginRenderPass()` 就是统一处理这个转换。

---

### 3.1 Primary vs Secondary CommandBuffer

| 类型 | 说明 |
|------|------|
| Primary CB | 可以直接提交给 Queue，可以调用 `vkCmdBeginRenderPass` |
| Secondary CB | 不能直接提交，只能被 Primary 调用，用于 RenderPass 内录制命令 |

**Secondary CB 解决的问题：** 单线程录制 DrawCall 是 CPU 瓶颈，多线程并行录制 Secondary CB 可以把时间缩短 N 倍。

```
线程1：录制物体 1-2500    → Secondary CB #1  ┐
线程2：录制物体 2501-5000 → Secondary CB #2  ├ 并行，时间缩短4倍
线程3：录制物体 5001-7500 → Secondary CB #3  │
线程4：录制物体 7501-10000→ Secondary CB #4  ┘
           ↓
主线程 Primary CB：
  vkCmdBeginRenderPass(..., VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS)
  vkCmdExecuteCommands(cb1, cb2, cb3, cb4)
  vkCmdEndRenderPass()
```

**两个 Contents 模式不能混用：**
```cpp
// INLINE 模式：DrawCall 直接写在 Primary CB
vkCmdBeginRenderPass(..., VK_SUBPASS_CONTENTS_INLINE);
vkCmdDraw(...);   // 合法

// SECONDARY 模式：Primary CB 只能调度，不能直接录制
vkCmdBeginRenderPass(..., VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS);
vkCmdDraw(...);            // 非法！Validation 报错
vkCmdExecuteCommands(...); // 只能这样
```

**Secondary CB 录制时必须声明继承信息：**
```cpp
VkCommandBufferInheritanceInfo inheritance{};
inheritance.renderPass  = renderPass;  // 会在哪个 RenderPass 里执行
inheritance.subpass     = 0;           // 第几个 Subpass

VkCommandBufferBeginInfo beginInfo{};
beginInfo.flags = VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT; // 关键
beginInfo.pInheritanceInfo = &inheritance;
```

**Secondary CB 的额外代价：**

| 开销 | 说明 |
|------|------|
| 合并开销 | `vkCmdExecuteCommands` 本身有 CPU 开销 |
| 状态继承复杂 | 需要继承 Viewport、Scissor 等动态状态 |
| 驱动差异 | 部分移动端驱动对 Secondary CB 优化不足，反而更慢 |

**Filament 不用 Secondary CB 的原因：**
1. 单线程录制，不需要多线程
2. GPU Driven 改造后 DrawCall 大幅减少，CPU 录制根本不是瓶颈
3. 移动端驱动兼容性问题

**和 GPU Driven 的关系：**
```
传统渲染：CPU 录制 10000 个 DrawCall → Secondary CB 多线程有意义
GPU Driven：CPU 只录制 1 个 vkCmdDrawIndexedIndirect → 根本不需要多线程录制
```
GPU Driven 从根本上消除了 Secondary CB 的使用动机。

---

### 3.2 Scissor（裁剪矩形）

只有落在 Scissor 矩形内的像素才会被渲染，矩形外直接丢弃，不执行 Fragment Shader。

| | Viewport | Scissor |
|---|---|---|
| 作用 | 把 NDC 坐标映射到屏幕坐标 | 限制哪个矩形区域内的像素可以被写入 |
| 影响阶段 | 顶点变换之后 | Fragment 写入之前 |
| 类比 | 摄像机取景框 | 取景框上再贴一张遮罩 |

Filament 每个 RenderPass 开始时重置 Scissor 为整个 RenderTarget 大小，防止上一个 Pass 的小 Scissor 状态污染当前 Pass。

---

### 3.3 为什么 vkCreateRenderPass 很重

调用 `vkCreateRenderPass` 时驱动要做真正的编译工作：

1. **验证合法性**：检查 Attachment 格式组合、Subpass 依赖是否有环、Layout transition 是否合理
2. **生成状态机**：把所有参数编译成 `开始 → loadOp → Subpass 0 → transition → Subpass 1 → storeOp → 结束` 的执行逻辑
3. **和 Pipeline 联动**：后续创建 VkPipeline 时会引用 RenderPass，驱动把 RenderPass 信息烘焙进 Pipeline 编译结果

```
vkCreateRenderPass   ≈ 编译一段程序   （很重）
vkCmdBeginRenderPass ≈ 运行这段程序   （很轻）
```

移动端单次 `vkCreateRenderPass` 可能需要 0.1ms - 1ms。Filament 用 `RenderPassKey` 缓存，相同配置整个生命周期只创建一次。

---

## 四、Filament 的 RenderPass 实现分析

### 3.1 文件位置

| 文件 | 说明 |
|------|------|
| `VulkanDriver.cpp:1922` | `beginRenderPass()` 实现 |
| `VulkanDriver.cpp:2102` | `nextSubpass()` 实现 |
| `VulkanFboCache.cpp` | VkRenderPass 和 VkFramebuffer 的创建与缓存 |
| `VulkanHandles.h:364` | `VulkanRenderTarget` 结构体定义 |
| `VulkanHandles.cpp:401` | `VulkanRenderTarget` 实现 |

> 注意：Filament 没有独立的 VulkanRenderTarget.h/cpp，RenderTarget 定义在 VulkanHandles.h/cpp 中。

### 3.2 beginRenderPass() 流程

```
beginRenderPass(rth, params)
  │
  ├── 处理 SwapChain 的 DONT_CARE 逻辑（首次 pass 强制 discard Color）
  │
  ├── 根据 params 确定 discardStart / discardEnd / clear flags
  │
  ├── 构造 RenderPassKey（包含 clear、discard、subpassMask）
  │
  ├── mFramebufferCache.getRenderPass(rpkey)  ← VkRenderPass 从缓存获取或新建
  │
  ├── mFramebufferCache.getFramebuffer(fbkey) ← VkFramebuffer 从缓存获取或新建
  │
  ├── rt->emitBarriersBeginRenderPass()        ← 发出 attachment layout transition barrier
  │
  ├── 填写 VkRenderPassBeginInfo（包括 ClearValues）
  │
  ├── vkCmdBeginRenderPass()
  │
  ├── vkCmdSetViewport() / vkCmdSetScissor()
  │
  └── 保存 mCurrentRenderPass 状态（包括 currentSubpass = 0）
```

**关键设计：** `RenderPassKey` 将 clear/discard/subpassMask 等所有变化因素编码成缓存键，相同配置复用同一个 VkRenderPass 对象，避免重复创建的开销。

### 3.3 Filament 的 Subpass 支持（有但受限）

**`nextSubpass()` 代码（VulkanDriver.cpp:2102）：**

```cpp
void VulkanDriver::nextSubpass(int) {
    // 注释：Only two subpasses are currently supported.
    FILAMENT_CHECK_PRECONDITION(mCurrentRenderPass.currentSubpass == 0);

    vkCmdNextSubpass(..., VK_SUBPASS_CONTENTS_INLINE);
    mPipelineCache.bindRenderPass(mCurrentRenderPass.renderPass,
            ++mCurrentRenderPass.currentSubpass);

    if (mCurrentRenderPass.params.subpassMask & 0x1) {
        // 把 Color0 绑定为 input attachment，供 Subpass 1 读取
        mDescriptorSetCache.updateInputAttachment({}, renderTarget->getColor0());
    }
}
```

**`VulkanFboCache.cpp` 中的 RenderPass 创建逻辑：**

```cpp
// 最多支持 2 个 subpass
.subpassCount = hasSubpasses ? 2u : 1u;

// subpass dependency 使用 VK_DEPENDENCY_BY_REGION_BIT
VkSubpassDependency dependencies[1] = {{
    .srcSubpass   = 0,
    .dstSubpass   = 1,
    .srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT,
    .dstStageMask = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT,
    .srcAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT,
    .dstAccessMask = VK_ACCESS_SHADER_READ_BIT,
    .dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT,  // TBDR 优化已设置！
}};

// 限制：subpassMask 必须等于 1（只支持第 0 个颜色附件作为 input attachment）
assert_invariant(config.subpassMask == 1);
```

**subpass 感知的 ColorTargetCount：**

```cpp
// VulkanHandles.cpp
uint8_t VulkanRenderTarget::getColorTargetCount(VulkanRenderPassContext const& pass) const {
    if (pass.currentSubpass == 0) {
        // Subpass 0：排除 subpassMask 标记的 attachment（它作为 input attachment 在 Subpass 1 使用）
        // ... 返回不含 input attachment 的颜色数
    }
    // Subpass 1：返回全部颜色数
}
```

---

## 四、VulkanFboCache 缓存机制详解

### 4.1 getRenderPass() 流程

```
getRenderPass(config)
  │
  ├── 缓存命中（UTILS_LIKELY）→ 更新 timestamp，直接返回   ← 热路径
  │
  └── 缓存未命中 → 真正创建：
       ├── 根据 subpassMask 决定 1 个还是 2 个 Subpass
       ├── 根据 clear/discard flags 决定每个附件的 loadOp：
       │     clear=true  → CLEAR
       │     discard=true → DONT_CARE
       │     否则         → LOAD（保留旧内容，多 View 场景）
       ├── 根据 lazilyAllocated 决定 storeOp：
       │     lazilyAllocated → DONT_CARE（On-Chip 内存不写主存）
       │     否则            → STORE
       ├── 设置 VK_DEPENDENCY_BY_REGION_BIT（TBDR 优化）
       └── vkCreateRenderPass() → 存入缓存
```

### 4.2 缓存驱逐策略

```cpp
// VulkanFboCache.cpp
static constexpr uint32_t TIME_BEFORE_EVICTION = FVK_MAX_COMMAND_BUFFERS; // = 45

// VulkanConstants.h
FVK_MAX_COMMAND_BUFFERS = 3 * 15 = 45
```

**45 的来源：**
- `3`：Triple Buffering，飞行中最多 3 帧的 CB 同时存在
- `15`：Filament 估算一帧最多 15 个 RenderPass（Shadow、G-Buffer、Lighting、SSAO、Bloom 各级、TAA 等）
- `3 × 15 = 45`：同时飞行的 CB 上限，RenderPass 必须在所有引用它的 CB 执行完后才能销毁

**注意：** 移动端实际上通常是**双缓冲 + Vsync**，不是三缓冲。Filament 写 3 是跨平台保守估算（桌面端可能三缓冲），且一帧内多次 commit 也会产生多个飞行中的 CB。

**对比：不同缓存的驱逐策略**

| 文件 | TIME_BEFORE_EVICTION | 管理对象 | 原因 |
|------|---------------------|---------|------|
| `VulkanFboCache.cpp` | **45** | VkRenderPass / VkFramebuffer | 创建代价高，保留更久 |
| `VulkanBufferCache.cpp` | **3** | 临时 Buffer | 生命周期短，3帧没用就清理 |
| `VulkanStagePool.cpp` | **3** | Staging Buffer | 同上 |

### 4.3 subpassMask 控制 Input Attachment 的完整逻辑

```cpp
// 同一个 attachmentIndex 同时注册到两个 Subpass
if (config.subpassMask & (1 << i)) {
    // Subpass 0：作为颜色输出
    colorAttachmentRefs[0][index] = { attachmentIndex, subpassLayout };

    // Subpass 1：同时作为 Input Attachment（从 Tile Memory 读）
    inputAttachmentRef[index] = { attachmentIndex, subpassLayout };
}
// Subpass 1：还有自己的颜色输出
colorAttachmentRefs[1][index] = { attachmentIndex, subpassLayout };
```

同一个 `attachmentIndex` 出现在 Subpass 0 的 colorAttachments 和 Subpass 1 的 inputAttachments，这是告诉 Vulkan 驱动：**数据通过 Tile Memory 传递，不走主存。**

---

## 五、为什么 Filament 只用了 2 个 Subpass

### 4.1 使用情况总结

| 功能 | 状态 |
|------|------|
| 多 Subpass 支持 | **有**，但最多 2 个 |
| subpassMask 限制 | 只支持第 0 个颜色附件（`assert_invariant(config.subpassMask == 1)`） |
| `VK_DEPENDENCY_BY_REGION_BIT` | **已设置**，TBDR 优化正确 |
| 实际使用场景 | 主要用于 `PostProcessManager` 的 Resolve/Blit 类操作 |

### 4.2 为什么不用更多 Subpass

**复杂度 vs 收益权衡：**

1. **兼容性成本高**：多 Subpass 配置难以在所有 Vulkan 设备上完美验证，桌面端 GPU 对 subpass 的优化不如 TBDR，不能保证普遍有益。

2. **Filament 的渲染策略**：Filament 的 FrameGraph 系统负责管理 Pass 之间的依赖，更倾向于将 Deferred Lighting 的各个步骤建模为独立的 FrameGraph Pass，而不是塞进 Subpass。这保持了渲染架构的灵活性。

3. **subpassLoad 的采样限制**：现代渲染算法（SSAO、Temporal AA、Screen Space Reflections）需要读取邻近像素，无法用 subpassLoad，只能通过纹理采样实现，必须回到主存。这限制了 Subpass 的适用范围。

4. **当前 2-Subpass 已覆盖主要场景**：Subpass 0 写入中间结果，Subpass 1 读取并写出最终结果，这个 1:1 的模式足以覆盖 Deferred 的核心路径。

### 4.3 与 GPU Driven 的关系

GPU Driven 改造不需要改动 Subpass 结构。RenderPass/Subpass 决定了**怎么渲染**，GPU Driven 改造的是**哪些物体被渲染**（Culling）和**如何提交 DrawCall**（Indirect Draw），两者正交。

---

## 五、VulkanRenderTarget 核心字段

```
VulkanRenderTarget
  ├── mOffscreen      是否是离屏渲染目标（false = SwapChain）
  ├── mProtected      是否使用受保护的内存
  └── mInfo (Auxiliary)
       ├── rpkey (RenderPassKey)   RenderPass 缓存键
       ├── fbkey (FboKey)          Framebuffer 缓存键
       ├── attachments[]           所有 VulkanAttachment
       ├── colors (bitset32)       哪些 slot 有颜色附件
       ├── depthIndex              深度附件在 attachments 中的索引
       ├── msaaDepthIndex          MSAA 深度附件索引
       └── msaaIndex               MSAA 颜色附件索引
```

关键方法：
- `emitBarriersBeginRenderPass()`：在 RenderPass 开始前，对所有附件发出 layout transition barrier
- `emitBarriersEndRenderPass()`：RenderPass 结束后，颜色附件转 `FRAG_READ`，深度附件转 `DEPTH_SAMPLER`
- `getColorTargetCount(pass)`：根据当前 subpass 返回正确的颜色附件数量

---

## 六、关键知识点小结

| 概念 | 要点 |
|------|------|
| `loadOp = DONT_CARE` | 告知驱动不需要从主存加载，TBDR 下 Tile 初始值未定义，节省带宽 |
| `storeOp = DONT_CARE` | 告知驱动不需要写回主存，深度/模板附件通常可以这样设置 |
| `subpassLoad()` | 从 On-Chip Tile Memory 读取同一像素的前一 Subpass 输出，零带宽 |
| `VK_DEPENDENCY_BY_REGION_BIT` | 允许 Tile 级别的并行，不同 Tile 的 Subpass 0→1 独立执行 |
| Filament 的 2-Subpass | 有实现，但限制在第 0 颜色附件，用于特定 resolve 路径 |
| RenderPassKey 缓存 | Filament 用 key 值缓存 VkRenderPass，避免重复创建 |

---

## 七、遗留问题与 TODO

- [ ] 找到 Filament 中实际调用 `nextSubpass()` 的上层代码（在哪个 PostProcess pass 里？）
- [ ] 理解 FrameGraph 如何决定是否启用 subpassMask
- [ ] 与 Day 4 的 loadOp/storeOp 分析对照，确认两者描述一致
