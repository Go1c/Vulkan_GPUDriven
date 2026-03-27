# GPU Driven 渲染技术笔记
> 整理自技术讨论，涵盖移动端 GPU Driven 渲染、Hi-Z 遮挡剔除、Two-Pass Occlusion Culling 完整知识体系

---

## 一、为什么需要 GPU Driven

### 传统渲染的瓶颈

传统渲染是 **CPU 主导**的模式，CPU 负责遍历、剔除、排序、逐一提交 DrawCall，GPU 大量时间在等待 CPU，DrawCall 数量成为性能瓶颈：

- PC：DrawCall 超过 2000~3000 开始明显下降
- 移动端：500~1000 个就是瓶颈

```
传统模式：
CPU [剔除] → [排序] → [提交DrawCall × N]   ← 瓶颈
                  ↓↓ N次传输 ↓↓
GPU [渲染] [渲染] [渲染]...                 ← 大量时间等CPU
```

### GPU Driven 的核心思路

把决策权从 CPU 转移到 GPU，CPU 只负责一次性上传数据和触发 Compute：

```
GPU Driven 模式：
CPU：上传数据（1次）+ 触发Compute（1次）
           ↓
GPU Compute：[剔除 × 万线程并行] → [写 DrawArgs]
GPU Render： [DrawIndirect × 个位数]
```

### 量化收益对比（千人同屏场景）

| 指标 | 传统方式 | GPU Driven |
|---|---|---|
| 每帧 DrawCall 数量 | ~1000+ | 个位数 |
| CPU 剔除耗时 | 3~8ms/帧 | 接近 0ms |
| GPU 等待 CPU 时间 | 明显 | 基本消除 |
| 物体数扩展性 | CPU 线性变慢 | GPU 并行，近线性扩展 |

---

## 二、移动端特殊性：TBDR 架构

移动端 GPU（Mali / Adreno / Apple GPU）均为 **TBDR（Tile-Based Deferred Rendering）** 架构，与 PC 的 IMR 架构有本质差异：

| | PC（IMR） | 移动端（TBDR） |
|---|---|---|
| 渲染方式 | 立即模式 | 分 Tile 延迟渲染 |
| 带宽压力 | 相对充裕 | 极度敏感 |
| Framebuffer 读取 | 有带宽开销 | Tile 内免费 |

### TBDR 下 GPU Driven 的 Pass 顺序要求

Compute Shader 写 Buffer 后立刻 Render 会打断 Tile 流程，造成隐性 Flush。正确做法：

```
✅ 正确顺序：
Pass 1：Render Occluders（独立 RenderPass，触发 Tile Flush）
Pass 2：Compute 生成 Hi-Z Mip 链
Pass 3：Compute Occlusion Cull → 写 IndirectBuffer
Pass 4：DrawIndirect 主渲染

Pass 之间必须有明确的 vkCmdPipelineBarrier
```

---

## 三、GPU Driven 系统的 Buffer 全景

### 输入类 Buffer（CPU 写，GPU 读）

```hlsl
// 1. Instance Data Buffer - 所有实例基础数据
struct InstanceData {
    float4x4 localToWorld;
    float4   boundingSphere;
    float3   aabbMin;
    float3   aabbMax;
    uint     meshID;
    uint     materialID;
    uint     lodGroupID;
    uint     flags;          // 位标记：SKIP_HIZ / ALWAYS_VISIBLE 等
};

// 2. Mesh Info Buffer - 每种 Mesh 的绘制元信息
struct MeshInfo {
    uint indexCount;
    uint indexOffset;
    uint vertexOffset;
    uint lodCount;
    float lodDistances[4];
};

// 3. Camera/View Buffer - 每帧更新
struct CameraData {
    float4x4 viewProj;
    float4x4 viewProjPrev;   // 上一帧（Two-Pass 用）
    float4   frustumPlanes[6];
    float3   cameraPos;
    float    nearZ, farZ;
};
```

### 状态类 Buffer（跨帧持久化）

