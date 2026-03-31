# 移动端兼容性与性能问题笔记

> 记录 GPU Driven 管线在移动端（TBDR 架构）落地时的已知问题与应对策略

---

## 一、SSBO 在移动端的四个核心问题

### 背景：SSBO 是什么

SSBO（Shader Storage Buffer Object）= GPU 上的大型可读写数组。

```glsl
layout(set=0, binding=0) buffer ObjectBuffer {
    ObjectData objects[];   // 运行时大小，可读写，支持原子操作
};
```

与 UNIFORM Buffer 的核心差异：

| 维度 | UNIFORM Buffer | SSBO |
|------|---------------|------|
| 读写 | 只读 | 可读可写 |
| 最大尺寸 | 16KB（保证最小值）| 128MB+（`maxStorageBufferRange`）|
| 可变长数组 | ❌ | ✅ |
| 原子操作 | ❌ | ✅ `atomicAdd / atomicMin` |
| 缓存行为 | 专用 constant cache | 走普通 L1/L2 |

GPU Driven 中 SSBO 承担：Scene Buffer（物体数据）、Indirect Buffer（Culling 输出）、Count Buffer（可见物体数）。

---

### 问题一：SSBO 读写走主存，不享受 Tile Memory

TBDR 的核心优化是让 Color/Depth Attachment 留在 On-Chip Tile Memory，
但 **SSBO 的地址在 Shader 运行时才确定**，GPU 无法预取到 On-Chip，每次访问都直接打 DRAM：

```
IMR（PC 独显）：  SSBO 读写 → L2 Cache → 影响有限
TBDR（移动端）：  SSBO 读写 → DRAM    → 每次都消耗带宽
```

**应对**：
- 用 `readonly` 修饰只读 SSBO，驱动可更激进地缓存
- 合并访问：一次读取尽量取完所需字段，避免多次随机访问同一对象

```glsl
// 推荐：readonly 告知驱动此 SSBO 不会被写入
layout(set=0, binding=0) readonly buffer SceneBuffer {
    ObjectData objects[];
};
```

---

### 问题二：Compute → Graphics Barrier 在移动端代价更高

PC 独显有独立的 Async Compute Queue，Compute 和 Graphics 可真正并行。
移动端（Mali/Adreno）通常**只有一个物理执行单元**，Barrier 会强制排空流水线：

```
PC（独显）：
  Compute Queue ──────────────────►
  Graphics Queue ──────────────────►   真正并行

移动端（Mali G 系列）：
  Compute ──► [Barrier 强制排空流水线] ──► Graphics   串行，有气泡
```

**应对**：
- Barrier 尽量粗粒度合并，避免多个细碎 Barrier
- 考虑把轻量 Culling 逻辑合并进一个 Compute Pass，减少 Barrier 次数
- 评估是否值得：场景物体少时（< 500），CPU Culling 的 Barrier 开销反而更低

---

### 问题三：Fragment Shader 写 SSBO 导致 HSR / Early-Z 失效

TBDR 的 Binning Pass 会提前确定每个 Tile 中哪些片元最终可见（HSR），
只 Shade 可见片元，大幅节省 ALU。

**但**：Fragment Shader 一旦写 SSBO，GPU 无法在 Binning 阶段确认哪些片元"最终存活"，
HSR 被迫关闭，所有片元都要执行 Shader：

```
正常 TBDR：
  Binning Pass → 确定可见片元 → 只 Shade 可见片元（节省 ALU）

Fragment Shader 写 SSBO 后：
  无法提前 Cull → 所有片元都执行 Shader → HSR 完全失效
```

**应对**：
- GPU Culling 的 SSBO 写入放在 **Compute Shader**，不放 Fragment Shader
- Filament 的设计已经遵守这一原则

---

### 问题四：`maxStorageBufferRange` 在旧移动设备上偏小

| 设备类型 | 典型 `maxStorageBufferRange` |
|---------|---------------------------|
| PC 独显 | 2GB+ |
| 高端移动（骁龙 8 Gen 2+）| 128MB ~ 2GB |
| 中低端移动（旧 Mali/Adreno）| **128MB（Vulkan 最低保证）**|

容量估算（`ObjectData ≈ 80 bytes`）：

```
10,000  物体 × 80B =   800KB  → 安全
100,000 物体 × 80B =     8MB  → 安全
500,000 物体 × 80B =    40MB  → 安全
1,500,000 物体 × 80B =  120MB  → 接近旧设备上限，需检查
```

**应对**：
- 初始化时查询并记录 `maxStorageBufferRange`
- 超出限制时降级为 CPU Culling + 传统 DrawCall
- 压缩 `ObjectData` 结构：Transform 用 3×4 矩阵（48B）替代 4×4（64B）

```cpp
// 初始化时检查
VkPhysicalDeviceProperties props;
vkGetPhysicalDeviceProperties(physDevice, &props);
uint32_t maxSSBORange = props.limits.maxStorageBufferRange;
bool canUseGPUDriven = maxSSBORange >= requiredSceneBufferSize;
```

---

## 二、移动端 SSBO 使用原则汇总

| 原则 | 原因 |
|------|------|
| SSBO 写入只在 Compute Shader | 避免 Fragment Shader 写导致 HSR/Early-Z 失效 |
| Graphics 侧只读 SSBO 加 `readonly` | 驱动缓存优化，减少 DRAM 访问 |
| 减少 SSBO 访问次数，合并读写 | 每次都是 DRAM 带宽消耗 |
| Barrier 粗粒度合并 | 移动端流水线排空代价高 |
| 启动前检查 `maxStorageBufferRange` | 旧设备上限仅 128MB |
| 少于 ~500 物体考虑 CPU Culling | Barrier + Compute 调度开销可能得不偿失 |

---

## 三、GPU Driven 在移动端的收益预期

```
主要收益（CPU 端）：
  ✅ DrawCall 提交从 O(N) 降为 O(1)
  ✅ CPU Culling 时间消除
  ✅ 场景物体越多，CPU 节省越显著

有限收益（GPU 端）：
  ⚠️  SSBO 访问走 DRAM，带宽增加
  ⚠️  Barrier 有流水线气泡
  ⚠️  与 PC 相比，GPU 端收益更小

结论：
  移动端 GPU Driven 的价值在于解放 CPU，而非优化 GPU。
  对 CPU bound 的场景（大量 DrawCall）收益最大。
```

---

## 待深入

- [ ] Adreno 和 Mali 对 Compute Shader 调度策略的差异
- [ ] `VK_KHR_shader_subgroup` 在 Culling Shader 中的加速潜力
- [ ] 移动端 Compute Workgroup 最优 `local_size`（64 vs 128 vs 256）
- [ ] Two-Phase Occlusion Culling 在移动端是否值得（Hi-Z 生成本身也有带宽）
