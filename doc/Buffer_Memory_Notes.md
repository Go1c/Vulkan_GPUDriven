# Vulkan Buffer 与内存管理笔记

> Day 3 — 资源与内存
> 参考：vulkan-tutorial.com Vertex buffers / Uniform buffers 章节 + VMA 文档

---

## 一、Buffer 创建的完整流程

Vulkan 中 Buffer（逻辑对象）和 VkDeviceMemory（实际内存）是**两个独立对象**，必须手动绑定。

```
vkCreateBuffer()                    ← 创建逻辑 Buffer（只是描述，没有内存）
    ↓
vkGetBufferMemoryRequirements()     ← 查询：size、alignment、兼容的内存类型位掩码
    ↓
vkAllocateMemory()                  ← 真正分配 VkDeviceMemory
    ↓
vkBindBufferMemory()                ← 把内存绑到 Buffer 上
```

### 关键：`VkMemoryRequirements`

```cpp
VkMemoryRequirements memReqs;
vkGetBufferMemoryRequirements(device, buffer, &memReqs);
// memReqs.size           — 需要的内存字节数（可能 > 你请求的 size）
// memReqs.alignment      — offset 必须是此值的倍数
// memReqs.memoryTypeBits — 位掩码：bit i = 1 表示第 i 种内存类型兼容
```

选择内存类型：在 `memoryTypeBits` 兼容的类型里，找一个满足需求的：
```cpp
uint32_t findMemoryType(VkPhysicalDevice physDevice,
                        uint32_t typeBitsFilter,
                        VkMemoryPropertyFlags requiredProps) {
    VkPhysicalDeviceMemoryProperties memProps;
    vkGetPhysicalDeviceMemoryProperties(physDevice, &memProps);
    for (uint32_t i = 0; i < memProps.memoryTypeCount; i++) {
        bool compatible = (typeBitsFilter >> i) & 1;
        bool hasProps = (memProps.memoryTypes[i].propertyFlags & requiredProps) == requiredProps;
        if (compatible && hasProps) return i;
    }
    throw std::runtime_error("no suitable memory type");
}
```

---

## 二、内存类型详解

Vulkan 内存类型由 `vkGetPhysicalDeviceMemoryProperties()` 返回，每种是 Property Flags 的组合：

| 内存类型 | Flag | 特点 | 典型用途 |
|---------|------|------|---------|
| **DEVICE_LOCAL** | `DEVICE_LOCAL_BIT` | GPU 最快，CPU **不能**直接访问 | 静态 Vertex/Index Buffer、Texture |
| **HOST_VISIBLE** | `HOST_VISIBLE_BIT` | CPU 可 `vkMapMemory` 写入 | Staging Buffer、Uniform Buffer |
| **HOST_COHERENT** | `HOST_COHERENT_BIT` | 写入后无需手动 flush，对 GPU 立即可见 | 通常和 HOST_VISIBLE 搭配 |
| **HOST_CACHED** | `HOST_CACHED_BIT` | CPU 缓存加速读，需 flush/invalidate | GPU→CPU Readback |
| **LAZILY_ALLOCATED** | `LAZILY_ALLOCATED_BIT` | **移动端专用**，On-Chip Tile Memory | TBDR Depth/Stencil Attachment |

### LAZILY_ALLOCATED — 移动端带宽优化核心

TBDR GPU 的 Tile Memory 在 On-Chip，如果 Attachment 满足以下条件，GPU 无需将数据写回主存：
- `storeOp = STORE_OP_DONT_CARE`
- `finalLayout = DEPTH_STENCIL_ATTACHMENT_OPTIMAL`（帧内完全消费）

声明为 `LAZILY_ALLOCATED` 后，OS **不会预先分配物理内存**，GPU 直接使用 On-Chip 存储。节省带宽通常达到显著水平（Depth Buffer 每帧读写量可高达几百 MB）。

```cpp
// 移动端 Depth Buffer 最优配置
VkMemoryPropertyFlags required  = VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT;
VkMemoryPropertyFlags preferred = VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT;
// 用 VMA 时：allocInfo.preferredFlags = LAZILY_ALLOCATED_BIT
```

---

## 三、Vertex Buffer — Staging Pattern

静态几何数据（不会改变）的标准上传方式：

```
CPU → [Staging Buffer: HOST_VISIBLE] ──vkCmdCopyBuffer──► [Vertex Buffer: DEVICE_LOCAL]
```

**完整步骤：**

