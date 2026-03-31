# Filament Vulkan 内存管理策略笔记

> 源码路径：`filament/filament/backend/src/vulkan/`
> 涉及文件：`VulkanMemory.h/.cpp`、`VulkanBufferCache.cpp`、`VulkanStagePool.cpp`、`VulkanTexture.cpp`、`VulkanPlatform.cpp`

---

## 一、VulkanMemory.h/.cpp — 架构定位

这两个文件非常薄，职责明确：

### VulkanMemory.h：定义核心数据结构

```cpp
// Buffer 类型枚举
enum class VulkanBufferBinding : uint8_t {
    UNKNOWN, VERTEX, INDEX, UNIFORM, SHADER_STORAGE,
};

// GPU Buffer 的统一句柄结构体
struct VulkanGpuBuffer {
    VkBuffer        vkbuffer      = VK_NULL_HANDLE;
    VmaAllocation   vmaAllocation = VK_NULL_HANDLE;  // VMA 分配句柄
    VmaAllocationInfo allocationInfo;                // 含 mappedData 指针等
    uint32_t        numBytes      = 0;
    VulkanBufferBinding binding   = VulkanBufferBinding::UNKNOWN;
};
```

### VulkanMemory.cpp：VMA 实现单元

```cpp
// 唯一定义 VMA_IMPLEMENTATION 的编译单元
// 避免多 .cpp include vk_mem_alloc.h 导致符号重复
#define VMA_STATIC_VULKAN_FUNCTIONS 0
#define VMA_IMPLEMENTATION
#include "vk_mem_alloc.h"
```

> **设计原因**：VMA 是单头文件库，只能在一个 .cpp 中 `define VMA_IMPLEMENTATION`，
> Filament 将其隔离到 `VulkanMemory.cpp` 作为唯一实现单元。

---

## 二、VmaAllocator 初始化 — VulkanDriver.cpp:96

```cpp
VmaAllocatorCreateInfo const allocatorInfo{
    // 禁用 VMA 内部同步锁 → 性能优化
    // Filament Vulkan 后端是单线程的，由上层保证线程安全
    .flags = VMA_ALLOCATOR_CREATE_EXTERNALLY_SYNCHRONIZED_BIT,

    .physicalDevice   = physicalDevice,
    .device           = device,
    .pVulkanFunctions = &funcs,     // 动态加载函数指针（非静态链接）
    .instance         = instance,
};
vmaCreateAllocator(&allocatorInfo, &allocator);
```

**关键决策**：
- `VMA_ALLOCATOR_CREATE_EXTERNALLY_SYNCHRONIZED_BIT`：免去 VMA 内部 mutex，
  依赖后端自身的单线程保证，节省锁开销
- `VMA_DYNAMIC_VULKAN_FUNCTIONS = 1`：运行时动态加载 Vulkan 函数指针（BlueVK），
  而非静态链接，支持多平台/多 loader

---

## 三、GPU Buffer 分配 — VulkanBufferCache.cpp

所有 Vertex / Index / Uniform / SSBO 统一走 `VulkanBufferCache::allocate()`：

```cpp
VmaAllocationCreateInfo const allocInfo{
    .flags         = vmaFlags,
    .usage         = VMA_MEMORY_USAGE_AUTO,           // 让 VMA 自动选型
    .requiredFlags = VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT,  // 必须在 GPU 本地
};
vmaCreateBuffer(mAllocator, &bufferInfo, &allocInfo,
                &gpuBuffer->vkbuffer, &gpuBuffer->vmaAllocation,
                &gpuBuffer->allocationInfo);
```

### UMA 是什么

**UMA = Unified Memory Architecture（统一内存架构）**

```
独显（非 UMA）：                     移动端/集成显卡（UMA）：
CPU ──── [DDR 系统内存]              CPU ┐
              ↕ PCIe（带宽瓶颈）          ├── [同一块 LPDDR 物理内存]
GPU ──── [VRAM 显存]                 GPU ┘
```

**UMA 判定逻辑**（`VulkanPlatform.cpp:477`）：
```cpp
// 遍历所有内存堆，如果有任意一个堆不带 DEVICE_LOCAL → 存在独立 CPU 内存 → 非 UMA
for (uint32_t i = 0; i < memoryProperties.memoryHeapCount; ++i) {
    if ((memoryProperties.memoryHeaps[i].flags & VK_MEMORY_HEAP_DEVICE_LOCAL_BIT) == 0) {
        return false;  // 独显：CPU 堆无 DEVICE_LOCAL
    }
}
return true;  // 所有堆都是 DEVICE_LOCAL → UMA
```

