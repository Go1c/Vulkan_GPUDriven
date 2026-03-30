# Vulkan Memory Allocator (VMA) 学习笔记

> 参考：[官方文档](https://gpuopen-librariesandsdks.github.io/VulkanMemoryAllocator/html/) | [GitHub](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)

---

## 一、核心问题：为什么需要 VMA？

Vulkan 内存管理的三个核心难题：

1. **资源与内存分离**：`VkBuffer`/`VkImage` 和 `VkDeviceMemory` 是独立对象，必须手动绑定
2. **分配数量上限**：`VkPhysicalDeviceLimits::maxMemoryAllocationCount` 低至 4096，不能每个资源独占一个 `VkDeviceMemory`
3. **内存类型复杂**：不同 GPU 厂商、不同内存堆（VRAM/RAM/BAR）特性不同，需要手动选型

VMA 的本质是一个 **sub-allocator**：申请大块 `VkDeviceMemory`，然后在内部切分给上层使用。

---

## 二、整体架构

```
Application
    │
    ▼
VmaAllocator  ─── 全局唯一入口，per-VkDevice
    │
    ├── Default Memory Pools  (按 memoryTypeIndex 分组)
    │       └── VmaBlockVector
    │               └── VmaDeviceMemoryBlock[]
    │                       └── VmaBlockMetadata (算法实现)
    │
    ├── Custom Pools (用户自定义)
    │       └── 同上结构，但固定 memoryTypeIndex
    │
    └── Dedicated Allocations (独占整块 VkDeviceMemory)
```

**核心对象**：

| 对象 | 职责 |
|------|------|
| `VmaAllocator` | 全局管理器，线程安全入口 |
| `VmaBlockVector` | 管理同一 memoryType 下的所有 Block |
| `VmaDeviceMemoryBlock` | 封装一个 `VkDeviceMemory`，含元数据 |
| `VmaBlockMetadata` | 追踪 Block 内部的已用/空闲区间 |
| `VmaAllocation` | 返回给用户的句柄，表示一次分配 |

---

## 三、分配决策流程

当调用 `vmaCreateBuffer()` 时，内部走以下流程：

```
1. 在现有 Block 中找合适的空闲区间
        ↓ 失败
2. 申请新的 VkDeviceMemory Block（preferred size）
        ↓ 失败
3. 尝试 size/2, size/4, size/8 的 Block
        ↓ 仍失败
4. 回退到 Dedicated Allocation（独占整块 VkDeviceMemory）
        ↓ 仍失败
5. 换一种兼容的 memoryType，重新从步骤 1 开始
        ↓ 全部失败
6. 返回 VK_ERROR_OUT_OF_DEVICE_MEMORY
```

---

## 四、TLSF 算法（默认分配器）

> Two-Level Segregated Fit，来自实时系统领域学术论文。
> 目标：**O(1) 分配/释放 + 接近 best-fit 的碎片率**。

### 4.1 两级索引结构

```
                 FL (First Level)
              0   1   2   3   4   5  ...  (按 2^n 划分)
              │   │   │   │   │   │
              ▼   ▼   ▼   ▼   ▼   ▼
         SL: [0][1][2][3] [0][1][2][3] ...  (每级再细分 2^SL 个桶)
              │
              ▼
         Free Block 双向链表
```

- **FL（First Level）**：`floor(log2(size))`，即 size 最高有效位位置
- **SL（Second Level）**：把 FL 区间再等分为 `2^SL_BITS` 个子桶（VMA 中 SL_BITS=5，即 32 个子桶）

**SL 推导**：FL=f 的区间为 `[2^f, 2^(f+1))`，宽度 `2^f`，32 等分后每个子桶宽度：

```
sub_width = 2^f / 32 = 2^(f - 5)

sl = (size - 2^f) / sub_width
   = (size >> (f - 5)) - 32     ← 纯移位，无除法
```

**例：size=400，SL_BITS=5**

```
fl = 8   (256 ≤ 400 < 512)
sub_width = 2^(8-5) = 8
sl = (400 >> 3) - 32 = 50 - 32 = 18

桶 [8][18] 覆盖范围：[400, 408)
```

size 的二进制中，FL 位之后的 5 位就是 sl，直接用移位掩码提取，不需要除法：

```
size = 400 = 0b 1_10010_000
                ↑  └5位┘
               FL=8  SL=18(10010)
```

**为什么 SL=32（SL_BITS=5）最合理**：

| SL_BITS | 子桶数 | 最大内碎片 | 位图类型 | 查找指令 |
|---------|--------|-----------|---------|---------|
| 4 | 16 | size/16 ≈ 6.3% | uint16_t | 浪费寄存器 |
| **5** | **32** | **size/32 ≈ 3.1%** | **uint32_t** | **单条指令** |
| 6 | 64 | size/64 ≈ 1.6% | uint64_t | 稍重 |

32 个子桶正好填满一个 `uint32_t`，`BSF`/`CLZ`/`__builtin_ctz` 是单条 CPU 指令。SL=32 是**碎片率与硬件效率的最佳平衡点**。

**位图加速**：
```cpp
uint32_t m_IsFreeBitmap;              // bit[fl] = 1 表示该 FL 有空闲块
uint32_t m_SubIsFreeBitmap[MAX_FL];   // bit[sl] = 1 表示桶[fl][sl]有空闲块
```
用 `find first set (FFS)` 指令（`__builtin_ctz` / `_BitScanForward`）实现 O(1) 查找。

### 4.2 块结构

```
┌─────────────────────────────────┐
│  size (最低位：0=空闲, 1=已用)    │
│  prev_phys_block (物理紧邻前块)   │
├─────────────────────────────────┤ ← 以下仅空闲块有
│  next_free (同 free list 下一块) │
│  prev_free (同 free list 上一块) │
├─────────────────────────────────┤
│  用户数据区                       │
└─────────────────────────────────┘
```

- **物理链**（`prev_phys_block`）：按地址顺序连接，用于合并（coalescing）
- **逻辑链**（`prev_free`/`next_free`）：同一桶内的双向链表

### 4.3 分配流程（O(1)）

```
1. size 向上取整到对应桶的最小值（round up），保证找到的块一定够用
2. 计算 (fl, sl)
3. 查位图找第一个"≥ (fl,sl)"的非空桶（2 次位操作）
4. 从桶链表头取出块
5. 若块比请求大，split 剩余部分插入对应桶
6. 返回块
```

> **round up 的必要性**：不取整可能在桶中找到小于请求 size 的块（桶是范围，不是精确值）。

### 4.4 释放流程（O(1)）

```
1. 检查物理前邻是否空闲 → 空闲则从 free list 摘除，合并
2. 检查物理后邻是否空闲 → 空闲则从 free list 摘除，合并
3. 将合并后的大块插入对应 (fl, sl) 的 free list 头部
4. 更新位图
```

### 4.5 完整示例

```
初始：256KB Block，全空（NullBlock）

分配 A=48KB  → [0,48K)
分配 B=16KB  → [48K,64K)
分配 C=32KB  → [64K,96K)

释放 B       → [48K,64K) 进入 free list，前邻A/后邻C均已用，不合并

分配 D=12KB  → 从 B 的 16KB 块分裂：D=[48K,60K)，剩余4KB回 free list

释放 A       → [0,48K) 进入 free list，后邻D已用，不合并

释放 D       → 前邻A空闲，合并 → [0,60K) 插入新桶，位图更新
```

### 4.6 与其他算法对比

| 算法 | 时间复杂度 | 碎片情况 | 适用场景 |
|------|-----------|---------|---------|
| First-fit | O(n) | 中等 | 简单实现 |
| Best-fit | O(n log n) | 低 | 追求低碎片 |
| Buddy | O(log n) | 内碎片大 | 内核/简单场景 |
| **TLSF** | **O(1)** | **接近 best-fit** | **VMA、实时系统** |

---

## 五、VMA 中 TLSF 的特殊处理

### 5.1 Null Block（哨兵）

```cpp
VmaSuballocation* m_NullBlock;  // Block 末尾的特殊空闲块
```

末尾始终保留 Null Block 代表剩余空间，优先从普通 free list 分配，不足时才从 Null Block 切割。

### 5.2 bufferImageGranularity 对齐

Vulkan 规定 buffer 和 image 相邻时需满足 `bufferImageGranularity` 对齐，VMA 在分配时检查相邻块类型，必要时在 offset 上加 padding：

```
[Buffer alloc][padding][Image alloc]
              ← padding 保证跨越 granularity 页边界 →
```

### 5.3 VmaAllocation 零堆分配

`VmaAllocation` 对象来自 VMA 内部 pool allocator（类似 slab），高频分配路径完全无 CPU heap 压力，平均 0 次动态 `malloc`。

---

## 六、线性分配算法（可选）

开启：`VMA_POOL_CREATE_LINEAR_ALGORITHM_BIT`

适合生命周期相似的资源，支持四种模式：

| 模式 | 描述 |
|------|------|
| Free-at-once | 始终在末尾追加，全部释放后从头开始 |
| Stack (LIFO) | LIFO 释放可复用尾部空间 |
| Double Stack | 两端各一个栈，相向增长 |
| Ring Buffer | 游标循环，适合流式上传 |

**Double Stack 示意**：
```
低地址                              高地址
[────────上栈▶ ......... ◀下栈────]
  默认分配从左边长          upper address 从右边长
```

---

## 七、碎片整理（Defragmentation）

增量式、协作式，VMA 只负责内存语义，数据迁移由用户负责：

```cpp
vmaBeginDefragmentation(allocator, &info, &context);
while (vmaBeginDefragmentationPass(allocator, context, &passInfo)) {
    // 用户：vkCmdCopyBuffer / vkCmdCopyImage 迁移数据
    // 可设 pMoves[i].operation = IGNORE 跳过某块
    vmaEndDefragmentationPass(allocator, context, &passInfo);
}
vmaEndDefragmentation(allocator, context, &stats);
```

**约束**：
- 不跨 memoryType 移动
- 目标内存大小与原始一致（保留对齐）
- VMA 不录制 GPU 命令

---

## 八、Virtual Allocator

TLSF 算法可脱离 GPU 独立使用，管理任意内存范围（如大 VkBuffer 内的 sub-range）：

```cpp
VmaVirtualBlock virtualBlock;
vmaCreateVirtualBlock(&info, &virtualBlock);

VmaVirtualAllocation alloc;
VkDeviceSize offset;
vmaVirtualAllocate(virtualBlock, &allocInfo, &alloc, &offset);
```

---

## 九、设计哲学

| 维度 | 选择 |
|------|------|
| 接口语言 | C（与 Vulkan API 风格一致） |
| 实现语言 | C++14，单头文件 `vk_mem_alloc.h` |
| 依赖 | 仅标准库，无 STL 容器/RTTI/exception |
| 核心算法 | TLSF（默认）+ Linear（可选） |
| 错误处理 | 返回 `VkResult`，与 Vulkan 统一 |
| 职责边界 | 不上传数据、不录制命令，只管内存 |

---

## 待深入

- [ ] bufferImageGranularity 对齐处理细节
- [ ] TLSF 与 VmaBlockVector 扩容策略的配合
- [ ] Linear 分配器与 TLSF 的选择依据
- [ ] Defragmentation pass 设计细节
- [ ] VMA vs D3D12 Memory Allocator 对比
