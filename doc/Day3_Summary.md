# Day 3 总结 — Vulkan 资源与内存

> 详细笔记见 [Buffer_Memory_Notes.md](./Buffer_Memory_Notes.md) | [VMA_Notes.md](./VMA_Notes.md)

---

## Buffer 创建四步曲

```
vkCreateBuffer
    → vkGetBufferMemoryRequirements
    → vkAllocateMemory
    → vkBindBufferMemory
```

`VkBuffer` 只是描述（无内存），`VkDeviceMemory` 才是实体。
`memoryTypeBits` 是位掩码，bit i = 1 表示第 i 种内存类型兼容该 Buffer。

---

## 内存类型三大类

| 类型 | 访问方 | 关键特征 |
|------|--------|---------|
| `DEVICE_LOCAL` | GPU | 带宽最高，CPU 不可直接访问 |
| `HOST_VISIBLE + COHERENT` | CPU 写 → GPU 读 | 可 `vkMapMemory`，写入立即对 GPU 可见，无需 flush |
| `LAZILY_ALLOCATED` | 移动端 TBDR 专用 | 数据驻留 On-Chip Tile Memory，配合 `DONT_CARE` 永不写回主存 |

`HOST_CACHED`：CPU 读回（Readback）场景，需手动 `vkInvalidateMappedMemoryRanges`。

---

## 两种核心使用模式

### Staging Pattern（静态数据）

```
CPU → [Staging: HOST_VISIBLE] → vkCmdCopyBuffer → [Vertex Buffer: DEVICE_LOCAL]
```

- 一次上传，永久 GPU 高速访问
- Staging 上传完成后立即销毁

### 持久 Map（每帧更新数据）

```cpp
vkMapMemory(..., &mappedPtr);   // 创建后 map 一次，全程有效
// 每帧：
memcpy(mappedPtr, &ubo, sizeof(ubo));
```

- 配合 `MAX_FRAMES_IN_FLIGHT`（通常 2）轮换独立 Buffer，避免 CPU/GPU 竞争写同一块内存

---

## 与 GPU Driven 的连接

| GPU Driven 组件 | Buffer Usage | 内存类型 | 写入方 |
|----------------|-------------|---------|--------|
| GPU Scene Buffer（ObjectData[]）| STORAGE_BUFFER | DEVICE_LOCAL | Compute |
| Indirect Draw Buffer | INDIRECT_BUFFER + STORAGE | DEVICE_LOCAL | Compute |
| 全局 Vertex / Index Buffer | VERTEX/INDEX_BUFFER | DEVICE_LOCAL | 一次 Staging |
| Hi-Z Depth Pyramid | Image | DEVICE_LOCAL | Compute |
| Depth / Stencil（移动端）| DEPTH_STENCIL_ATTACHMENT | DEVICE_LOCAL + LAZILY | — |

---

## 关键结论

1. **不要每个 Buffer 独占一个 `vkAllocateMemory`**，上限仅 4096；改用 VMA Sub-allocation
2. **静态 Mesh 一定要走 Staging → DEVICE_LOCAL**，GPU 读取带宽差异可达 4-10×
3. **移动端 Depth Buffer 必须声明 LAZILY_ALLOCATED + storeOp DONT_CARE**，这是 TBDR 带宽节省的核心手段
4. **Indirect Buffer 由 Compute Shader 写入，不需要 CPU 介入**，Pipeline Barrier 负责 Compute→Graphics 的内存可见性

---

## Filament 内存管理：三个模块

> 详细分析见 [Filament_Memory_Notes.md](./Filament_Memory_Notes.md)

| 模块 | 文件 | 管理对象 | 使用 VMA？ |
|------|------|---------|-----------|
| `VulkanBufferCache` | BufferCache.cpp | Vertex / Index / Uniform / SSBO | ✅ `VMA_MEMORY_USAGE_AUTO` |
| `VulkanStagePool` | StagePool.cpp | Staging Buffer（上传专用，持久 map）| ✅ `VMA_MEMORY_USAGE_CPU_ONLY` |
| `VulkanTexture` | Texture.cpp | Image / Attachment | ❌ 原生 `vkAllocateMemory` |

---

## Filament 内存管理：三个关键设计发现

**1. UMA 快速路径**（移动端骁龙 / Apple M 系列）

UMA = Unified Memory Architecture，CPU/GPU 共享同一块物理内存，无需跨 PCIe 复制。
判定方式：所有内存堆都带 `VK_MEMORY_HEAP_DEVICE_LOCAL_BIT` → UMA。

```
非 UMA：CPU → Staging（HOST_VISIBLE）→ vkCmdCopyBuffer → Buffer（DEVICE_LOCAL）
   UMA：CPU → memcpy 直接写 Buffer（DEVICE_LOCAL + 持久 map），一步完成
```

```cpp
if (mContext.isUnifiedMemoryArchitecture()) {
    vmaFlags |= VMA_ALLOCATION_CREATE_MAPPED_BIT                      // 持久 map
              | VMA_ALLOCATION_CREATE_HOST_ACCESS_SEQUENTIAL_WRITE_BIT; // 减少 cache coherency 开销
}
```

受益场景：动态顶点、Uniform Buffer、GPU Driven Scene Buffer 初始化。

**2. LAZILY_ALLOCATED 触发条件**（`VulkanTexture.cpp:456`）
```cpp
// 只有 TRANSIENT_ATTACHMENT 才申请 LAZILY — Depth/Stencil 帧内消耗型 Attachment
bool useTransient = imageInfo.usage & VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT;
VkFlags required  = VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT
                  | (useTransient ? VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT : 0U);
```

**3. RenderDoc 调试陷阱**（`VulkanPlatform.cpp:1024`）
```cpp
// RenderDoc 不支持 LAZILY_ALLOCATED → 捕获模式强制禁用！
// 注意：RenderDoc 下内存用量和带宽数据与实机不符
if constexpr (!FVK_RENDERDOC_CAPTURE_MODE) {
    // 检测并启用 mLazilyAllocatedMemorySupported
}
```

---

## UMA 优化 vs LAZILY_ALLOCATED — 两者不同，可同时生效

| | UMA 快速路径 | LAZILY_ALLOCATED |
|-|------------|-----------------|
| **解决问题** | 省 CPU→GPU 数据复制 | 省 Attachment 写回主存带宽 |
| **作用对象** | Buffer（顶点/Uniform/SSBO）| Image（Depth/Stencil/MSAA）|
| **触发条件** | 所有堆都是 DEVICE_LOCAL | `TRANSIENT_ATTACHMENT_BIT` |
| **RenderDoc** | 正常 | **捕获模式强制禁用！** |
| **可同时生效** | ✅ 移动端高端机两者都打开 | ✅ |

---

## GPU Driven 改造好消息

`SHADER_STORAGE` → `VK_BUFFER_USAGE_STORAGE_BUFFER_BIT` 已在 `VulkanBufferCache` 中实现。
**GPU Scene Buffer 和 Indirect Buffer 可直接复用现有 BufferCache 机制，无需新增底层 Buffer 类型。**