```hlsl
// 4. Visibility Buffer - 每个实例上一帧是否可见（Two-Pass 核心）
RWBuffer<uint> visibilityBuffer;
// 大小：ceil(实例总数 / 32) × 4 字节，10000实例 ≈ 1.25KB

// 5. LOD Selection Buffer
RWBuffer<uint> lodBuffer;
```

### 中间计算类 Buffer（单帧生命周期）

```hlsl
// 6. Culling Result Buffer（Two-Pass 各一份）
RWStructuredBuffer<uint> visibleInstancesPass1;
RWStructuredBuffer<uint> visibleInstancesPass2;

// 7. Hi-Z Buffer（Hierarchical Z）
Texture2D<float> hizBuffer; // Mip 链，移动端推荐 512×512 起
```

### 输出类 Buffer（GPU 写，DrawIndirect 消费）

```hlsl
// 8. Indirect Args Buffer
struct IndirectArgs {
    uint indexCount;      // 固定值
    uint instanceCount;   // GPU 动态写入
    uint firstIndex;
    int  vertexOffset;
    uint firstInstance;
};

// 9. Culled Instance Buffer - 剔除后紧凑排列的可见实例
RWStructuredBuffer<PerInstanceData> culledInstances;
```

### 内存估算（1000单位场景）

| Buffer | 大小 | 生命周期 |
|---|---|---|
| Instance Data Buffer | ~80KB | 持久 |
| Visibility Buffer | ~4KB | 持久跨帧 |
| Hi-Z Buffer | ~700KB | 单帧 |
| Indirect Args Buffer | ~1KB | 单帧 |
| Culled Instance Buffer | ~40KB | 单帧 |
| **合计** | **~1MB** | - |

---

## 四、Two-Pass Occlusion Culling

### 核心问题：深度图从哪来

| 方案 | 原理 | 缺点 |
|---|---|---|
| 手动 Occluder | 美术标记遮挡体 | 需人工维护 |
| 深度重投影 | 上一帧深度投影到当前帧 | 精度不保守，可能画面错误 |
| **Two-Pass** | 上一帧可见物体渲染做深度图 | 仅 1 帧误差窗口 |

### 完整流程

```
【帧开始】
    ↓
① Compute（First Pass）
   遍历所有对象：上一帧 visible=1 → 写 DrawArgs（可选 Frustum Cull）
    ↓
② Indirect Draw（First Pass）
   渲染上一帧可见对象 → 生成深度图
    ↓
③ Compute 生成 Hi-Z Mip 链
    ↓
④ Compute（Second Pass）
   遍历【所有】对象：
   - Frustum Cull
   - Hi-Z Occlusion Cull
   - 通过 且 上一帧未渲染 → 写 DrawArgs
   - 更新 Visibility Buffer（供下帧使用）← 必须遍历所有对象的原因
    ↓
⑤ Indirect Draw（Second Pass）
   渲染新出现的可见对象
【帧结束，Visibility Buffer 已更新】
```

### Second Pass 为什么要遍历所有对象

**必须更新所有对象的 Visibility Buffer，包括上一帧可见的对象：**

```
若跳过上一帧可见的对象：
第N帧：物体A可见 → VB[A]=1
第N+1帧：A被遮挡，但 Pass2 跳过A → VB[A] 永远=1 → A永远被渲染
```

### 四种状态转换

| 上帧 VB | 本帧 visible | 结果 |
|---|---|---|
| 0 | false | 不画，VB→0，正确 |
| 0 | true | Pass2 画，VB→1，正确（新出现） |
| 1 | true | Pass1 画，Pass2 跳过，VB→1，正确 |
| 1 | false | Pass1 多画一帧（1帧残影），VB→0，下帧正确 |

### Second Pass 伪代码

```hlsl
void SecondPassCull(uint instanceID)
{
    bool frustumCulled = isFrustumCulled(instanceID);
    bool visible = !frustumCulled;

    if (visible) {
        bool occlusionCulled = HZBOcclusionCull(instanceID);
        visible = !occlusionCulled;
    }

    // 只渲染"本帧可见 且 Pass1 没画过"的物体
    bool shouldDraw = visible && (visibilityBuffer[instanceID] == 0);
    if (shouldDraw) {
        WriteDrawArgs(instanceID);
    }

    // 所有对象都更新 VB（这是必须遍历所有对象的原因）
    visibilityBuffer[instanceID] = visible;
}
```

