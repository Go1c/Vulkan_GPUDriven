# TBDR 讨论纪要：Binning、全流程、并行与执行效率

> 整理自学习与讨论，便于对照 [GPU_Driven_Daily_Plan.md](./GPU_Driven_Daily_Plan.md) 中 **Day 4 — TBDR 架构深度理解**。具体微架构因厂商/代际而异，以官方文档与实测为准。

---

## 1. 术语与目标

- **TBDR（Tile-Based Deferred Rendering）** 在移动端常指：**先按瓦片（tile）对图元分桶，再按 tile 做光栅与着色**。其中 “Deferred” 多指 **分阶段 / 瓦片化**，不等同于经典 G-Buffer Deferred Shading。
- **Phase 1 — Binning / Tiling**：决定每个图元可能落在 **哪些 tile**，生成 **per-tile 图元列表**（及参数/索引等元数据，实现因硬件而异）。
- **Phase 2**：对每个 tile **只消费本 tile 列表**，在 **片上 tile 缓冲** 上做光栅、深度、混合等，再按需要写回显存。

---

## 2. Binning 的原理与实现要点

1. **输入几何**：顶点经 **VS（及可选 TS/GS）** → **装配** → **裁剪** → **透视除法 / NDC / Viewport** → 得到 **屏幕空间 2D 三角形**（与后续光栅一致）。
2. **Tile 判定**：对每个三角形在屏幕平面求 **保守覆盖**（常见为 **轴对齐包围盒 AABB** 与 tile 网格求交；可再对候选 tile 做更紧测试以减少假阳性）。**漏登记 tile 会导致洞，偏保守多登记几个 tile 可接受**（Phase 2 再精确丢弃）。
3. **输出**：每个 tile 一份 **图元引用列表**；全局常有 **参数缓冲**（图元索引、插值 setup 等）。写入方式常见为 **每 tile 原子追加** 或 **两遍法（计数 → 前缀和 → 填充）** 以减少竞争。
4. **API 边界**：Vulkan/Metal 应用层通常 **看不到** 显式 binning API；这是 **Render Pass 在 TBDR GPU 上** 的内部行为。

---

## 3. 顶点 Attribute 与 VS 改位置

- **Tile 判定只依赖「最终屏幕空间几何」**，即裁剪与 viewport 之后的位置；**UV、法线等 Attribute 不直接进入「三角形–tile 相交」公式**，除非经 **VS 写入了 `gl_Position`（或等价输出）** 从而 **间接** 改变覆盖区域。
- **Attribute 多** 主要增加 **顶点拉取与 VS 开销**，**不会** 因「属性多」而直接让 tile 集合变大。
- **在 VS 中修改位置**：Binning 使用的是 **同一套管线产出的最终顶点位置**，**不是**「忽略 VS 再算一遍 tile」。因此 **正常、合法的 VS 改 Pos 不会单独造成「tile 与光栅不一致」**；若出现怪异结果，优先查 **NaN/Inf、clip 空间约定、或自写模拟 binning 与 GPU 数学不一致**。

---

## 4. 常见误解：不是「两遍完整 VS」

- **两遍的是阶段**：**Phase 1（分桶）** 与 **Phase 2（按 tile 光栅+着色）**。
- **VS 对每个顶点实例在语义上执行一次**；Phase 2 给 PS 的是 **光栅器对三个顶点输出的插值**，**不是** 再跑一遍完整 VS。
- 实现上可能对顶点结果做 **缓存 / 参数缓冲 / 个别 replay**（带宽与容量折中），**不等于** API 意义上的「必须第二遍 VS」。

---

## 5. 完整过程（逻辑顺序）

```text
Draw / RenderPass
  → Vertex Fetch + VS (+ 可选 TS/GS)
  → 装配 → 裁剪 → 透视 → Viewport  →  屏幕空间三角形
  → 【Phase 1】Binning：per-tile 图元列表 + 参数元数据
  → 【Phase 2】逐 tile：读列表 → 光栅 → 插值 → PS → 混合（片上）→ 按需写回附件
  → EndRenderPass / Present
```

与 **IMR** 对照：TBDR 的 Phase 2 **不扫全场景图元流**，只扫 **本 tile 列表**；全屏附件的反复访问尽量落在 **片上**，从而降低对外部带宽的依赖。

---

## 6. 并行与「打断」

- **并行**：多按 **图元（或小批）** 并行做 AABB 与 scatter 写列表；写列表用 **原子** 或 **两遍 + 前缀和**。Phase 2 常按 **tile** 并行或流水调度。
- **打断 / 串行化常见原因**：
  - **Phase 1 → Phase 2** 的数据依赖（内部 barrier / 调度，应用层多为 Render Pass 语义）。
  - **两遍 binning** 的 **pass 间同步**（计数与填充之间）。
  - **同一 pass 内不当依赖 framebuffer**、复杂 **Subpass / input attachment** 导致 **flush、拆 pass 或非 tile 优化路径**（实现相关）。
  - **图元顺序敏感**（如部分透明/顺序相关语义）可能引入 **排序或保留顺序**，削弱纯乱序 binning。
  - **队列与同步**（fence/semaphore、CPU 等待）导致 GPU 空转。
  - **热点 tile 上原子竞争** 或 **内存带宽打满**，表现为有效并行度下降。

---

## 7. 影响 TBDR 执行效率的情况（重点）

### 7.1 破坏「片上 tile」优势（对带宽最敏感）