典型 UMA 设备：高通骁龙 8 系、天玑 9000+、Apple M/A 系列、部分 AMD APU。

### UMA 特殊路径（VulkanBufferCache.cpp:143）

```cpp
if (mContext.isUnifiedMemoryArchitecture()) {
    vmaFlags |= VMA_ALLOCATION_CREATE_MAPPED_BIT                      // 持久 map
              | VMA_ALLOCATION_CREATE_HOST_ACCESS_SEQUENTIAL_WRITE_BIT; // CPU 顺序写标志
}
```

**非 UMA（独显）上传路径**（2 次写 + 1 次 GPU 复制）：
```
CPU memcpy → Staging Buffer（HOST_VISIBLE）
               → vkCmdCopyBuffer → Vertex Buffer（DEVICE_LOCAL）
```

**UMA 上传路径**（1 次写，完毕）：
```
CPU memcpy → Buffer（DEVICE_LOCAL + MAPPED，直接就是 GPU 内存）
```

**受益场景**：动态顶点、Uniform Buffer 每帧更新、GPU Driven 的 Scene Buffer 初始化。
**不受益场景**：静态 Mesh（只上传一次，差异微乎其微）。

**`HOST_ACCESS_SEQUENTIAL_WRITE` 的作用**：
告知 VMA "CPU 只顺序写，不随机读"，VMA 可选择不带 CPU cache 的内存路径，
减少 cache coherency 刷新开销，避免 UMA 的潜在性能代价。

### Buffer 类型 → Vulkan Usage 映射

| `VulkanBufferBinding` | `VkBufferUsageFlags` |
|-----------------------|---------------------|
| VERTEX | `VERTEX_BUFFER_BIT` |
| INDEX | `INDEX_BUFFER_BIT` |
| UNIFORM | `UNIFORM_BUFFER_BIT` |
| SHADER_STORAGE | `STORAGE_BUFFER_BIT` |

所有类型都自动叠加 `TRANSFER_DST_BIT`（支持通过 Staging 更新）。

---

## 四、Staging Buffer — VulkanStagePool.cpp

Staging Buffer 专用池，生命周期独立：

```cpp
// 创建：VMA_MEMORY_USAGE_CPU_ONLY
VmaAllocationCreateInfo allocInfo { .usage = VMA_MEMORY_USAGE_CPU_ONLY };
// 等价：HOST_VISIBLE | HOST_COHERENT

// 创建后立即持久 map
void* pMapping = nullptr;
vmaMapMemory(mAllocator, memory, &pMapping);
// Stage 存活期间 pMapping 始终有效，上层直接 memcpy
```

**Pool 复用机制**：
- Stage 上传完成后不销毁，插回 `mStages` 多重集合（key = 剩余容量）
- 下次请求时优先找容量足够的已有 Stage，减少 `vmaCreateBuffer` 频率
- 按帧 GC：超过 N 帧未使用的 Stage 才真正销毁

---

## 五、LAZILY_ALLOCATED — VulkanTexture.cpp + VulkanPlatform.cpp

### 设备能力检测（VulkanPlatform.cpp:1021）

```cpp
context.mLazilyAllocatedMemorySupported = false;

// 关键：RenderDoc 不支持 LAZILY_ALLOCATED，调试模式下跳过！
if constexpr (!FVK_RENDERDOC_CAPTURE_MODE) {
    for (uint32_t i = 0; i < typeCount; i++) {
        if (type.propertyFlags & VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT) {
            context.mLazilyAllocatedMemorySupported = true;
            // Spec 保证：LAZILY_ALLOCATED 一定同时有 DEVICE_LOCAL
            assert(type.propertyFlags & VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT);
            break;
        }
    }
}
```

### 实际使用（VulkanTexture.cpp:456）

```cpp
// TRANSIENT_ATTACHMENT 触发条件：Depth/Stencil 等帧内消耗型 Attachment
bool const useTransientAttachment =
    imageInfo.usage & VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT;

VkFlags const requiredMemoryFlags =
    VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT |
    (useTransientAttachment ? VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT : 0U) |
    (isProtected ? VK_MEMORY_PROPERTY_PROTECTED_BIT : 0U);

uint32_t memoryTypeIndex = context.selectMemoryType(memReqs.memoryTypeBits, requiredMemoryFlags);
```

**注意**：Texture 内存分配走的是**原生 `vkAllocateMemory`，不经过 VMA**，
原因是 LAZILY_ALLOCATED 内存需要严格绑定到特定 Attachment 生命周期，
VMA 的 pool sub-allocation 无法保证这种独占语义。

### RenderDoc 调试陷阱

