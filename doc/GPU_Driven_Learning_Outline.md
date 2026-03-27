# GPU Driven Rendering 学习大纲

> 基于 Google Filament 引擎 | 目标平台：Android 高端机 | 周期：30 天
> 前置要求：C++ 基础、线性代数基础、了解图形学基本概念

---

## 全局路线图

```
Week 1                Week 2                Week 3                Week 4
Vulkan 基础           Filament 源码         GPU Driven 核心       完整管线 + 优化
+ TBDR 理论           深度阅读              实现                   调试验证
```

---

## 模块一：Vulkan 基础与移动端 GPU 架构（Week 1）

### 1.1 Vulkan 核心概念

**学什么：** Vulkan 区别于 OpenGL 最大的点——显式控制。你需要手动管理一切。

**核心知识点：**

- **Instance & Physical Device**
  - Instance 是 Vulkan 的入口，Physical Device 是你选择的物理 GPU
  - 在移动端通常只有一个 GPU

- **Logical Device & Queue**
  - Logical Device 是对 GPU 的抽象句柄
  - Queue 是命令提交的通道，分为 Graphics Queue / Compute Queue / Transfer Queue
  - 移动端 GPU 通常 Graphics 和 Compute 共享同一个 Queue Family

- **Command Buffer & Command Pool**
  - 所有渲染指令先录制到 Command Buffer，再一次性提交给 GPU
  - Command Pool 负责分配和管理 Command Buffer 的内存

- **Synchronization（同步机制）**
  - Fence：CPU 等 GPU 完成
  - Semaphore：Queue 之间同步
  - Pipeline Barrier：同一 Queue 内的资源依赖

- **Swapchain**
  - 管理呈现到屏幕的图像链
  - 移动端通常用 MAILBOX 或 FIFO 模式