| 因素 | 影响 |
| ------ | ------ |
| 大附件 **`loadOp = LOAD`** | 每 tile 从显存 **读入** 已有内容，带宽显著上升。 |
| 多附件、高分辨率、**频繁 STORE + 下一 Pass 采样** | **写回与后续读** 增加，仍属常见路径，但总量需控制。 |
| Pass 过多、附件 ping-pong | **总 DRAM 流量** 上升，难以摊薄固定开销。 |

**优化倾向**：能 `CLEAR` 避免无谓 `LOAD`；合并 Render Pass；减少附件位数与分辨率；理解 `loadOp`/`storeOp` 与 TBDR 的关系。

### 7.2 Phase 1（Binning）变重

- 图元数量极大；**大量极小三角形**（每像素摊到的图元处理成本高）。
- 三角形 **跨很多 tile**（同一图元多次写入不同 tile 列表）。
- 列表 **假阳性多**（实现偏保守时 Phase 2 无效 work 增多）。

### 7.3 Phase 2（Tile 内）变重

- **同一 tile 内 overdraw** 严重（透明、粒子、early-Z 帮助小）。
- **过重 PS**（纹理、分支、寄存器压力 → occupancy 低）。
- **MSAA / 高样本**；**tile 负载不均**（最慢 tile 拖帧）。

### 7.4 用法与调度

- 频繁切换 RT、极碎的小 Pass；不当的 framebuffer 反馈路径；过粗的 CPU/GPU 同步。

### 7.5 小结

- **TBDR 更吃香**：少无谓全屏 LOAD、Pass 合理合并、opaque 多且 early-Z 有效、片上混合占比高。
- **TBDR 更吃亏**：全屏读写多、binning 列表与参数写爆、tile 内 overdraw + 重 PS、极小三角形洪水、Pass 过碎。

---

## 8. TBDR vs IMR 核心对比（Day 4 补充）

| 维度 | IMR | TBDR |
| ------ | ----- | ------ |
| Framebuffer 位置 | DRAM | 片上 SRAM（工作期间） |
| Depth Buffer 带宽 | 高（每像素读写 DRAM） | 极低（片上做完即丢） |
| Overdraw 代价 | 高（反复写 DRAM） | 低（片上覆盖，不出 tile） |
| 几何处理方式 | 立即处理 | 先全部 Binning，再 tile 内处理 |
| 大三角形性能 | 好 | 差（跨多 tile，重复分桶） |
| 极小三角形性能 | 差 | 更差（binning 开销 + 列表爆炸） |
| loadOp/storeOp 影响 | 较小 | **极大**（决定是否触碰 DRAM） |
| MSAA 代价 | 高（样本数 × DRAM 带宽） | 低（片上 resolve，近乎免费） |

**核心直觉：TBDR 的收益来自"片上 tile 不出 DRAM"。强制 LOAD 或多余 STORE 就破坏了这个优势。**

---

## 9. Phase 2 — Tile 内执行细节

### 9.1 片上 Tile Buffer 结构

```text
片上 SRAM（带宽是 DRAM 的 10~100x）：
┌─────────────────────────────┐
│  Color Buffer (RGBA8 等)    │
├─────────────────────────────┤
│  Depth Buffer (D24/D32)     │  ← 通常不写回 DRAM
├─────────────────────────────┤
│  Stencil Buffer (S8)        │  ← 通常不写回 DRAM
└─────────────────────────────┘
tile 大小：通常 16×16 或 32×32 像素
附件越多、位宽越高 → 片上容量压力越大
```

### 9.2 Tile 内执行顺序

```text
for each tile:
  1. loadOp  → CLEAR / DONT_CARE（不碰 DRAM）或 LOAD（读 DRAM，最贵）
  2. 按图元列表处理每个三角形：光栅 → HSR/Early-Z → PS → 混合（全在片上）
  3. storeOp → STORE（写回 DRAM）或 DONT_CARE（丢弃，零带宽）
```

### 9.3 HSR / Early-Z 在 TBDR 上更彻底

- **IMR Early-Z**：依赖 DRAM/Cache 中的深度值，乱序时需保守处理。
- **TBDR HSR**（如 ARM Mali）：Phase 2 前已知 tile 内所有图元，可对不透明图元排序，只对最终可见 fragment 执行 PS，理论上消除 overdraw 的 PS 开销。
- **破坏 HSR 的操作**：PS 写 `gl_FragDepth`、使用 `discard`/Alpha Test、半透明物体。

### 9.4 MSAA 在 TBDR 上近乎免费

- 多样本点全在片上，resolve 在片上完成，写回 DRAM 只写 resolved 结果。
- 代价：片上 SRAM 压力增加（4xMSAA = 4 倍 tile buffer 容量需求）。

---

## 10. loadOp / storeOp 策略

### 10.1 选项本质

| 选项 | DRAM 操作 | 适用场景 |
| ------ | ---------- | --------- |
| `loadOp = CLEAR` | 不读 DRAM，片上填充 | 每帧重新渲染（最常见） |
| `loadOp = DONT_CARE` | 不读 DRAM，不初始化 | 全屏覆写（全屏后处理） |
| `loadOp = LOAD` | **读 DRAM** | 在已有内容上继续绘制 |
| `storeOp = STORE` | **写 DRAM** | 后续 Pass 需要采样此附件 |
| `storeOp = DONT_CARE` | 不写 DRAM | Depth/Stencil/临时 RT |

### 10.2 各附件最优策略

