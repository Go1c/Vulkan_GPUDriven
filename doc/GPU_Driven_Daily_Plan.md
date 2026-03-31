# GPU Driven 30 天每日执行计划

> 每天 3-4 小时 | 基于 Filament 引擎 | 目标：Android 高端机 GPU Driven 完整管线
> 配套大纲文档：GPU_Driven_Learning_Outline.md

---

## ═══════════════════════════════════════
## Week 1：Vulkan 基础 + TBDR 理论（Day 1-7）
## ═══════════════════════════════════════

---

### Day 1 — 环境搭建 + Vulkan 初印象

**上午（理论 1.5h）：**
- [ ] 阅读 [vulkan-tutorial.com](https://vulkan-tutorial.com/) Overview 和 Drawing a triangle / Setup 章节
- [ ] 理解 Vulkan 的核心设计哲学：显式控制、无隐式状态
- [ ] 画出 Vulkan 对象关系图：Instance → PhysicalDevice → Device → Queue → CommandBuffer

**下午（动手 2h）：**
- [x] 安装 Android Studio + NDK (r25+)
- [ ] 安装 Vulkan SDK（LunarG）
  - 已安装 `vulkan-loader / vulkan-tools / vulkan-validationlayers`，LunarG SDK 需图形安装器完成最后一步
- [x] Clone Filament 仓库：`git clone https://github.com/google/filament.git`
- [x] 阅读 Filament 的 `README.md` 和 `BUILDING.md`

**产出：** 环境就绪，能编译 Filament（已用 `./build.sh -c -p desktop release matc` 验证通过）

---

### Day 2 — 编译运行 Filament

**上午（动手 2h）：**
- [x] 按照 Filament 文档在 macOS 上编译 Desktop 版本
  ```bash
  cd filament
  ./build.sh -p desktop release
  ```
- [x] 运行 Filament 自带的 sample-gltf-viewer
- [x] 如果编译出错，记录问题并解决
  - 本轮 `./build.sh -p desktop release` 编译成功（`exit_code: 0`），无阻塞错误
  - `gltf_viewer` 已通过 `./out/cmake-release/samples/gltf_viewer -a vulkan` 启动

**下午（动手 2h）：**
- [x] 编译 Android 版本（需要 Android SDK/NDK）
  ```bash
  cd filament
  # 当前 Filament 要求 NDK 29（见 build/common/versions），需 sdkmanager 安装 ndk;29.0.14206865
  # 常见 64 位真机可只编 arm64 以省时间：
  ./build.sh -p android release -q arm64-v8a
  # 全 ABI 则去掉 -q arm64-v8a（耗时明显更长）
  ```
- [ ] 在 Android 真机上运行 Filament 的 sample app
  - 已生成 APK：`filament/android/samples/sample-hello-triangle/build/outputs/apk/release/sample-hello-triangle-release-unsigned.apk`（由 `./build.sh -p android -q arm64-v8a -k sample-hello-triangle release` 产出）
  - 接上真机并开启 USB 调试后：`export PATH="$HOME/Library/Android/sdk/platform-tools:$PATH"` → `adb install -r <上述 apk 路径>`
- [ ] 用 `adb logcat` 确认跑的是 Vulkan 后端（搜索 "Vulkan" 关键字）
  - 示例：`adb logcat | grep -i -E 'Vulkan|Filament|FEngine'`（或先 `adb shell pidof com.google.android.filament.hellotriangle` 再按 PID 过滤）

**产出：** Filament Desktop + Android 双端运行成功（真机安装与 logcat 待设备连接后完成）

---

### Day 3 — Vulkan 核心概念：资源与内存

**上午（理论 2h）：**
- [x] 阅读 vulkan-tutorial.com：Vertex buffers、Uniform buffers 章节
- [x] 理解 Buffer 创建流程：`vkCreateBuffer` → 查询内存需求 → 分配内存 → 绑定
- [ ] 理解内存类型：DEVICE_LOCAL / HOST_VISIBLE / LAZILY_ALLOCATED
- [ ] 阅读 [VMA 文档](https://gpuopen-librariesandsdks.github.io/VulkanMemoryAllocator/html/) 快速入门章节

**下午（代码阅读 2h）：**
- [x] 打开 Filament 源码 `backend/src/vulkan/`
- [x] 阅读 `VulkanMemory.h/.cpp` — 看 Filament 如何集成 VMA
- [x] 搜索 `LAZILY_ALLOCATED` 关键字 — 看在哪里使用了移动端专用内存
- [x] 做笔记：Filament 的内存管理策略

**产出：** 理解 Vulkan 内存模型，读懂 Filament 内存管理代码

---

### Day 4 — TBDR 架构深度理解

**上午（理论 2.5h）：**
- [ ] 阅读 [ARM Mali Best Practices](https://developer.arm.com/documentation/101897/latest) 前 5 章
  - 重点：Tile-Based Rendering 章节
  - 重点：Bandwidth 章节
- [ ] 理解 TBDR 两阶段流水线（Binning + Rendering）
- [ ] 画出 TBDR vs IMR 对比示意图

**下午（理论 + 代码 1.5h）：**
- [ ] 阅读 Khronos 官方 [Vulkan Mobile Best Practices](https://github.com/KhronosGroup/Vulkan-Samples/blob/main/samples/performance/render_passes/README.md)
- [ ] 在 Filament 源码中搜索 `loadOp` 和 `storeOp`
- [ ] 记录 Filament 对每种 Attachment 的 load/store 策略

**产出：** 深入理解 TBDR 原理，知道为什么移动端需要特殊优化

**延伸阅读：** [TBDR_Notes.md](./TBDR_Notes.md)（Binning、全流程、并行与执行效率讨论纪要）

---

### Day 5 — RenderPass 与 Subpass

**上午（理论 1.5h）：**
- [ ] 阅读 vulkan-tutorial.com：Render passes 章节
- [ ] 理解 VkRenderPass 结构：Attachment Description + Subpass Description + Dependency
- [ ] 理解 Subpass 如何在 TBDR 上避免主存读写

**下午（代码阅读 2.5h）：**
- [ ] 阅读 Filament `VulkanDriver.cpp` 中 `beginRenderPass()` 实现
- [ ] 追踪一个完整的 RenderPass 创建流程
- [ ] 看 Filament 是否使用了多 Subpass
- [ ] 如果没有用多 Subpass，思考为什么（兼容性？复杂度？）
- [ ] 阅读 `VulkanRenderTarget.h/.cpp`

**产出：** 理解 Filament 的 RenderPass 设计，评估 Subpass 使用现状

---

### Day 6 — Pipeline 与 Shader

**上午（理论 1.5h）：**
- [ ] 阅读 vulkan-tutorial.com：Graphics pipeline basics 章节
- [ ] 理解 Pipeline 的组成：Shader Stage → Vertex Input → Rasterizer → Depth/Stencil → Color Blending
- [ ] 理解 Pipeline Layout、Descriptor Set Layout

**下午（代码阅读 2.5h）：**
- [ ] 阅读 Filament `VulkanPipelineCache.h/.cpp`
- [ ] 理解 Filament 如何缓存和复用 Pipeline
- [ ] 阅读 Filament 的 Shader 编译流程：`.mat` → `matc` → SPIR-V
- [ ] 看 Descriptor Set 的布局方式（后续 Bindless 改造需要理解这里）

**产出：** 理解 Filament 的 Pipeline 和 Shader 管理

---

### Day 7 — Week 1 总结与 Command Buffer 录制

**上午（理论 1.5h）：**
- [ ] 阅读 vulkan-tutorial.com：Command buffers、Drawing 章节
- [ ] 理解 Primary / Secondary Command Buffer 区别
- [ ] 理解同步原语：Fence、Semaphore、Pipeline Barrier

**下午（总结 2.5h）：**
- [ ] 回顾 Week 1 所有笔记
- [ ] 在 Filament 源码中追踪一次完整的渲染帧流程：
  ```
  Renderer::beginFrame()
    → 场景准备
    → RenderPass 开始
    → DrawCall 录制
    → RenderPass 结束
  Renderer::endFrame()
    → 提交 CommandBuffer
    → Present
  ```
- [ ] 写一份 Week 1 总结文档（自己的理解，不抄文档）

**产出：** 对 Vulkan + TBDR + Filament 有完整的基础认知

---

## ═══════════════════════════════════════
## Week 2：Filament 源码深度阅读（Day 8-14）
## ═══════════════════════════════════════

---

### Day 8 — Filament 渲染主循环

**全天（4h）：**
- [ ] 打开 `filament/src/Renderer.cpp`
- [ ] 从 `Renderer::render()` 开始，逐行跟踪
- [ ] 画出完整的调用链流程图
- [ ] 理解 `FrameGraph`（帧图）的作用 — Filament 用它管理渲染通道依赖
- [ ] 找到 FrameGraph 相关文件并阅读
- [ ] 记录：哪些地方是 CPU 瓶颈（DrawCall 排序、Culling 等）

**产出：** 完整的 Filament 渲染主循环流程图

---

### Day 9 — Filament 场景管理与 Culling

**全天（4h）：**
- [ ] 阅读 `filament/src/Scene.cpp`
- [ ] 找到 Frustum Culling 的实现位置
- [ ] 理解 Filament 的 Culling 算法：用的是包围球还是 AABB？
- [ ] 找到 `RenderableManager` — 理解可渲染实体的数据结构
- [ ] 分析 Culling 的 CPU 开销：遍历所有物体，逐个测试
- [ ] **标记改造点：这里是 GPU Driven 第一个要替换的地方**

**产出：** 明确知道 Filament CPU Culling 在哪里、怎么工作、怎么替换

---

### Day 10 — Filament DrawCall 排序与提交

**全天（4h）：**
- [ ] 阅读 `filament/src/RenderPass.cpp`
- [ ] 理解 SortKey 的设计：64-bit key 是怎么编码的
- [ ] 理解排序策略对 GPU 性能的影响（减少状态切换）
- [ ] 追踪 DrawCall 从排序到最终 `vkCmdDrawIndexed()` 的完整路径
- [ ] **标记改造点：这里要改为 `vkCmdDrawIndexedIndirect()`**

**产出：** 理解 DrawCall 提交路径，标记 GPU Driven 改造点

---

### Day 11 — Filament Vulkan 后端深度阅读

**全天（4h）：**
- [ ] 阅读 `VulkanDriver.cpp` 核心函数：
  - `draw()` — 单次绘制调用
  - `beginRenderPass()` / `endRenderPass()`
  - `createBufferObject()` / `createTexture()`
- [ ] 理解 Filament 的 Backend 抽象层（`DriverAPI`）
- [ ] 看 `DriverAPI.inc` — 所有后端接口的声明
- [ ] 思考：在哪里可以加入 `drawIndirect()` 接口

**产出：** 理解 Filament 后端抽象层，规划新接口添加位置

---

### Day 12 — Filament 材质系统

**全天（4h）：**
- [ ] 阅读 Filament 的材质文档
- [ ] 理解 `.mat` 文件格式
- [ ] 跟踪材质编译流程：`.mat` → `matc` → `.filamat` → SPIR-V
- [ ] 理解 Shader 变体系统（skinning、instancing 等）
- [ ] 看 Descriptor Set 绑定频率：per-view / per-renderable / per-material
- [ ] **标记改造点：per-renderable 的绑定需要改为 Bindless**

**产出：** 理解材质系统，标记 Bindless 改造点

---

### Day 13 — Compute Shader 在 Filament 中的使用

**全天（4h）：**
- [ ] 搜索 Filament 源码中所有 Compute Shader 的使用
- [ ] Filament 是否有 Compute Pipeline？如果有，在哪里？
- [ ] 看 Vulkan 后端是否支持 Compute Dispatch
- [ ] 如果 Filament 没有完整的 Compute 支持：
  - 找到 `VulkanDriver` 中可以扩展 Compute 的位置
  - 理解 Queue 选择策略（Graphics vs Compute）

**产出：** 评估 Filament Compute 支持现状，规划 Compute Pipeline 添加方案

---

### Day 14 — Week 2 总结与 GPU Driven 改造方案设计

**全天（4h）：**
- [ ] 回顾 Week 2 所有笔记和标记的改造点
- [ ] 设计 GPU Driven 改造方案，写成文档：
  ```
  改造点 1：CPU Culling → GPU Culling
    位置：Scene.cpp prepare()
    方案：新增 Compute Pipeline，替换 Culling 逻辑
    依赖：Compute Pipeline 支持、SSBO 支持

  改造点 2：vkCmdDrawIndexed → vkCmdDrawIndexedIndirect
    位置：VulkanDriver.cpp draw()
    方案：新增 drawIndirect() 接口
    依赖：Indirect Buffer 创建

  改造点 3：Per-renderable Descriptor → Bindless
    位置：材质绑定逻辑
    方案：Descriptor Indexing
    依赖：设备特性检查

  改造点 4：CPU Scene Data → GPU Scene Buffer
    位置：RenderableManager
    方案：大 SSBO 存储所有 Transform + Material 索引
    依赖：SSBO 支持
  ```
- [ ] 评估每个改造点的难度和优先级
- [ ] 确定 Week 3-4 的实施顺序

**产出：** 完整的 GPU Driven 改造方案文档

---

## ═══════════════════════════════════════
## Week 3：GPU Driven 核心实现（Day 15-21）
## ═══════════════════════════════════════

---

### Day 15 — GPU Scene Buffer 实现

**上午（设计 1.5h）：**
- [ ] 设计 GPU 端场景数据结构：
  ```glsl
  struct ObjectData {
      mat4 transform;
      vec4 boundingSphere;   // xyz=center, w=radius
      uint meshIndex;        // 指向全局 Mesh 数据
      uint materialIndex;    // 指向材质索引
      uint padding[2];
  };
  ```
- [ ] 确定 Buffer 大小策略（固定大小 or 动态扩展）

**下午（实现 2.5h）：**
- [ ] 在 Filament 中创建 GPU Scene SSBO
- [ ] 实现 CPU → GPU 的场景数据上传
- [ ] 处理动态更新（只上传变化的物体）
- [ ] 验证 Buffer 创建正确（通过 RenderDoc 检查）

**产出：** GPU Scene Buffer 创建和上传功能

---

### Day 16 — Compute Pipeline 基础搭建

**上午（实现 2h）：**
- [ ] 在 Filament Vulkan 后端添加 Compute Pipeline 创建功能
- [ ] 创建 Compute Shader 编译流程
- [ ] 实现 `vkCmdDispatch` 封装

**下午（验证 2h）：**
- [ ] 写一个最简单的 Compute Shader（例如：数组乘 2）
- [ ] 在 Filament 中调度执行
- [ ] 用 RenderDoc 验证 Compute Shader 执行结果

**产出：** Filament 中 Compute Pipeline 可用

---

### Day 17 — GPU Frustum Culling Shader 编写

**全天（4h）：**
- [ ] 编写 `culling.comp`：
  ```
  输入：ObjectData[] + Camera Frustum Planes[6]
  输出：VkDrawIndexedIndirectCommand[] + DrawCount
  ```
- [ ] 实现包围球 vs 视锥 6 平面测试
- [ ] 处理 atomicAdd 竞争（Indirect Buffer 写入）
- [ ] 编译 Shader 为 SPIR-V
- [ ] 单独测试（用简单场景，手动验证 Culling 结果是否正确）

**产出：** GPU Culling Compute Shader 编写完成并初步验证

---

### Day 18 — Indirect Draw 路径实现

**全天（4h）：**
- [ ] 在 Filament Backend 添加 `drawIndirect()` 接口
- [ ] 在 `VulkanDriver` 中实现 `vkCmdDrawIndexedIndirect`
- [ ] 创建 Indirect Buffer（存储 DrawCommand）
- [ ] 创建 Count Buffer（存储可见物体数量）
- [ ] 处理 Pipeline Barrier：
  ```
  Compute Write (Culling)
    → Barrier (COMPUTE → INDIRECT_DRAW)
  Indirect Draw
  ```
- [ ] 在简单场景上测试：GPU Culling → Indirect Draw

**产出：** GPU Culling + Indirect Draw 基础管线跑通

---

### Day 19 — 整合进 Filament 渲染循环

**全天（4h）：**
- [ ] 修改 `Scene::prepare()`：
  - 原有 CPU Culling 保留作为 fallback
  - 新增 GPU Culling 路径（通过配置开关）
- [ ] 修改渲染提交路径：
  - 支持在 GPU Culling 和传统 DrawCall 之间切换
- [ ] 处理帧同步：
  - Compute 在 Graphics 之前执行
  - Fence / Semaphore 正确设置
- [ ] 在 Filament sample 上运行测试

**产出：** GPU Driven 路径整合进 Filament 主循环

---

### Day 20 — Mesh 合并与全局 Vertex/Index Buffer

**上午（设计 1.5h）：**
- [ ] 设计全局 Mesh Buffer 方案：
  ```
  所有 Mesh 的顶点数据 → 一个大 Vertex Buffer
  所有 Mesh 的索引数据 → 一个大 Index Buffer
  每个 Mesh 记录 (firstIndex, indexCount, vertexOffset)
  ```
- [ ] 这样 Indirect Draw 只需要绑定一次 Vertex/Index Buffer

**下午（实现 2.5h）：**
- [ ] 实现 Mesh 数据合并加载
- [ ] 修改 Indirect Command 的 `firstIndex` / `vertexOffset` 字段
- [ ] 测试多 Mesh 场景的 Indirect Draw

**产出：** 全局 Mesh Buffer，一次绑定画所有 Mesh

---

### Day 21 — Week 3 总结与 Bug 修复

**全天（4h）：**
- [ ] 用较复杂的场景测试（1000+ 物体）
- [ ] 对比 CPU Culling 和 GPU Culling 的结果是否一致
- [ ] 修复发现的 Bug
- [ ] 性能初步对比：
  ```
  指标            CPU Culling    GPU Culling
  CPU 帧时间      ?ms            ?ms
  DrawCall 数     ?              ?
  ```
- [ ] 写 Week 3 总结

**产出：** GPU Culling + Indirect Draw 基本稳定可用

---

## ═══════════════════════════════════════
## Week 4：Bindless + Hi-Z + 优化（Day 22-30）
## ═══════════════════════════════════════

---

### Day 22 — Bindless Descriptor 基础

**全天（4h）：**
- [ ] 理解 `VK_EXT_descriptor_indexing` 扩展
- [ ] 在 Filament 设备初始化中启用相关特性：
  ```
  descriptorBindingPartiallyBound
  runtimeDescriptorArray
  shaderSampledImageArrayNonUniformIndexing
  ```
- [ ] 创建 Bindless Descriptor Set Layout（大纹理数组）
- [ ] 验证目标设备是否支持这些特性

**产出：** Bindless 环境准备完成

---

### Day 23 — Bindless 纹理实现

**全天（4h）：**
- [ ] 实现纹理注册系统：
  ```
  加载纹理 → 分配索引 → 写入 Descriptor Array → 返回索引
  ```
- [ ] 修改 Shader：通过 `materialIndex` 索引纹理数组
  ```glsl
  layout(set = 1, binding = 0) uniform sampler2D textures[];
  vec4 albedo = texture(textures[nonuniformEXT(matIdx)], uv);
  ```
- [ ] 测试多材质场景的 Bindless 渲染

**产出：** Bindless 纹理渲染工作

---

### Day 24 — Bindless 整合 + GPU Driven 完整路径

**全天（4h）：**
- [ ] 将 Bindless 与 GPU Culling + Indirect Draw 整合
- [ ] 完整的数据流：
  ```
  CPU: 上传场景数据 → GPU Scene Buffer
  GPU Compute: Frustum Culling → Indirect Draw Buffer
  GPU Graphics: Indirect Draw → Bindless 纹理采样
  ```
- [ ] 测试完整管线
- [ ] 修复整合问题

**产出：** GPU Driven 完整管线（Culling + Indirect + Bindless）

---

### Day 25 — Hi-Z Buffer 生成

**全天（4h）：**
- [ ] 编写 Hi-Z Mipmap 生成 Compute Shader：
  ```glsl
  // 每级取 2x2 像素的最大深度（最远距离）
  float d0 = texelFetch(prevLevel, ivec2(x*2, y*2), 0).r;
  float d1 = texelFetch(prevLevel, ivec2(x*2+1, y*2), 0).r;
  float d2 = texelFetch(prevLevel, ivec2(x*2, y*2+1), 0).r;
  float d3 = texelFetch(prevLevel, ivec2(x*2+1, y*2+1), 0).r;
  float maxDepth = max(max(d0, d1), max(d2, d3));
  ```
- [ ] 从上一帧的 Depth Buffer 生成 Hi-Z Mipmap 链
- [ ] 验证 Hi-Z 结果（用 RenderDoc 查看每级 Mipmap）

**产出：** Hi-Z Mipmap 生成管线

---

### Day 26 — Hi-Z Occlusion Culling

**全天（4h）：**
- [ ] 修改 `culling.comp` 加入 Occlusion 测试：
  ```
  1. Frustum Culling（已有）
  2. 通过 Frustum 的物体 → 投影包围盒到屏幕空间
  3. 根据包围盒屏幕大小选择 Hi-Z Mip 级别
  4. 采样 Hi-Z → 比较深度 → 判断是否被遮挡
  ```
- [ ] 处理 Two-Phase Culling 逻辑
- [ ] 测试遮挡场景

**产出：** Hi-Z Occlusion Culling 实现

---

### Day 27 — 移动端性能优化

**全天（4h）：**
- [ ] Compute Workgroup 大小优化：
  - 测试 64 / 128 / 256 不同 local_size
  - 选择目标 GPU 最优配置
- [ ] Buffer 优化：
  - Indirect Buffer 使用 DEVICE_LOCAL 内存
  - 减少不必要的 Pipeline Barrier
- [ ] 带宽优化：
  - 使用 16-bit Index Buffer
  - 压缩顶点格式（half float）
- [ ] SubPass 优化：
  - 确保 G-Buffer 使用 Subpass Input（On-Chip Memory 传递）
  - Depth Buffer 使用 LAZILY_ALLOCATED

**产出：** 针对移动端的性能调优

---

### Day 28 — 性能测量与对比

**全天（4h）：**
- [ ] 准备测试场景：
  - 场景 A：100 个物体（简单场景）
  - 场景 B：1000 个物体（中等场景）
  - 场景 C：10000 个物体（压力测试）
- [ ] 每个场景分别测试：
  ```
  模式 1：原始 Filament（CPU Culling + 传统 DrawCall）
  模式 2：GPU Frustum Culling + Indirect Draw
  模式 3：GPU Frustum Culling + Hi-Z Occlusion + Indirect Draw + Bindless
  ```
- [ ] 记录指标：
  ```
  □ CPU 帧时间 (ms)
  □ GPU 帧时间 (ms)
  □ DrawCall 数量
  □ 被 Culling 物体比例
  □ 总带宽消耗
  ```
- [ ] 使用 Android GPU Inspector 或 Snapdragon Profiler 抓取数据

**产出：** 完整的性能对比报告

---

### Day 29 — Bug 修复 + 边界情况处理

**全天（4h）：**
- [ ] 处理已知问题：
  - 相机快速旋转时 Culling 延迟一帧的伪影
  - 物体动态增删时 GPU Buffer 同步
  - 极端情况：所有物体被 Cull / 所有物体可见
- [ ] 设备兼容性测试：
  - 至少在 2 款不同 GPU 的设备上测试
  - 处理不支持 Descriptor Indexing 的 fallback
- [ ] 代码清理和注释

**产出：** 稳定可用的 GPU Driven 管线

---

### Day 30 — 项目总结

**全天（4h）：**
- [ ] 撰写项目总结文档：
  ```
  1. 架构设计：整体管线图
  2. 实现细节：每个模块的关键决策
  3. 性能数据：对比图表
  4. 踩坑记录：遇到的问题和解决方案
  5. 未来方向：
     - Mesh Shader（等移动端支持）
     - Virtual Geometry（类 Nanite）
     - Cluster Rendering
  ```
- [ ] 整理代码，可以考虑提交到 GitHub
- [ ] 回顾 30 天学到的所有知识，查漏补缺

**产出：** 完整的项目文档 + 可运行的 GPU Driven 渲染管线

---

**TBDR 讨论纪要（Binning、全流程、并行、执行效率）** 已单独成文，见：[TBDR_Notes.md](./TBDR_Notes.md)。

---

## 每日检查清单

每天结束前确认：

```
□ 今天的目标是否完成？
□ 遇到了什么问题？记录下来
□ 明天需要什么准备？
□ 有没有需要回头复习的概念？
```

---

## 风险与应对

| 风险 | 应对策略 |
|------|---------|
| Filament 编译失败 | 参考 GitHub Issues，切换到已知稳定的 release tag |
| 目标设备不支持某特性 | 准备 fallback 路径，优先保证 Frustum Culling + Indirect Draw |
| Compute Shader 调试困难 | 先在 Desktop Vulkan 上调通，再移植到 Android |
| 进度落后 | Week 4 的 Hi-Z 可以延后，优先保证 Week 3 的核心管线 |
| 性能不如预期 | 移动端收益主要在 CPU 端，GPU 端可能持平，这是正常的 |

---

## 最低完成标准（如果时间不够）

```
必须完成 ★：
  ★ GPU Frustum Culling (Compute Shader)
  ★ Indirect Draw
  ★ GPU Scene Buffer

尽量完成 ☆：
  ☆ Bindless Resources
  ☆ Hi-Z Occlusion Culling

可以延后 ○：
  ○ Two-Phase Occlusion Culling
  ○ 完整性能优化
```