**推荐资源：**
- [Vulkan Tutorial](https://vulkan-tutorial.com/) — 前 10 章
- [vkguide.dev](https://vkguide.dev/) — 更现代的写法

---

### 1.2 移动端 GPU 架构 —— TBDR（Tile-Based Deferred Rendering）

**学什么：** 移动 GPU 和 PC GPU 的本质架构差异，这是你做所有优化的基础。

**核心知识点：**

- **IMR vs TBDR**
  - PC GPU（NVIDIA/AMD）用 Immediate Mode Rendering（IMR）：三角形提交后立即光栅化
  - 移动 GPU（Mali/Adreno/Apple）用 TBDR：先把整个帧分成 Tile，再逐 Tile 渲染

- **TBDR 两阶段流水线**
  ```
  阶段 1：Binning Pass（顶点处理）
    → 所有顶点变换完成
    → 每个三角形被分配到它覆盖的 Tile

  阶段 2：Rendering Pass（逐 Tile 光栅化）
    → 每个 Tile 在 On-Chip Memory 中完成所有片元操作
    → 最终结果写回主存
  ```

- **On-Chip Memory**
  - 每个 Tile 有专用的片上存储（几十 KB）
  - 读写速度远快于主存（DDR），带宽成本接近 0
  - 这就是 TBDR 省电的核心原因

- **带宽是移动端第一杀手**
  - 主存带宽 = 最大的功耗来源
  - 所有优化的核心目标：减少主存读写

- **Vulkan 上的 TBDR 对应**
  - Subpass = 在同一个 Tile 上完成多道渲染（不需要写回主存）
  - `loadOp = CLEAR`（不读主存）比 `loadOp = LOAD` 好
  - `storeOp = DONT_CARE`（不写主存）用于深度图等临时 Attachment
  - `LAZILY_ALLOCATED` 内存 = 告诉驱动：这个 Attachment 不需要真实主存

**推荐资源：**
- [ARM Mali Best Practices](https://developer.arm.com/documentation/101897/latest)
- [Adreno Vulkan Developer Guide](https://developer.qualcomm.com/docs/adreno-gpu/developer-guide/)

---

### 1.3 Vulkan 内存管理与 VMA

**学什么：** GPU 内存类型、分配策略，以及 VMA 如何简化这些。

**核心知识点：**

- **GPU 内存类型**
  ```
  DEVICE_LOCAL          — GPU 专用显存，最快，CPU 不可见
  HOST_VISIBLE          — CPU 可读写，用于 Upload
  HOST_COHERENT         — CPU 写入自动对 GPU 可见，无需 flush
  LAZILY_ALLOCATED      — 移动端专用，不分配真实内存（给 Tile 用）
  ```

- **VMA 核心功能**
  - Sub-allocation：一次 vkAllocateMemory 分一大块，内部切分
  - Memory Pool：针对特定用途的内存池
  - Defragmentation：内存碎片整理

- **移动端最佳实践**
  - Depth/Stencil Buffer 用 LAZILY_ALLOCATED + TRANSIENT_ATTACHMENT
  - Staging Buffer 用 HOST_VISIBLE + HOST_COHERENT
  - Vertex/Index/Uniform Buffer 用 DEVICE_LOCAL

---

### 1.4 RenderPass 与 Subpass

**学什么：** 这是移动端优化最关键的 Vulkan 概念。

**核心知识点：**

- **RenderPass**
  - 定义一次渲染过程用到的所有 Attachment（颜色、深度等）
  - 指定每个 Attachment 的 load/store 操作

- **Subpass**
  - 一个 RenderPass 内可以有多个 Subpass
  - Subpass 之间的 Input Attachment 可以直接在 Tile On-Chip Memory 传递
  - 不经过主存 = 零带宽开销

- **移动端延迟渲染的 Subpass 方案**
  ```
  Subpass 0：G-Buffer 写入（Position、Normal、Albedo）
    ↓ （On-Chip Memory 传递，不写主存）
  Subpass 1：Lighting Pass，读取 G-Buffer 做光照
    ↓
  Subpass 2：后处理（Tone Mapping 等）
  ```

---

## 模块二：Filament 源码深度阅读（Week 2）

### 2.1 Filament 架构总览

**学什么：** Filament 代码组织、模块划分、渲染管线流程。

**核心模块：**
```
filament/
├── backend/            ← Vulkan/Metal/OpenGL 后端抽象
│   └── vulkan/         ← Vulkan 具体实现 ★ 重点
├── src/
│   ├── RenderPass.cpp  ← 渲染通道管理
│   ├── Renderer.cpp    ← 渲染主循环
│   ├── Scene.cpp       ← 场景管理
│   └── View.cpp        ← 视图管理（Camera + Scene）
├── include/
│   └── filament/       ← 公开 API
└── libs/
    ├── math/           ← 数学库
    └── utils/          ← 工具类
```

**阅读顺序：**
1. `Renderer.cpp` → 渲染主循环入口
2. `RenderPass.cpp` → DrawCall 排序、提交
3. `backend/vulkan/` → Vulkan 资源创建、命令录制

---

### 2.2 Filament Vulkan 后端

**学什么：** Filament 如何封装 Vulkan，如何处理 TBDR 优化。

**重点文件：**
```
backend/src/vulkan/
├── VulkanContext.h/cpp        ← Vulkan 初始化、设备选择
├── VulkanDriver.h/cpp         ← 核心驱动，资源创建+命令录制
├── VulkanHandles.h/cpp        ← Buffer、Texture、RenderTarget 封装
├── VulkanPipelineCache.h/cpp  ← Pipeline 缓存管理
├── VulkanSwapChain.h/cpp      ← Swapchain 管理
└── VulkanMemory.h/cpp         ← 内存管理（VMA 集成）
```

**关注点：**
- `VulkanDriver::beginRenderPass()` — 看它如何配置 loadOp/storeOp
- `VulkanDriver::createTexture()` — 看它如何选择内存类型
- Subpass 是否被使用、如何使用

---

### 2.3 Filament 的 DrawCall 管理

**学什么：** CPU 端如何组织和提交 DrawCall，这是你后续改造为 GPU Driven 的起点。

**核心概念：**
- **SortKey** — Filament 用 64-bit Key 对 DrawCall 排序，减少状态切换
- **RenderableManager** — 管理所有可渲染实体
- **Scene::prepare()** — CPU 端 Culling（你要改成 GPU 端）

**关键问题（带着问题读代码）：**
```
□ DrawCall 是怎么从 Scene 收集的？
□ 排序策略是什么？（材质优先 or 深度优先）
□ Frustum Culling 在哪里做的？用什么数据结构？
□ 实例化（Instancing）是否支持？怎么做的？
```

---

### 2.4 Filament 材质系统

**学什么：** 材质编译、Shader 变体、Descriptor 管理。

**核心概念：**
- **matc** — Filament 的材质编译器，.mat → .filamat
- **Shader 变体** — 根据功能开关生成不同 Shader 组合
- **Descriptor Set** — 当前 Filament 如何绑定资源

**为什么重要：** GPU Driven 需要改造材质绑定方式，从"每个 DrawCall 切换 Descriptor Set"变为"Bindless + 索引查找"。

---

## 模块三：GPU Driven 核心实现（Week 3）

### 3.1 GPU Frustum Culling

**学什么：** 把 Filament 的 CPU Culling 迁移到 GPU Compute Shader。

**原理：**
```
输入：所有物体的 AABB（轴对齐包围盒）+ Camera 的 6 个裁剪平面
处理：Compute Shader 每个线程检测一个物体是否在视锥内
输出：可见物体的 Indirect Draw 参数
```

**Compute Shader 伪代码：**
```glsl
layout(local_size_x = 64) in;

struct ObjectData {
    vec4 boundingSphere;  // xyz = center, w = radius
    uint meshIndex;
};

layout(set = 0, binding = 0) readonly buffer Objects { ObjectData objects[]; };
layout(set = 0, binding = 1) writeonly buffer DrawCommands { VkDrawIndexedIndirectCommand cmds[]; };
layout(set = 0, binding = 2) buffer DrawCount { uint count; };
layout(set = 0, binding = 3) uniform Frustum { vec4 planes[6]; };

void main() {
    uint idx = gl_GlobalInvocationID.x;
    if (idx >= objectCount) return;

    ObjectData obj = objects[idx];

    // 测试包围球 vs 6 个裁剪平面
    bool visible = true;
    for (int i = 0; i < 6; i++) {
        if (dot(planes[i], vec4(obj.boundingSphere.xyz, 1.0)) < -obj.boundingSphere.w) {
            visible = false;
            break;
        }
    }

    if (visible) {
        uint drawIdx = atomicAdd(count, 1);
        cmds[drawIdx] = buildDrawCommand(obj.meshIndex);
    }
}
```

**在 Filament 中的改造点：**
- `Scene::prepare()` 中的 CPU Frustum Culling → 替换为 Compute Dispatch
- 新增 Compute Pipeline 管理
- 需要 `vkCmdDrawIndexedIndirect()` 替代逐个 `vkCmdDrawIndexed()`

---

### 3.2 Indirect Draw

**学什么：** 让 GPU 决定画什么、画多少。

**核心概念：**
```
传统 Draw：         CPU 告诉 GPU "画这个 Mesh，从第 0 个顶点开始，画 36 个"
Indirect Draw：    GPU 自己从 Buffer 读取 "画什么、画多少"
```

**VkDrawIndexedIndirectCommand 结构：**
```cpp
struct VkDrawIndexedIndirectCommand {
    uint32_t indexCount;      // 索引数量
    uint32_t instanceCount;   // 实例数量（0 = 不画）
    uint32_t firstIndex;      // 起始索引
    int32_t  vertexOffset;    // 顶点偏移
    uint32_t firstInstance;   // 起始实例
};
```

**关键 API：**
```cpp
// 单次 Indirect Draw
vkCmdDrawIndexedIndirect(cmdBuffer, indirectBuffer, offset, drawCount, stride);

// Multi Draw Indirect（一次调用画多个物体）—— 更高效
vkCmdDrawIndexedIndirectCount(cmdBuffer, indirectBuffer, offset,
                              countBuffer, countOffset, maxDrawCount, stride);
```

**移动端注意事项：**
- `vkCmdDrawIndexedIndirectCount` 需要 `VK_KHR_draw_indirect_count`
- Adreno 640+ / Mali G77+ 支持
- 检查 `drawIndirectFirstInstance` 特性是否支持

---

### 3.3 GPU Scene Buffer

**学什么：** 把场景数据从 CPU 搬到 GPU，减少每帧上传。

**数据布局：**
```
GPU Buffer (SSBO):
┌───────────────────────────────────────┐
│ Object 0: Transform(mat4) + MaterialID │
│ Object 1: Transform(mat4) + MaterialID │
│ Object 2: Transform(mat4) + MaterialID │
│ ...                                     │
└───────────────────────────────────────┘

CPU 每帧只更新：
  - 移动了的物体的 Transform
  - 新增/删除的物体
```

**Shader 中访问：**
```glsl
layout(set = 0, binding = 0) readonly buffer SceneData {
    ObjectData objects[];
};

void main() {
    uint objectID = gl_InstanceIndex;  // 通过 Indirect Draw 传入
    mat4 model = objects[objectID].transform;
    // ...
}
```

---

### 3.4 Bindless Resources

**学什么：** 消除"每个 DrawCall 切换纹理绑定"的开销。

**原理：**
```
传统方式：
  DrawCall 1 → 绑定纹理 A → 画
  DrawCall 2 → 绑定纹理 B → 画
  DrawCall 3 → 绑定纹理 C → 画

Bindless 方式：
  一次绑定所有纹理（作为数组）
  DrawCall 1/2/3 → 通过索引访问不同纹理
```

**Vulkan 实现：**
```glsl
#extension GL_EXT_nonuniform_qualifier : require

layout(set = 0, binding = 0) uniform sampler2D textures[];  // Bindless 纹理数组

void main() {
    uint texIdx = objects[objectID].materialID;
    vec4 color = texture(textures[nonuniformEXT(texIdx)], uv);
}
```

**前提条件：**
- 需要 `VK_EXT_descriptor_indexing`
- 需要 `descriptorBindingPartiallyBound` 特性
- 需要 `runtimeDescriptorArray` 特性
- 高端安卓机（Snapdragon 855+ / Dimensity 9000+）支持

---

## 模块四：完整管线整合与优化（Week 4）

### 4.1 Hi-Z Occlusion Culling

**学什么：** 在 GPU Frustum Culling 基础上，加入遮挡剔除。

**原理：**
```
第 1 帧：
  1. 用上一帧的深度图生成 Hi-Z Mipmap（逐级取最大深度）
  2. Compute Shader 中：
     - 把物体包围盒投影到屏幕空间
     - 查找对应 Hi-Z Mip 级别
     - 如果物体最近深度 > Hi-Z 深度 → 被遮挡 → 剔除

第 2 帧：
  用第 1 帧的深度图重复上述过程
```

**Two-Phase Culling（解决第一帧没有深度图的问题）：**
```
Phase 1：用上一帧的 Hi-Z 做保守剔除，画确定可见的物体
Phase 2：用 Phase 1 生成的深度图重新测试 Phase 1 中被剔除的物体
```

---

### 4.2 移动端特有优化

**带宽优化：**
- Indirect Buffer 尽量紧凑，减少 GPU 读取量
- 合并 Vertex Buffer（所有 Mesh 放一个大 Buffer）
- 使用 16-bit Index Buffer

**Compute 调度优化：**
- Workgroup size 对齐到 Warp/Wave 大小（Mali=16, Adreno=64）
- 避免 Compute 和 Graphics 之间频繁 Pipeline Barrier

**功耗优化：**
- GPU Culling 减少的 DrawCall → 直接减少 Binning Pass 工作量
- 被剔除的物体不进入 Tile 分配 → 省带宽

---

### 4.3 性能验证与调试

**工具：**
- **RenderDoc**（Android 支持）— 抓帧分析
- **Snapdragon Profiler** — Adreno GPU 专用
- **ARM Streamline** — Mali GPU 专用
- **Android GPU Inspector (AGI)** — Google 官方

**验证指标：**
```
□ DrawCall 数量：GPU Driven 前 vs 后
□ CPU 帧时间：应大幅降低
□ GPU 帧时间：应略微降低或持平
□ 带宽消耗：通过 Profiler 验证
□ 功耗：通过 Battery Historian 验证
```

---

## 核心参考资料

| 类别 | 资源 | 用途 |
|------|------|------|
| Vulkan 入门 | [vulkan-tutorial.com](https://vulkan-tutorial.com/) | Vulkan 基础概念 |
| Vulkan 进阶 | [vkguide.dev](https://vkguide.dev/) | 现代 Vulkan 写法 |
| Filament 源码 | [github.com/google/filament](https://github.com/google/filament) | 移动端渲染引擎参考 |
| GPU Driven 经典 | [Wihlidal 2015 - GPU Driven Rendering](https://advances.realtimerendering.com/s2015/) | Assassin's Creed Unity 的 GPU Driven 方案 |
| GPU Driven 现代 | [Harada et al. - Forward+](https://takahiroharada.files.wordpress.com/2015/04/forward_plus.pdf) | Forward+ 渲染管线 |
| TBDR 深度 | [ARM Mali Best Practices](https://developer.arm.com/documentation/101897/latest) | 移动端 GPU 架构 |
| Vulkan 移动端 | [Vulkan Mobile Best Practices](https://github.com/KhronosGroup/Vulkan-Samples/tree/main/samples/performance) | Khronos 官方移动端最佳实践 |
| Compute Culling | [Arseny Kapoulkine - niagara](https://github.com/zeux/niagara) | 开源 GPU Driven 渲染器参考 |

---

## 术语速查表

| 术语 | 全称 | 含义 |
|------|------|------|
| TBDR | Tile-Based Deferred Rendering | 移动端 GPU 渲染架构 |
| IMR | Immediate Mode Rendering | PC GPU 渲染架构 |
| VMA | Vulkan Memory Allocator | Vulkan 内存管理库 |
| SSBO | Shader Storage Buffer Object | Shader 可读写的大容量 Buffer |
| Hi-Z | Hierarchical-Z | 层级化深度图，用于遮挡剔除 |
| AABB | Axis-Aligned Bounding Box | 轴对齐包围盒 |
| MDI | Multi Draw Indirect | 一次 API 调用画多个物体 |
| Bindless | - | 无需逐 DrawCall 绑定资源的技术 |