| 附件 | loadOp | storeOp | 理由 |
| ------ | -------- | --------- | ------ |
| Color（主场景） | `CLEAR` | `STORE` | 每帧重绘；结果需要呈现 |
| Depth | `CLEAR` | `DONT_CARE` | 每帧重建；后续通常不采样 |
| Stencil | `CLEAR`/`DONT_CARE` | `DONT_CARE` | 同上 |
| GBuffer | `DONT_CARE` | `STORE` | 全屏覆写；Lighting Pass 需要 |
| MSAA Color | `CLEAR` | `DONT_CARE` + Resolve | 片上 resolve，只 STORE resolved |
| Shadow Map | `DONT_CARE` | `STORE` | 全屏写入；主 Pass 采样 |

### 10.3 三个危险反模式

1. **Depth 用 STORE**：1080p D24S8 每帧多写 ~8MB，严重消耗带宽。
2. **不必要的 LOAD**：全屏重绘却读入上一帧内容，读完立刻被覆盖，完全浪费。
3. **Pass 拆得过碎**：本可合并的 Pass 分开，中间附件反复落 DRAM 再读回。

### 10.4 Subpass + Input Attachment：终极优化

```text
经典 Deferred（移动端很贵）：
  Pass 1 → GBuffer 写回 DRAM → Pass 2 从 DRAM 读 GBuffer → Lighting

TBDR 最优（Subpass）：
  Subpass 0 → GBuffer 留在片上
  Subpass 1 → 直接从片上读 GBuffer → Lighting
  最终只写 Color 结果到 DRAM
```

### 10.5 决策流程

```text
需要已有内容？ → 是 → LOAD
              → 否 → 全屏覆写？ → 是 → DONT_CARE
                               → 否 → CLEAR

后续有消费者？ → 是 → STORE
              → 否 → DONT_CARE
```

### 10.6 与 GPU Driven / Meshlet 的关联

Meshlet 粒度影响 Binning：

- 切太细 → 三角形变小 → Binning 压力上升
- 切太粗 → Culling 效率下降
- 移动端经验值：**64~128 三角形/Meshlet**，覆盖范围尽量对齐 tile 边界

---

## 11. Filament 源码验证：loadOp / storeOp 实际策略

> 源码位置：`filament/backend/src/vulkan/VulkanFboCache.cpp`、`VulkanHandles.cpp`、`VulkanDriver.cpp`

### 11.1 核心决策代码（VulkanFboCache.cpp）

```cpp
// Color Attachment
.loadOp  = clear ? kClear : (discard ? kDontCare : kKeep),
.storeOp = (config.usesLazilyAllocatedMemory & (1 << i)) ? kDisableStore : kEnableStore,

// MSAA Resolve Attachment
.loadOp  = kDontCare,
.storeOp = kEnableStore,

// Depth Attachment
.loadOp       = clear ? kClear : (discardStart ? kDontCare : kKeep),
.storeOp      = discardEnd ? kDisableStore : kEnableStore,
.stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_DONT_CARE,  // 硬编码
.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE, // 硬编码
```

### 11.2 对比最优矩阵

| 附件 | 最优矩阵期望 | Filament 实际 | 结论 |
| ------ | ------------ | ------------- | ---- |
| Color loadOp | CLEAR / DONT_CARE | 三路分支，调用方决定 | ✅ 符合 |
| Color storeOp | STORE | 默认 STORE，Lazily 时跳过 | ✅ 更优 |
| MSAA 源 loadOp | DONT_CARE | kDontCare | ✅ 符合 |
| MSAA 源 storeOp | DONT_CARE | Lazily Allocated → kDisableStore | ✅ 更优 |
| MSAA Resolve storeOp | STORE | kEnableStore | ✅ 符合 |
| Depth loadOp | CLEAR | 调用方决定 clear/discard/keep | ✅ 符合 |
| Depth storeOp | DONT_CARE | 调用方通过 discardEnd 决定 | ✅ 符合 |
| Stencil | 全 DONT_CARE | 硬编码 DONT_CARE | ✅ 更激进 |

### 11.3 两处超出最优矩阵的设计

#### ① Lazily Allocated Memory（MSAA buffer）

```text
普通 Color Buffer → 分配真实 DRAM → storeOp = STORE
MSAA 源 Buffer   → Lazily Allocated → storeOp = DONT_CARE
                   → 物理内存从未分配，片上处理完直接丢弃
```

MSAA 源 buffer（多样本）永远不需要写回 DRAM，只有 resolve 结果（单样本）才写回。
`VulkanHandles.cpp` 在创建 MSAA texture 时打标记：

```cpp
if (msaaTexture && msaaTexture->isTransientAttachment()) {
    rpkey.usesLazilyAllocatedMemory |= (1 << index);
}
```

#### ② Stencil 硬编码 DONT_CARE

不给调用方选错的机会，Stencil 永远不落 DRAM。

### 11.4 VulkanFboCache vs VulkanHandles 职责分工

| 文件 | 职责 | 类比 |
| ------ | ---- | ---- |
| `VulkanHandles` | 资源本身（texture、RT、buffer 的数据结构与生命周期） | "名词"：描述资源是什么 |
| `VulkanFboCache` | RenderPass/Framebuffer 的缓存工厂，根据配置组装 loadOp/storeOp | "动词"：根据需求组装 RenderPass |

协作关系：`VulkanHandles` 设置 `usesLazilyAllocatedMemory` 等 flag → `VulkanFboCache` 消费这些 flag 决定 storeOp。

---

## 12. SwapChain 机制

### 12.1 本质