```cpp
// 1. 创建 Staging Buffer（CPU 可写，仅作传输源）
VkBufferCreateInfo stagingInfo = {
    .sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO,
    .size  = dataSize,
    .usage = VK_BUFFER_USAGE_TRANSFER_SRC_BIT,
};
// 内存类型：HOST_VISIBLE | HOST_COHERENT

// 2. Map → 写数据 → Unmap
void* data;
vkMapMemory(device, stagingMemory, 0, dataSize, 0, &data);
memcpy(data, vertices.data(), dataSize);
vkUnmapMemory(device, stagingMemory);
// HOST_COHERENT 保证写入对 GPU 立即可见，无需 vkFlushMappedMemoryRanges

// 3. 创建真正的 Vertex Buffer（GPU 本地，最快）
VkBufferCreateInfo vbInfo = {
    .size  = dataSize,
    .usage = VK_BUFFER_USAGE_TRANSFER_DST_BIT     // 接受 CopyBuffer
           | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT,   // 用于顶点输入
};
// 内存类型：DEVICE_LOCAL

// 4. 录制 Copy 命令（一次性 Command Buffer）
VkBufferCopy copyRegion = { .size = dataSize };
vkCmdCopyBuffer(cmd, stagingBuffer, vertexBuffer, 1, &copyRegion);

// 5. 提交并等待完成，然后销毁 staging buffer
```

**为什么不直接用 HOST_VISIBLE 作 Vertex Buffer？**
- 可以，但 HOST_VISIBLE 内存通过 PCIe 访问，GPU 读取带宽是 DEVICE_LOCAL 的 1/4 ~ 1/10
- 静态数据一次上传后长期使用，值得付出一次 Staging 的代价

---

## 四、Uniform Buffer — 每帧更新模式

Uniform Buffer 存储 MVP 矩阵、光照参数等每帧变化的数据：

```cpp
// 常见做法：HOST_VISIBLE + HOST_COHERENT，持久 Map
VkBufferCreateInfo uboInfo = {
    .size  = sizeof(UniformData),
    .usage = VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT,
};
// 内存类型：HOST_VISIBLE | HOST_COHERENT

// 持久 Map：创建后一次 map，全程有效
void* mappedPtr;
vkMapMemory(device, uniformMemory, 0, sizeof(UniformData), 0, &mappedPtr);

// 每帧更新：
UniformData ubo = { .mvp = camera.getMVP(), ... };
memcpy(mappedPtr, &ubo, sizeof(ubo));
// HOST_COHERENT: 无需 flush，GPU 读时自动看到最新值
```

### 多帧并行（In-Flight Frames）

GPU 渲染 Frame N 时，CPU 已在准备 Frame N+1，不能直接覆盖正在使用的 UBO：

```
Frame Index:  0           1           0           1
CPU写入:    [Frame0 UBO] [Frame1 UBO] [Frame0 UBO] ...
GPU读取:    [Frame0 UBO]             [Frame1 UBO]
```

解决方案：创建 `MAX_FRAMES_IN_FLIGHT`（通常 2）份 Uniform Buffer，按帧索引轮换：

```cpp
std::vector<VkBuffer>       uniformBuffers(MAX_FRAMES_IN_FLIGHT);
std::vector<VkDeviceMemory> uniformMemories(MAX_FRAMES_IN_FLIGHT);
std::vector<void*>          uniformMapped(MAX_FRAMES_IN_FLIGHT);
// 每帧：memcpy(uniformMapped[currentFrame], &ubo, sizeof(ubo));
```

---

## 五、Buffer 选型速查表

| 场景 | 内存类型 | Staging 需要？ | Usage Flags |
|------|---------|--------------|-------------|
| 静态顶点/索引 | DEVICE_LOCAL | 是 | VERTEX/INDEX_BUFFER + TRANSFER_DST |
| Uniform（per-frame） | HOST_VISIBLE + COHERENT | 否 | UNIFORM_BUFFER |
| SSBO（GPU Culling 输出）| DEVICE_LOCAL | 视情况 | STORAGE_BUFFER + TRANSFER_DST |
| Indirect Draw Buffer | DEVICE_LOCAL | 否（Compute写）| INDIRECT_BUFFER + STORAGE_BUFFER |
| Staging（CPU→GPU传输）| HOST_VISIBLE + COHERENT | — | TRANSFER_SRC |
| Depth/Stencil（移动端）| DEVICE_LOCAL + LAZILY | — | DEPTH_STENCIL_ATTACHMENT |
| Readback（GPU→CPU）| HOST_VISIBLE + CACHED | — | TRANSFER_DST |

