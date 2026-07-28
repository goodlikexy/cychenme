---
title: "Linux 内核学习路径"
description: "从操作系统原理、体系结构和启动流程出发，逐步进入进程、内存、I/O、文件系统与内核实验。"
layout: "linux-kernel"
---

这是 Linux 内核专题的导航页。内容按照“先建立地图，再理解机制，最后动手验证”的顺序整理；文章正文保留在独立的 Beautiful Article 页面中。

## 推荐学习顺序

1. 先看操作系统原理和内核整体架构，知道内核要解决什么问题。
2. 再补 CPU、ARM、x86 汇编和 Makefile 等源码阅读前置知识。
3. 沿着启动链路理解机器如何从固件进入 Linux 内核。
4. 进入进程管理、同步、内存管理和 I/O 等核心机制。
5. 最后用 QEMU、GDB 和实际实验验证源码中的调用路径。

## 前言与整体地图

- [操作系统原理总览：从资源管理到四个基本特性](/articles/linux-os-principles-overview-flowchart/)
- [Linux Kernel 内核整体架构](/articles/linux-kernel-overall-architecture/)
- [Linux 内核架构和工作原理](/articles/linux-kernel-architecture/)
- [Linux 学习笔记](/study/linux/)

## 操作系统原理

- [操作系统原理总览：从资源管理到四个基本特性](/articles/linux-os-principles-overview-flowchart/)
- [Linux Kernel 内核整体架构](/articles/linux-kernel-overall-architecture/)

## 体系结构与汇编

- [计算机 Intel CPU 体系结构分析：指令执行、流水线、乱序执行与缓存](/study/intel-cpu-pipeline-basics/)
- [ARM 处理器架构总览：从 ISA 到 Cortex 与 SoC](/articles/linux-arm-architecture-processor-overview/)
- [x86 汇编基础：从寄存器到分段分页](/articles/linux-assembly-language-basics-x86/)
- [Linux 内核 Makefile 系统文件详解](/study/linux-kernel-makefile/)

## 启动与初始化

- [Linux 启动：从 BIOS 到内核入口](/articles/linux-kernel-boot-and-init/)
- [Linux 内核运行：从实模式切换到早期初始化](/articles/linux-kernel-early-init/)

## 进程管理与调度

- [Linux 进程状态模型：从三态到七态](/articles/linux-process-state-models/)
- [Linux 处理器调度：从三态选择到多级反馈队列](/articles/linux-processor-scheduling-basics/)
- [Linux 进程同步：从互斥临界区到 P/V 原语](/articles/linux-process-synchronization-principles/)

## 内存管理

- [Linux page cache：从文件读写到延迟回写](/articles/linux-disk-page-cache-mechanism/)
- [x86 汇编基础：从寄存器到分段分页](/articles/linux-assembly-language-basics-x86/)

## 文件系统、I/O 与存储

- [Linux 内核看 socket 底层的本质：I/O 模型](/articles/linux-socket-io-models/)
- [Linux page cache：从文件读写到延迟回写](/articles/linux-disk-page-cache-mechanism/)
- [Linux RAID：从磁盘阵列取舍到 mdadm 实验](/articles/linux-raid-disk-array-configuration/)

## 调试与实验环境

- [QEMU 调试 Linux 内核环境搭建](/study/qemu-linux-kernel-debugging/)
- [别再只会按 F5：VS Code 调试器新手教程](/articles/vscode-debugging-tutorial/)

## 后续计划

后续会继续补充进程管理、内存管理、VFS、设备驱动、网络协议栈、并发与同步、模块机制以及 eBPF 等专题。新增文章会优先归入本页对应分类，避免“学习”栏目中的技术内容继续平铺堆叠。
