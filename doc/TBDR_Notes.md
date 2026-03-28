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

```
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
|------|------|
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