---

## 六、VMA 简化版（Day 3 快速入门）

> 详细算法原理见 [VMA_Notes.md](./VMA_Notes.md)

手写内存管理的痛点：
1. `vkAllocateMemory` 次数上限 = 4096，不能每个 Buffer 独占一个
2. 对齐计算繁琐（`bufferImageGranularity`）
3. 内存碎片自己管理

**VMA 一行搞定 CreateBuffer + AllocateMemory + BindBufferMemory：**

```cpp
VkBufferCreateInfo bufferInfo = {
    .size  = dataSize,
    .usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT,
};
VmaAllocationCreateInfo allocInfo = {
    .usage = VMA_MEMORY_USAGE_GPU_ONLY,  // 自动选 DEVICE_LOCAL
};

VkBuffer buffer;
VmaAllocation allocation;
vmaCreateBuffer(allocator, &bufferInfo, &allocInfo, &buffer, &allocation, nullptr);
```

**`VmaMemoryUsage` 枚举对应关系：**

| VmaMemoryUsage | 等价内存类型 | 典型用途 |
|---------------|------------|---------|
| `GPU_ONLY` | DEVICE_LOCAL | 静态 Mesh、Texture |
| `CPU_TO_GPU` | HOST_VISIBLE + 尽量 DEVICE_LOCAL | Uniform、动态顶点 |
| `GPU_TO_CPU` | HOST_VISIBLE + HOST_CACHED | Readback |
| `CPU_ONLY` | HOST_VISIBLE + HOST_COHERENT | Staging Buffer |

**Staging Buffer + VMA 完整示例：**

```cpp
// Staging Buffer
VmaAllocationCreateInfo stagingAllocInfo = { .usage = VMA_MEMORY_USAGE_CPU_ONLY };
VkBuffer stagingBuf;  VmaAllocation stagingAlloc;
vmaCreateBuffer(allocator, &stagingBufInfo, &stagingAllocInfo,
                &stagingBuf, &stagingAlloc, nullptr);

// 写数据（VMA 封装了 map/unmap）
void* data;
vmaMapMemory(allocator, stagingAlloc, &data);
memcpy(data, vertices, dataSize);
vmaUnmapMemory(allocator, stagingAlloc);

// GPU Buffer
VmaAllocationCreateInfo gpuAllocInfo = { .usage = VMA_MEMORY_USAGE_GPU_ONLY };
VkBuffer vertexBuf;  VmaAllocation vertexAlloc;
vmaCreateBuffer(allocator, &vertexBufInfo, &gpuAllocInfo,
                &vertexBuf, &vertexAlloc, nullptr);

// 复制 + 销毁 staging
vkCmdCopyBuffer(cmd, stagingBuf, vertexBuf, 1, &copyRegion);
// 提交、等待后：
vmaDestroyBuffer(allocator, stagingBuf, stagingAlloc);
```

---

## 七、与 GPU Driven 的关联

Day 3 学到的内存概念，在后续 GPU Driven 实现中的对应：

| GPU Driven 组件 | Buffer 类型 | 内存类型 |
|----------------|------------|---------|
| GPU Scene Buffer（ObjectData[]）| SSBO | DEVICE_LOCAL（Compute 写，Graphics 读）|
| Indirect Draw Buffer | INDIRECT_BUFFER + STORAGE | DEVICE_LOCAL（Compute 写，Graphics 读）|
| Draw Count Buffer | STORAGE | DEVICE_LOCAL + HOST_VISIBLE（或 Compute atomic）|
| Hi-Z Depth Pyramid | Image | DEVICE_LOCAL |
| 全局 Vertex Buffer | VERTEX_BUFFER | DEVICE_LOCAL（一次 Staging 上传）|
| 全局 Index Buffer | INDEX_BUFFER | DEVICE_LOCAL（一次 Staging 上传）|

---

## 待深入

- [ ] Pipeline Barrier：Compute Write → Indirect Read 的正确写法
- [ ] `vkCmdPipelineBarrier` vs `VkMemoryBarrier` vs `VkBufferMemoryBarrier` 的选择
- [ ] 集成显卡（AMD APU / Intel）的内存模型差异（DEVICE_LOCAL + HOST_VISIBLE 同时成立）
- [ ] `VK_BUFFER_USAGE_SHADER_DEVICE_ADDRESS_BIT`（Bindless/RTX 需要）
