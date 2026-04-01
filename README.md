# Vulkan GPU Driven 学习笔记

> 30 天系统学习计划 | 基于 Filament 引擎 | 目标：Android 高端机 GPU Driven 完整管线

---

## 项目简介

本仓库记录从零开始系统学习 Vulkan + GPU Driven Rendering 的过程，以 [Filament](https://github.com/google/filament) 引擎源码为参考，聚焦移动端（Android）高性能渲染管线的设计与实现。

---

## 文档目录

| 文件 | 内容 |
|------|------|
| [GPU_Driven_Daily_Plan.md](doc/GPU_Driven_Daily_Plan.md) | 30 天每日执行计划（持续更新） |
| [GPU_Driven_Learning_Outline.md](doc/GPU_Driven_Learning_Outline.md) | 完整学习大纲 |
| [GPU_Driven_Rendering_Notes.md](doc/GPU_Driven_Rendering_Notes.md) | GPU Driven 渲染核心笔记 |
| [TBDR_Notes.md](doc/TBDR_Notes.md) | TBDR 架构深度理解（Binning、loadOp/storeOp、Subpass、MSAA、Barrier、AFBC） |
| [VMA_Notes.md](doc/VMA_Notes.md) | Vulkan Memory Allocator 学习笔记（TLSF 算法、内存类型） |
| [Buffer_Memory_Notes.md](doc/Buffer_Memory_Notes.md) | Vulkan Buffer 与内存管理笔记 |
| [Filament_Memory_Notes.md](doc/Filament_Memory_Notes.md) | Filament 内存管理源码分析 |
| [Mobile_Compatibility_Notes.md](doc/Mobile_Compatibility_Notes.md) | 移动端兼容性笔记（SSBO、GPU Driven 收益分析） |
| [Day3_Summary.md](doc/Day3_Summary.md) | Day 3 学习总结 |

---

## 学习路线

```
Week 1：Vulkan 基础 + TBDR 理论
Week 2：GPU Driven 核心技术（Meshlet、Culling、Indirect Draw）
Week 3：Filament 源码精读
Week 4：完整管线实现与优化
```

---

## 核心参考资料

- [Filament 引擎](https://github.com/google/filament)
- [Khronos Vulkan-Samples](https://github.com/KhronosGroup/Vulkan-Samples)
- [ARM Mali GPU Best Practices](https://developer.arm.com/documentation/101897)
- [Vulkan Memory Allocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)