---

## 五、Hi-Z（Hierarchical Z-Buffer）原理

### Hi-Z Mip 链的生成

对深度图做特殊降采样：每 2×2 个像素取**最大深度值**（最远）：

```
Mip0（原始）：每像素实际深度
Mip1：每 2×2 像素区域的最大深度
Mip2：每 4×4 像素区域的最大深度
MipN：每 2^N × 2^N 像素区域的最大深度

为什么取 Max？
保守原则：区域内最远的遮挡物深度
若被测物体比这个最远值还远 → 该区域内每个像素都能挡住它 → 安全剔除
```

### HZBOcclusionCull 完整实现

```hlsl
bool HZBOcclusionCull(uint instanceID, Texture2D hizBuffer, float4x4 viewProj)
{
    InstanceData inst = instanceBuffer[instanceID];

    // 第一步：定义 AABB 8 个角
    float3 aabbMin = inst.aabbMin;
    float3 aabbMax = inst.aabbMax;
    float3 corners[8] = {
        float3(aabbMin.x, aabbMin.y, aabbMin.z),
        float3(aabbMax.x, aabbMin.y, aabbMin.z),
        float3(aabbMin.x, aabbMax.y, aabbMin.z),
        float3(aabbMax.x, aabbMax.y, aabbMin.z),
        float3(aabbMin.x, aabbMin.y, aabbMax.z),
        float3(aabbMax.x, aabbMin.y, aabbMax.z),
        float3(aabbMin.x, aabbMax.y, aabbMax.z),
        float3(aabbMax.x, aabbMax.y, aabbMax.z),
    };

    float2 screenMin = float2(1, 1);
    float2 screenMax = float2(0, 0);
    float  minDepth  = 1.0;  // 物体最近深度

    [unroll]
    for (int i = 0; i < 8; i++)
    {
        float4 clipPos = mul(float4(corners[i], 1.0), viewProj);

        // 边界情况：角点在相机后方
        if (clipPos.w <= 0) return false; // 保守：不剔除

        // 钳制 Z，防止近平面穿越导致负深度
        clipPos.z = max(clipPos.z, 0);

        // 透视除法 → NDC
        clipPos.xyz /= clipPos.w;

        // NDC → UV，钳制到 [0,1]
        float2 uv = saturate(clipPos.xy * float2(0.5, -0.5) + 0.5);

        screenMin = min(screenMin, uv);
        screenMax = max(screenMax, uv);
        minDepth  = min(minDepth, clipPos.z); // 取最近深度
    }

    // 第三步：选择 Mip 级别
    float2 screenSize;
    hizBuffer.GetDimensions(screenSize.x, screenSize.y);
    float2 pixelSize = (screenMax - screenMin) * screenSize;

    // 亚像素物体：直接认为可见
    if (pixelSize.x < 1.0 && pixelSize.y < 1.0) return false;

    float mip = ceil(log2(max(pixelSize.x, pixelSize.y)));
    mip = clamp(mip, 0, MAX_MIP_LEVEL);

    // 超大物体：跳过 Hi-Z
    if (mip >= MAX_MIP_LEVEL) return false;

    // 第四步：采样 4 个角点
    float d_TL = hizBuffer.SampleLevel(samplerPoint, float2(screenMin.x, screenMin.y), mip);
    float d_TR = hizBuffer.SampleLevel(samplerPoint, float2(screenMax.x, screenMin.y), mip);
    float d_BL = hizBuffer.SampleLevel(samplerPoint, float2(screenMin.x, screenMax.y), mip);
    float d_BR = hizBuffer.SampleLevel(samplerPoint, float2(screenMax.x, screenMax.y), mip);

    float occluderDepth = max(max(d_TL, d_TR), max(d_BL, d_BR));

    // 第五步：比较判断
    // minDepth > occluderDepth：物体最近点比遮挡物最远点还远 → 被遮挡
    return (minDepth > occluderDepth);
}
```