> `if constexpr (!FVK_RENDERDOC_CAPTURE_MODE)`
>
> RenderDoc 捕获模式下**强制禁用** LAZILY_ALLOCATED。
> 原因：RenderDoc 需要在捕获时读回所有 Attachment 内容，
> 而 LAZILY_ALLOCATED 内存从未真正写入主存，RenderDoc 读不到正确内容。
> **调试时注意**：RenderDoc 下的内存用量和带宽数据与实机不同。

---

## 六、整体内存分层架构

```
                    Filament Vulkan 后端
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
   VulkanBufferCache   VulkanStagePool   VulkanTexture
   (VMA 管理)          (VMA 管理)        (原生 vkAllocateMemory)
           │               │               │
     DEVICE_LOCAL      CPU_ONLY         DEVICE_LOCAL
     (+ UMA 时可 map)  (HOST_VISIBLE    (TRANSIENT →
                        持久 map)        LAZILY_ALLOCATED)
           │               │
           └───────┬───────┘
                   ▼
            VmaAllocator
            (单一实例, 外部同步)
                   │
            VkDeviceMemory
            (大块, TLSF sub-alloc)
```

---

## 七、关键设计策略总结

| 策略 | 实现 | 原因 |
|------|------|------|
| Buffer 池复用 | `VulkanBufferCache` 缓存未使用 Buffer | 避免频繁 vmaCreateBuffer |
| Staging 池复用 | `VulkanStagePool` 持有 Stage 直到 GC | 减少 CPU_ONLY 内存的反复分配 |
| UMA 快速路径 | 检测到 UMA → Buffer 持久 map | 省去 Staging + CopyBuffer |
| LAZILY_ALLOCATED | 仅 Transient Attachment Image 使用 | TBDR On-Chip 带宽节省 |
| Texture 不走 VMA | 直接 vkAllocateMemory | LAZILY 需要独占 VkDeviceMemory |
| VMA 外部同步 | `EXTERNALLY_SYNCHRONIZED_BIT` | 后端单线程，免 mutex 开销 |
| RenderDoc 兼容 | 捕获模式禁用 LAZY | RenderDoc 无法读 LAZY 内存 |

---

## 八、与 GPU Driven 改造的关联

| 改造组件 | Filament 现有路径 | 需要的变化 |
|---------|-----------------|----------|
| GPU Scene SSBO | 无（CPU 端 RenderableManager）| 新增 `VulkanBufferBinding::SHADER_STORAGE`（已有！）路径，用 `VulkanBufferCache` 创建 |
| Indirect Draw Buffer | 无 | 需扩展 `VulkanBufferBinding` 加 `INDIRECT`，或复用 `SHADER_STORAGE` + 额外 usage flag |
| Culling Compute | 无 | 新增 Compute Pipeline，调度写 Indirect Buffer |

`SHADER_STORAGE` 对应的 `VK_BUFFER_USAGE_STORAGE_BUFFER_BIT` 已在 `getVkBufferUsage()` 中实现，
SSBO 路径**不需要新增 Buffer 类型**，可直接复用。

---

## 九、UMA 优化 vs LAZILY_ALLOCATED 优化 — 对比

这是两个**独立的**移动端优化，常被混淆：

| 维度 | UMA 快速路径 | LAZILY_ALLOCATED |
|------|------------|-----------------|
| **解决的问题** | 省去 CPU→GPU 数据复制 | 省去 Attachment 写回主存的带宽 |
| **作用对象** | Buffer（顶点、Uniform、SSBO）| Image（Depth、Stencil、MSAA）|
| **触发条件** | 所有内存堆都是 DEVICE_LOCAL | `TRANSIENT_ATTACHMENT_BIT` 用途 |
| **内存分配方** | VMA（`VMA_MEMORY_USAGE_AUTO`）| 原生 `vkAllocateMemory`（独占）|
| **典型节省** | 每帧动态数据上传的 PCIe 复制开销 | Depth Buffer 每帧数百 MB 写回带宽 |
| **RenderDoc** | 正常工作 | **捕获模式强制禁用** |
| **可同时生效？** | ✅ 两者独立，移动端高端机同时打开 | ✅ |

---

## 待深入

- [x] `VulkanBufferCache` 的 GC 策略：超过 **3 帧**未使用销毁（`TIME_BEFORE_EVICTION = 3`）
- [x] UMA 检测逻辑：所有堆都带 `DEVICE_LOCAL_BIT` 则为 UMA（`hasUnifiedMemoryArchitecture()`）
- [ ] `VulkanStagePool` 的容量策略：Stage 默认大小是多少（`STAGE_SIZE = 1048576` = 1MB）
- [ ] Texture 为什么不走 VMA：LAZILY 独占 VkDeviceMemory 的 Spec 要求