SwapChain 是 **GPU 渲染** 与 **显示器扫描** 两个异步过程之间的缓冲队列。

```text
GPU 渲染 → 写 backbuffer → Present（swap）→ 显示器扫描 frontbuffer
```

### 12.2 双缓冲 vs 三缓冲

| 模式 | 缓冲数 | 问题 |
| ---- | ------ | ---- |
| 双缓冲 | 2 | GPU 渲染完等显示器，卡顿或撕裂 |
| 三缓冲 | 3 | GPU 不等显示器，帧率稳、延迟低 |

### 12.3 Vulkan Present Mode

| 模式 | 特点 | 适用 |
| ---- | ---- | ---- |
| `IMMEDIATE` | 立刻显示，不等 VSync，有撕裂 | PC 追求帧率 |
| `FIFO` | 等 VSync，无撕裂 | 移动端默认（强制省电） |
| `FIFO_RELAXED` | 队列空时不等 VSync，低帧率减延迟 | 帧率不稳场景 |
| `MAILBOX` | 新帧覆盖等待帧，无撕裂低延迟 | PC 三缓冲，移动端通常不支持 |

### 12.4 每帧固定流程

```text
vkAcquireNextImageKHR()   → 拿空闲 image，触发 imageAvailable semaphore
        ↓
等 imageAvailable 信号
        ↓
录制 CommandBuffer → beginRenderPass（目标此 image）→ 渲染 → endRenderPass
        ↓
触发 renderFinished semaphore
        ↓
vkQueuePresentKHR()       → 等 renderFinished → 送显示系统
```

### 12.5 第一帧特殊处理（VulkanDriver.cpp）

```cpp
if (rt->isSwapChain()) {
    if (sc->isFirstRenderPass()) {
        discardStart |= TargetBufferFlags::COLOR; // 强制 DONT_CARE
        sc->markFirstRenderPass();
        acquireNextSwapchainImage();
    }
}
```

**原因：** SwapChain image 第一次 acquire 时内容未定义（可能是垃圾数据）。
多 View 场景下上层默认传 LOAD（后续 View 需要叠加前一个 View 的结果），
Driver 层在第一次使用时强制覆盖为 DONT_CARE，避免读入垃圾数据的无效带宽消耗。
这是**防御性封装**：把平台细节隔离在 Driver 层，上层渲染逻辑不需要感知"是否第一帧"。

### 12.6 SwapChain 与 TBDR 的关系

```text
TBDR 片上 Tile Buffer（工作区）
        ↓ storeOp = STORE（唯一不能优化掉的 STORE）
SwapChain backbuffer（DRAM）
        ↓ vkQueuePresentKHR
显示器
```

SwapChain 的 Color **必须 STORE**，是整条渲染管线中唯一强制落 DRAM 的输出。

---

## 13. Khronos Vulkan Mobile Best Practices — 官方实测数据