### 为什么 4 次采样能覆盖整个矩形

Mip 选择公式保证：**矩形的宽高都 ≤ 该 Mip 一个纹素的大小**

```
2^mip ≥ max(pixelWidth, pixelHeight)
→ pixelWidth  ≤ 一个纹素宽
→ pixelHeight ≤ 一个纹素高

宽度 ≤ 1纹素 → 无论如何放置，最多跨 2 列
高度 ≤ 1纹素 → 最多跨 2 行
→ 最坏情况恰好是 2×2 = 4 个纹素
→ 4 个角点采样精确覆盖所有可能被矩形触碰的纹素
```

### 为什么用 minDepth（最近深度）

```
判断"整体是否被遮挡" = 判断"最难被遮挡的点是否也被遮挡"
最难被遮挡的点 = 离相机最近的点 = minDepth

若用 maxDepth：
物体前面是可见的，后面深度大 → maxDepth > occluderDepth → 错误剔除 ✗
```

---

## 六、特殊情况处理

### 完整边界情况清单

| 特殊情况 | 现象 | 处理方案 | 优先级 |
|---|---|---|---|
| Reversed-Z 配置错误 | 全局失效或画面错误 | HZB 改取 Min，比较逻辑取反 | 🔴 必须 |
| 透明物体写入深度图 | 背景物体消失 | 深度预渲染跳过透明物体 | 🔴 必须 |
| 蒙皮动画 AABB 不准 | 手脚消失 | 保守AABB / 跳过Hi-Z | 🔴 必须 |
| 相机在物体内部 | clipPos.w ≤ 0 | return false（保守） | 🔴 必须 |
| 物体跨越近平面 | 部分角 w ≤ 0 | return false（保守） | 🔴 必须 |
| 镜头切换 VB 未重置 | 性能抖动 1 帧 | 切场景时 memset(VB, 0) | 🟡 建议 |
| 亚像素物体 | 远处小物体闪烁 | pixelSize < 1 时 return false | 🟡 建议 |
| HZB 分辨率混用 | 剔除精度下降 | GetDimensions 用 HZB 尺寸 | 🟡 建议 |
| 极速相机运动 | 剔除失效 1 帧 | 可接受，Two-Pass 兜底 | 🟢 可接受 |
| 超大物体剔除无效 | 无剔除收益 | 白名单跳过 Hi-Z | 🟡 建议 |
| 多相机 VB 混用 | 小地图错误 | 每个相机独立 VB | 🟡 按需 |

---

## 七、大型物体白名单机制

### flags 标记位方案（推荐）

```hlsl
// flags 位定义
#define FLAG_SKIP_HIZ        (1 << 0)  // 跳过 Hi-Z，仍做视锥剔除
#define FLAG_ALWAYS_VISIBLE  (1 << 1)  // 永远可见，跳过所有剔除
#define FLAG_CAST_SHADOW     (1 << 2)
#define FLAG_RECEIVE_SHADOW  (1 << 3)

// Compute Shader 判断
InstanceData inst = instanceBuffer[instanceID];

if (inst.flags & FLAG_ALWAYS_VISIBLE) {
    WriteDrawArgs(instanceID);    // 直接渲染
    visibilityBuffer[instanceID] = 1;
    return;
}

if (inst.flags & FLAG_SKIP_HIZ) {
    if (FrustumCull(inst)) {      // 只做视锥剔除
        WriteDrawArgs(instanceID);
        visibilityBuffer[instanceID] = 1;
    }
    return;
}
// 正常走完整 Hi-Z 流程
```

### 各类型物体的推荐策略

```
地面 / Terrain          → FLAG_ALWAYS_VISIBLE
天空盒                   → 不进 GPU Driven，单独渲染
大型建筑（占屏 > 50%）   → FLAG_SKIP_HIZ
透明物体                  → FLAG_SKIP_HIZ
蒙皮动画单位              → FLAG_SKIP_HIZ 或保守 AABB
普通单位 / 小物体          → 完整 Hi-Z 流程
```

---

## 八、移动端降级方案

### 设备能力分级

| 级别 | 设备 | 方案 |
|---|---|---|
| Level 0 | 骁龙 865+ / Vulkan 1.2 | 完整 GPU Driven，DrawIndirectCount |
| Level 1 | 骁龙 750 / Vulkan 1.1 | 固定上限 + VS 内跳过不可见实例 |
| Level 2 | 中端机 / GLES 3.1 | GPU Cull + CPU 延迟读回（1帧延迟） |
| Level 3 | 骁龙 710 / GLES 3.0 | CPU 视锥剔除 + GPU Instancing |

### GLES 与 Vulkan 的 Indirect Draw 差异

| 能力 | GLES 3.1 | Vulkan 1.2 |
|---|---|---|
| 基础 Indirect Draw | ✅ | ✅ |
| MultiDrawIndirect（多Mesh一次Draw） | ❌ | ✅ |
| DrawIndirectCount（GPU写数量） | ❌ | ✅ |
| 驱动稳定性 | ⚠️ 厂商差异大 | ✅ 较稳定 |

### Level 1 降级实现（固定上限 + VS 跳过）

```hlsl
// Vertex Shader 内跳过不可见实例
InstanceData inst = instanceBuffer[instanceID];
if (inst.visible == 0) {
    // 输出退化三角形，不产生实际像素
    o.pos = float4(0, 0, 0, 0);
    return o;
}
```

---

## 九、Unity 中的实践

### 设备能力检测

```csharp
public enum GPUDrivenTier {
    Full,            // DrawIndirectCount + Compute Cull
    IndirectOnly,    // DrawIndirect + VS Skip
    DelayedReadback, // Compute Cull + CPU Delayed Count
    CPUCull          // CPU Cull + GPU Instancing
}

GPUDrivenTier DetectTier() {
    if (!SystemInfo.supportsComputeShaders)
        return GPUDrivenTier.CPUCull;
    if (!SystemInfo.supportsIndirectArgumentsBuffer)
        return GPUDrivenTier.CPUCull;
    if (SystemInfo.graphicsDeviceType == GraphicsDeviceType.Vulkan)
        return GPUDrivenTier.Full;
    return GPUDrivenTier.IndirectOnly;
}
```

### Unity 官方 GPU Driven 接口

- **BatchRendererGroup（BRG）**：Unity 2022+ 官方 GPU Driven 接口，无需 DOTS，支持 Vulkan/Metal
- `Graphics.RenderMeshIndirect`：灵活度高，需自行管理 Buffer

---

## 十、SLG 千人同屏推荐架构

```
战场单位（1000人）：
    ↓
空间分 Cluster（按战场区域）
    ↓
Compute：Frustum Culling（Cluster 级）
    ↓
Compute：Hi-Z Occlusion Culling（Instance 级）
    ↓
写 IndirectBuffer
    ↓
GPU Instancing + DrawIndirect
    ↓
单位动画：Animation Texture（VAT）或 GPU Skinning

城墙 / 建筑：FLAG_SKIP_HIZ（天然 Occluder，本身就是遮挡物）
地面：FLAG_ALWAYS_VISIBLE
透明特效：单独渲染，绕过 GPU Driven
```

---

## 参考资料

- [Experiments in GPU-based occlusion culling – Interplay of Light](https://interplayoflight.wordpress.com/2017/11/15/experiments-in-gpu-based-occlusion-culling/)
- [Two-Pass Occlusion Culling – Milos Kruskonja](https://medium.com/@mil_kru/two-pass-occlusion-culling-4100edcad501)
- [GPU-Driven Rendering Pipelines – Ulrich Haar & Sebastian Aaltonen](https://advances.realtimerendering.com/s2015/aaltonenhaar_siggraph2015_combined_final_footer_220dpi.pdf)
- [Nanite – UE5 虚拟几何体系统](https://advances.realtimerendering.com/s2021/Karis_Nanite_SIGGRAPH_Advances_2021_final.pdf)