> 来源：[Khronos Vulkan-Samples / render_passes](https://github.com/KhronosGroup/Vulkan-Samples/tree/main/samples/performance/render_passes)，测试平台 1080p ~60FPS。

### 13.1 官方实测带宽数据

**Color loadOp 的影响（External Read Bytes）：**

```text
LOAD_OP_LOAD  → 1533.9 MiB/s
LOAD_OP_CLEAR →  933.7 MiB/s
节省 600.2 MiB/s ≈ 理论值 2220×1080×4B×60 ≈ 575 MiB/s
```

**Depth storeOp 的影响（External Write Bytes）：**

```text
STORE_OP_STORE     → 986.3 MiB/s
STORE_OP_DONT_CARE → 431.5 MiB/s
节省 554.8 MiB/s ≈ 同一张全屏图的尺寸
```

**两项合计：一帧节省 ~1155 MiB/s 无效带宽。**
移动端 DRAM 总带宽约 10~30 GB/s，这两项优化即可节省 4~11%。

### 13.2 官方文档补充的三个要点

#### ① 不要用 `vkCmdClear*` 代替 loadOp

```text
错误：loadOp = DONT_CARE + vkCmdClearColorImage()
  → 移动端退化为 per-fragment clear shader
  → 每秒多消耗 ~600 万 fragment cycles

正确：loadOp = CLEAR
  → 硬件在 tile 初始化阶段直接填充，零 fragment 开销
```

#### ② Depth image 加 `TRANSIENT_ATTACHMENT` 标志

```cpp
// 加这个标志告诉驱动：此 image 只在一个 RenderPass 内存活
depth_info.usage = VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT
                 | VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT;

// 配合 Lazily Allocated → 物理内存从不分配
depth_alloc.preferredFlags = VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT;
```

比 `storeOp = DONT_CARE` 更彻底：连 DRAM 物理内存都不分配。

#### ③ Render Area Granularity 对齐

```cpp
VkExtent2D granularity;
vkGetRenderAreaGranularity(device, renderPass, &granularity);
// renderArea 的 offset 和 extent 必须对齐 granularity
// 不对齐 → 边缘 tile 产生额外开销
```

与 Binning 章节呼应：tile 边界对齐是移动端性能的基础要求。

### 13.3 官方 Best Practice 总结

**必须做：**

- 每个 attachment 在 Pass 开始时用 `CLEAR` 或 `DONT_CARE`
- 不再使用的 attachment（Depth/Stencil）用 `storeOp = DONT_CARE`
- 临时 attachment 加 `TRANSIENT_ATTACHMENT` + Lazily Allocated 内存

**绝对不要做：**

- `loadOp = LOAD` 除非算法真的依赖上一帧内容
- 用 `vkCmdClearColorImage()` 代替 `loadOp = CLEAR`
- 用 shader 手写 clear（消耗 fragment cycles）
- 对 RenderPass 内不需要的 attachment 设置 loadOp/storeOp（触发无效的 tile memory 往返）

---

## 14. Subpasses 深度解析

> 来源：[Khronos Vulkan-Samples / subpasses](https://github.com/KhronosGroup/Vulkan-Samples/tree/main/samples/performance/subpasses)

### 14.1 实测数据（Mali-G76，2220×1080）

```text
双 RenderPass 方案：物理 Tile 数 = 614.7k/s
单 RenderPass + Subpass：物理 Tile 数 = 262.2k/s
节省 ≈ 55% tile 处理量 = 55% 带宽
```

### 14.2 为什么 Subpass 更高效

```text
双 RenderPass（传统 Deferred，移动端很贵）：
  Pass 0（Geometry）→ G-Buffer 写回 DRAM
  Pass 1（Lighting）→ 从 DRAM 读 G-Buffer → 计算 → 写 Color

单 RenderPass + Subpass（TBDR 最优）：
  Subpass 0（Geometry）→ G-Buffer 留在片上 tile memory
  Subpass 1（Lighting）→ 直接从片上读（vkCmdNextSubpass = NOP）
  最终只写 Color 结果到 DRAM
```

### 14.3 Subpass 合并（Merging）条件 — ARM 规定

| 条件 | 说明 |
| ---- | ---- |
| 两个 subpass 有共享数据 | 不共享数据的 subpass 不会被合并 |
| color attachment 总数 ≤ 8 | depth/stencil 不计入此限制 |
| depth/stencil attachment 不变 | 两个 subpass 用同一个 depth |
| MSAA 采样数相同 | 所有 attachment 一致 |
| **G-Buffer ≤ 128bit/pixel** | Mali-G72+ 放宽到 256bit/pixel |

**G-Buffer 超出预算的代价：** tile 缩小 → 物理 tile 数从 262.2k/s 涨到 409.6k/s，带宽接近翻倍。

### 14.4 G-Buffer 布局建议

```text
Lighting  (RGBA8_SRGB)    = 32bit  ← 可享受 Transaction Elimination
Depth     (D32_SFLOAT)    = 不计入 128bit 限制
Albedo    (RGBA8_UNORM)   = 32bit
Normal    (RGB10A2_UNORM) = 32bit
总计：32 + 32 + 32 = 96bit/pixel < 128bit ✅
```

Position 从 Depth 重建，不需要单独的 Position buffer：

```glsl
mat4 inv_view_proj = inverse(projection * view);
vec4 clip    = vec4(in_uv * 2.0 - 1.0, subpassLoad(i_depth).x, 1.0);
vec4 world_w = inv_view_proj * clip;
vec3 world   = world_w.xyz / world_w.w;
```

### 14.5 Transient Attachment 正确用法

G-Buffer 中 depth/albedo/normal 只在 RenderPass 内存活，必须标记：

```cpp
image_info.usage = VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT
                 | VK_IMAGE_USAGE_INPUT_ATTACHMENT_BIT;
alloc_info.preferredFlags = VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT;
```

不这样做的代价：fragment jobs 翻倍（56/s → 113/s），驱动被迫写回 DRAM。

---

## 15. MSAA 正确用法

> 来源：[Khronos Vulkan-Samples / msaa](https://github.com/KhronosGroup/Vulkan-Samples/tree/main/samples/performance/msaa)

### 15.1 核心结论

```text
正确（片上 inline resolve）：+3% 带宽，4x MSAA 几乎免费
错误（独立 resolve pass）：+5 GB/s 带宽，耗电 500mW（20% 功耗预算）
```

### 15.2 正确做法 — pResolveAttachments 片上 Resolve

```cpp
// MSAA 源 buffer：transient + lazily allocated
load_store[i_color_ms].store_op = VK_ATTACHMENT_STORE_OP_DONT_CARE;
image_info.usage |= VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT;
alloc_info.preferredFlags = VK_MEMORY_PROPERTY_LAZILY_ALLOCATED_BIT;

// Subpass 里配置 resolve → 片上完成，resolve 结果写入 swapchain
subpass->set_color_resolve_attachments({i_swapchain});
```

### 15.3 错误做法 — vkCmdResolveImage

```cpp
// 移动端性能炸弹：强制 MSAA buffer 写回 DRAM 再读回做 resolve
vkCmdResolveImage(...); // ← 禁止使用
```

### 15.4 Depth Resolve（需要后处理时）

需要将 MSAA depth 传给后处理 pass（如 SSAO）时：

```cpp
// 需要 Vulkan 1.2 / VK_KHR_depth_stencil_resolve
load_store[i_depth].store_op = VK_ATTACHMENT_STORE_OP_DONT_CARE;
subpass->set_depth_stencil_resolve_attachment(i_depth_resolve);
subpass->set_depth_stencil_resolve_mode(VK_RESOLVE_MODE_MIN_BIT);
// 可选：SAMPLE_ZERO / AVERAGE / MIN / MAX
```

### 15.5 最坏情形数据（4x MSAA，color+depth 都走独立 resolve）

```text
读带宽 +2366 MiB/s
写带宽 +3951 MiB/s
总计 +6.3 GB/s = 630mW（25% 功耗预算）

对比 inline resolve：
  4x MSAA 1080p @60FPS inline resolve  = 500 MB/s
  4x MSAA 1080p @60FPS 独立 resolve   = 3.9 GB/s
```

---

## 16. Layout Transitions 与 Transaction Elimination

> 来源：[Khronos Vulkan-Samples / layout_transitions](https://github.com/KhronosGroup/Vulkan-Samples/tree/main/samples/performance/layout_transitions)

### 16.1 Transaction Elimination 是什么

Mali GPU 为每张 image 维护**签名缓冲（Signature Buffer）**，记录每个 tile 的 CRC 摘要。写回时若 CRC 匹配（tile 内容未变），**跳过本次写回**。

```text
帧 N：tile[0] CRC = 0xABCD → 写回 DRAM，记录签名
帧 N+1：tile[0] CRC 仍 = 0xABCD → 命中 → 跳过写回
适用：静态 UI、不动的背景、HUD 叠加层
```

### 16.2 触发条件（同时满足）

- sample count = 1
- mip level = 1
- 使用 `COLOR_ATTACHMENT_BIT`
- **不**使用 `TRANSIENT_ATTACHMENT_BIT`
- tile size = 16×16（由 pixel data storage 决定）

### 16.3 "安全" vs "不安全" Layout

| 类型 | Layout | 说明 |
| ---- | ------ | ---- |
| **安全**（不破坏签名） | `COLOR_ATTACHMENT_OPTIMAL` | 最常用 |
| **安全** | `SHADER_READ_ONLY_OPTIMAL` | 采样时 |
| **安全** | `TRANSFER_SRC_OPTIMAL` | 作为拷贝源 |
| **安全** | `PRESENT_SRC_KHR` | 呈现前 |
| **不安全**（签名失效） | `UNDEFINED` | **最常见的错误** |
| **不安全** | `GENERAL` | 通用布局 |
| **不安全** | `TRANSFER_DST_OPTIMAL` | 作为拷贝目标 |

**例外：SwapChain image 从 `UNDEFINED` 转换是安全的**（驱动特殊处理）。

### 16.4 正确做法

```cpp
// 错误：每帧用 UNDEFINED，签名失效，transaction elimination 不触发
barrier.oldLayout = VK_IMAGE_LAYOUT_UNDEFINED;

// 正确：跟踪上一帧的实际 layout
barrier.oldLayout = last_known_layout; // 如 COLOR_ATTACHMENT_OPTIMAL
```

### 16.5 实测数据（Mali-G76）

```text
oldLayout = UNDEFINED：
  → CRC kill 数量低
  → 写带宽基准值

oldLayout = 上次实际 layout：
  → CRC kill 数量翻倍
  → 写带宽减少 ~10%
```

### 16.6 与 vkCmdBlitImage 的关系

`vkCmdBlitImage` 永远使签名失效，**用 shader-based blit 代替**。

---

## 17. Pipeline Barriers 原理与高效使用

> 来源：[Khronos Vulkan-Samples / pipeline_barriers](https://github.com/KhronosGroup/Vulkan-Samples/tree/main/samples/performance/pipeline_barriers)

### 17.0 为什么需要 Barrier

GPU 为了性能会激进地乱序执行、流水线重叠。**GPU 不会自动推断两个 Pass 之间的数据依赖**，这是 Vulkan 的设计哲学——把同步责任交给应用层，换取最大并行度。

```text
没有 Barrier 时：
  Pass1: [顶点][片元——写G-Buffer——]
  Pass2: [顶点][片元—读G-Buffer—]
                ↑
          Pass2 可能在 Pass1 写完前就开始读 → 画面错误

两个具体问题：
  1. 执行顺序：Pass2 的片元可能在 Pass1 写完之前启动
  2. 缓存可见性：Pass1 写完了，但数据还在 L1 Cache，Pass2 的核看不到
```

**Barrier = 同时解决这两个问题的栅栏：**

```text
Pass1 写完
    ↓
━━━━ Barrier ━━━━
  ① srcStage/srcAccess → 等这个写操作完成，刷出写缓存
  ② Layout 转换（可选）→ 重新组织内存布局
  ③ dstStage/dstAccess → 确保读缓存看到新数据
━━━━━━━━━━━━━━━━
    ↓
Pass2 开始读
```

### 17.1 GLES 对比：驱动隐藏了 Barrier

```text
GLES：驱动自动插入 Barrier，等价于最保守的全局屏障
  → 开发简单，但性能上限低（驱动无法猜到精确的依赖范围）

Vulkan：你手动插入，精确指定 Stage
  → 开发复杂，但 GPU 能重叠更多工作

GLES 自动 Barrier ≈ srcStage=ALL_COMMANDS, dstStage=ALL_COMMANDS
                  = 昨天说的"禁止使用"的最坏组合
```

### 17.2 Barrier 的四个字段

```cpp
VkImageMemoryBarrier barrier = {
    .srcAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT, // 写了什么
    .dstAccessMask = VK_ACCESS_SHADER_READ_BIT,            // 读什么
    .oldLayout     = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL,
    .newLayout     = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL,
};

vkCmdPipelineBarrier(
    cmd,
    VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT, // 写在哪个阶段完成
    VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT,         // 读在哪个阶段开始
    ...
);
```

| 字段 | 含义 | 类比 |
| ---- | ---- | ---- |
| `srcStageMask` | 写操作在哪个流水线阶段完成 | "我几点下班" |
| `srcAccessMask` | 写操作的类型（刷出哪种缓存） | "我写了什么" |
| `dstStageMask` | 读操作在哪个流水线阶段开始 | "你几点上班" |
| `dstAccessMask` | 读操作的类型（让哪种读看到新数据） | "你要读什么" |

**Stage 决定执行顺序，Access 决定缓存刷新。两者缺一不可。**

### 17.3 Image Layout 转换为什么也在 Barrier 里

Layout 不只是一个标记，GPU 用不同的内存访问模式读写不同 Layout 的 Image：

```text
COLOR_ATTACHMENT_OPTIMAL  → 为"写颜色"优化（按 tile 组织）
SHADER_READ_ONLY_OPTIMAL  → 为"采样纹理"优化（按 mip 组织）
DEPTH_STENCIL_ATTACHMENT  → 为"深度测试"优化
PRESENT_SRC_KHR           → 为"显示输出"优化
```

Layout 转换本身是一次内存操作（重排数据、解压缩等），**必须在写完成后、读开始前执行**——这正好是 Barrier 的时间窗口，因此合并在一起：

```text
Barrier 原子完成三件事：
  ① 等写操作完成
  ② 执行 Layout 转换（重新组织内存）
  ③ 确保转换结果对后续可见
```

**`oldLayout = UNDEFINED` 的特殊用法：** 告诉 GPU"不在乎原内容"，GPU 跳过数据重排只做缓存刷新，更快。代价是原内容丢失（且会破坏 Transaction Elimination 签名）。

### 17.4 完整的一帧 Layout 生命周期

```text
Image 创建：      UNDEFINED
      ↓ Barrier
RenderPass 写颜色：COLOR_ATTACHMENT_OPTIMAL
      ↓ Barrier
下一 Pass 采样：   SHADER_READ_ONLY_OPTIMAL
      ↓ Barrier
Present：         PRESENT_SRC_KHR
```

**注：RenderPass 的 `initialLayout`/`finalLayout` 可自动完成部分转换，不用手写 Barrier（Day 5 内容）。**

### 17.1 Mali 两槽流水线

Mali GPU 有两个独立的硬件调度槽，可同时运行：

```text
┌──────────────────────────────────┐
│  顶点/计算槽（Vertex/Compute）    │  ← 可运行 第 N+1 帧的顶点
├──────────────────────────────────┤
│  片元槽（Fragment）               │  ← 同时运行 第 N 帧的片元
└──────────────────────────────────┘
Barrier 太宽 → 强制清空两个槽 → GPU 空转（气泡）
```

### 17.2 三种 Barrier 对比（Deferred 两 Pass 之间同步 G-Buffer）

#### ① 最保守（禁止使用）

```cpp
srcStageMask = VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT
dstStageMask = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT
// 结果：顶点与片元完全串行，两槽全部空转
```

#### ② 常见但次优

```cpp
srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT
dstStageMask = VK_PIPELINE_STAGE_VERTEX_SHADER_BIT
// 问题：G-Buffer 只在片元阶段采样，等到顶点阶段是浪费
// 结果：仍然串行，无改善
```

#### ③ 最优（推荐）

```cpp
srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT
dstStageMask = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT
// Pass 2 顶点与 Pass 1 片元可并行 → 帧时间提升 13%
```

```text
保守 Barrier：[Pass1 Vert][Pass1 Frag][____][Pass2 Vert][Pass2 Frag]
最优 Barrier：[Pass1 Vert][Pass1 Frag]
                               [Pass2 Vert][Pass2 Frag]
              ← Pass2 顶点与 Pass1 片元重叠，气泡消失 →
```

### 17.3 核心规则

| 规则 | 说明 |
| ---- | ---- |
| `srcStageMask` 尽量**晚** | 写操作在哪个阶段完成，就写哪个阶段 |
| `dstStageMask` 尽量**早** | 读操作在哪个阶段开始，就写哪个阶段（不要提前） |
| 避免反向依赖 | Fragment → Vertex 会引入调度气泡 |
| Pass 间标准写法 | `COLOR_ATTACHMENT_OUTPUT → FRAGMENT_SHADER` |

### 17.4 禁止使用的组合

```text
BOTTOM_OF_PIPE_BIT → TOP_OF_PIPE_BIT     ← 全流水线清空
ALL_GRAPHICS_BIT   → ALL_GRAPHICS_BIT    ← 全流水线清空
ALL_COMMANDS_BIT   → ALL_COMMANDS_BIT    ← 全流水线清空
```

### 17.5 其他注意事项

- **VkEvent**：不要在 signal 后立即 wait，改用 `vkCmdPipelineBarrier`
- **VkSemaphore**：只用于队列间同步，不用于同队列内同步
- **TRANSFER 操作**：尽量避免，会打断流水线；优先用 zero-copy 算法

---

---

## 18. AFBC — ARM 帧缓冲无损压缩

> 来源：[Khronos Vulkan-Samples / afbc](https://github.com/KhronosGroup/Vulkan-Samples/tree/main/samples/performance/afbc)，需要 Mali G-51+ 且驱动 r16p0+

### 18.1 是什么

AFBC 是 Mali GPU **硬件层的实时无损压缩**，驱动自动应用，对应用代码完全透明。

```text
官方实测（Sponza，Samsung Galaxy S10）：
  AFBC 关闭：788 MiB/s 写带宽
  AFBC 开启：528 MiB/s 写带宽
  节省 33%（官方最高可达 50%）
```

### 18.2 与其他优化的关系

```text
loadOp/storeOp 优化  → 减少"不必要的"读写（逻辑层优化）
AFBC                 → 压缩"必要的"读写（物理层压缩）
ASTC 纹理压缩        → 压缩"输入纹理"（离线压缩，与 AFBC 独立）

三者互不替代，全部叠加生效。
```

### 18.3 触发条件（全部满足才启用）

```cpp
// 必须满足：
VkSampleCountFlagBits == VK_SAMPLE_COUNT_1_BIT  // 不能是 MSAA
VkImageType           == VK_IMAGE_TYPE_2D
VkImageTiling         == VK_IMAGE_TILING_OPTIMAL
// 格式在支持列表内（Mali-G77+ 支持所有 ≤32bit/pixel 格式）

// 不能包含以下 flag（任何一个都会禁用 AFBC）：
VK_IMAGE_USAGE_STORAGE_BIT           // ← 最常见的意外禁用
VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT
VK_IMAGE_CREATE_ALIAS_BIT
VK_IMAGE_CREATE_MUTABLE_FORMAT_BIT
```

### 18.4 最常见的意外禁用

```cpp
// 错误：不需要 compute 写入却加了 STORAGE_BIT
swapchainImageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT
                    | VK_IMAGE_USAGE_STORAGE_BIT;  // ← AFBC 被禁，带宽 788 MiB/s

// 正确：按需设置，不加多余 flag
swapchainImageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT;
                                                    // AFBC 启用，带宽 528 MiB/s
```

**不要"以防万用"加 `STORAGE_BIT`，这是最常见的 AFBC 误杀。**

### 18.5 与 MSAA 的关系

```text
MSAA 源 buffer（SAMPLE_COUNT > 1）→ 不能用 AFBC
MSAA Resolve 目标（SAMPLE_COUNT = 1）→ 可以用 AFBC

这也是 pResolveAttachments 片上 resolve 的额外收益：
  resolve 后的单样本 image 才能享受 AFBC 压缩
```

### 18.5.1 VK_IMAGE_USAGE_STORAGE_BIT 详解

`STORAGE_BIT` 声明这张 Image 要被 Compute Shader 以**随机读写**方式访问：

```glsl
// Compute Shader 中：
layout(set=0, binding=0, rgba8) uniform image2D storageImage;
imageLoad(storageImage, ivec2(...));   // 随机读
imageStore(storageImage, ivec2(...), color); // 随机写
```

**与普通纹理采样的区别：**

| | 普通纹理（SAMPLED_BIT） | Storage Image（STORAGE_BIT） |
| - | ---------------------- | ---------------------------- |
| 访问方式 | 只读，走采样器 | 可读写，直接内存地址 |
| 硬件路径 | 纹理单元（支持过滤/mip） | 直接内存操作 |
| AFBC 兼容 | ✅ 纹理单元能解压 | ❌ 绕过纹理单元，无法处理压缩 |

**为什么 STORAGE_BIT 禁用 AFBC：** AFBC 改变了内存布局，`imageLoad/imageStore` 按原始地址访问，无法处理压缩格式，驱动必须禁用 AFBC。

**需要 STORAGE_BIT 的场景：** Compute 后处理（Bloom/SSAO）、GPU 粒子、GPU Driven Culling 结果写入。

**不需要的场景：** 普通 RenderTarget、SwapChain Image（最常被误加）。

**需要 Compute 写入又想保留 AFBC 时：**

```cpp
// 分两张 Image：
imageA.usage = VK_IMAGE_USAGE_STORAGE_BIT;           // Compute 写（无 AFBC）
imageB.usage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT
             | VK_IMAGE_USAGE_SAMPLED_BIT;            // 最终 RT（有 AFBC）
// Compute 写 imageA → copy 到 imageB → 后续采样 imageB
```

### 18.6 验证方法

```cpp
// 方法 1：用 VK_EXT_image_compression_control 扩展查询压缩状态
// 方法 2：Streamline 观察 L2_EXT_WRITE_BEATS 计数器
//         开启 AFBC 后写带宽会明显下降
```

---

## 19. 全部官方文档汇总 Best Practice

| 类别 | 规则 | 违反代价 |
| ---- | ---- | -------- |
| **RenderPass loadOp** | 用 `CLEAR`/`DONT_CARE`，禁用无谓 `LOAD` | +600 MiB/s 读带宽 |
| **RenderPass storeOp** | Depth 用 `DONT_CARE`，禁用无谓 `STORE` | +555 MiB/s 写带宽 |
| **Subpass** | 合并 G-Buffer pass，G-Buffer ≤ 128bit/pixel | tile 数/带宽翻倍 |
| **Transient** | depth/G-Buffer 加 `TRANSIENT` + `LAZILY_ALLOCATED` | fragment jobs 翻倍 |
| **MSAA** | 用 `pResolveAttachments`，禁用 `vkCmdResolveImage` | +5 GB/s，+500mW |
| **Layout** | 跟踪实际 layout，不滥用 `UNDEFINED` | transaction elimination 失效，+10% 写带宽 |
| **Clear** | 用 `loadOp = CLEAR`，禁用 `vkCmdClear*` | +600 万 fragment cycles/s |
| **Barrier** | `srcStageMask` 晚，`dstStageMask` 早，Pass 间用 `COLOR_ATTACHMENT_OUTPUT→FRAGMENT_SHADER` | 流水线气泡，帧时间 +13% |
| **AFBC** | 不加多余 `STORAGE_BIT`，format 选支持列表内，不用 MSAA 源 buffer | -33% 写带宽 |
