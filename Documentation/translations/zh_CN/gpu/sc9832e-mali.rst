.. SPDX-License-Identifier: GPL-2.0

.. include:: ../disclaimer-zh_CN.rst

:Original: （本文档无对应英文原文，为 SC9832E Mali T820 GPU 支持的中文说明）

:作者:

 Unisoc 内核团队

.. _cn_sc9832e_mali:

================================================
SC9832E（SharkLE）Mali T820 GPU 支持说明
================================================

概述
====

紫光展锐（Unisoc）SC9832E 是一款面向入门级移动设备的 ARM64 SoC，内部集成了
ARM Mali-T820 MP1 GPU（属于 Mali Midgard 系列）。本文档说明了在 Linux 主线内
核中为该平台添加 GPU 支持所做的各项修改及其设计原理。

硬件信息
========

SC9832E SoC 的 GPU 子系统具有如下特征：

- **GPU 核心**：ARM Mali-T820 MP1（单核 Midgard 架构）
- **寄存器基地址**：``0x60300000``，寄存器空间大小 ``0x4000``（16 KB）
- **中断**：所有三路中断（job / mmu / gpu）共用一条 GIC SPI 39 中断线
- **核心时钟**（``core``）：来自 AON 时钟控制器的 ``CLK_NIC_GPU``，

  支持以下频率档位选择：256 MHz、307.2 MHz、384 MHz、512 MHz 或 GPLL 频率
- **总线使能时钟**（``bus``）：来自 AON APB 门控的 ``CLK_GPU_EB``，

  用于开关 GPU 在 AON APB 总线上的访问通道

为何三路中断可以共享同一 IRQ 线
================================

Panfrost 驱动在向内核注册 job、mmu、gpu 三个中断处理函数时，均使用了
``IRQF_SHARED`` 标志（参见 ``panfrost_gpu.c``、``panfrost_job.c``、
``panfrost_mmu.c``）。这意味着三个处理函数可以安全地绑定到同一条物理中断线
上。当中断触发时，三个处理函数依次被调用，各自通过读取硬件状态寄存器来判断
是否属于本处理函数负责的事件，若不属于则立即返回 ``IRQ_NONE``，否则进行处理
并返回 ``IRQ_HANDLED``。

这一设计与其他 Unisoc 平台（如 UMS9230）的实现保持一致。

所做的代码修改
==============

1. 设备树绑定（``arm,mali-midgard.yaml``）
------------------------------------------

在 ARM Mali Midgard GPU 的设备树绑定文档中，将 ``sprd,sc9832e-mali`` 加入了
与 ``arm,mali-t820`` 配对的 SoC 兼容字符串枚举列表。这使得设备树校验工具
（``dt-schema``）能够正确验证 SC9832E 的 GPU 节点。

修改位置：
``Documentation/devicetree/bindings/gpu/arm,mali-midgard.yaml``

2. SC9832E SoC 设备树（``sc9832e.dtsi``）
-----------------------------------------

在 SoC 公共设备树文件中新增 ``gpu`` 节点：

.. code-block:: devicetree

   gpu: gpu@60300000 {
       compatible = "sprd,sc9832e-mali", "arm,mali-t820";
       reg = <0x0 0x60300000 0x0 0x4000>;
       interrupts = <GIC_SPI 39 IRQ_TYPE_LEVEL_HIGH>,
                    <GIC_SPI 39 IRQ_TYPE_LEVEL_HIGH>,
                    <GIC_SPI 39 IRQ_TYPE_LEVEL_HIGH>;
       interrupt-names = "job", "mmu", "gpu";
       clocks = <&aon_clk CLK_NIC_GPU>, <&aonapb_gate CLK_GPU_EB>;
       clock-names = "core", "bus";
       status = "disabled";
   };

各字段说明：

- ``compatible``：第一项为紫光展锐特有字符串，第二项为 ARM 通用 T820 字符串，
  Panfrost 驱动通过后者匹配。
- ``reg``：Mali T820 寄存器基地址及大小。
- ``interrupts`` / ``interrupt-names``：三条中断均映射到 GIC SPI 39，
  名称分别为 ``job``（作业调度）、``mmu``（内存管理单元故障）、``gpu``
  （通用 GPU 事件），详见上一节说明。
- ``clocks`` / ``clock-names``：
  - ``core``（``CLK_NIC_GPU``）：GPU 核心频率时钟，可动态调频。
  - ``bus``（``CLK_GPU_EB``）：GPU 总线使能门控时钟。
- ``status = "disabled"``：默认关闭，由板级 DTS 负责开启，避免在不具备
  GPU 的板子上产生不必要的探测。

修改位置：
``arch/arm64/boot/dts/sprd/sc9832e.dtsi``

3. sp9832e-1h10 板级设备树（``sp9832e-1h10.dts``）
----------------------------------------------------

在 sp9832e-1h10 参考板的 DTS 中，覆写 ``&gpu`` 节点将其启用：

.. code-block:: devicetree

   &gpu {
       status = "okay";
   };

修改位置：
``arch/arm64/boot/dts/sprd/sp9832e-1h10.dts``

4. Panfrost 驱动（``panfrost_drv.c``）
--------------------------------------

在 Panfrost DRM 驱动的 OF 设备匹配表中，新增 ``sprd,sc9832e-mali`` 条目，
使用标准的 ``default_data`` 配置（无需特殊电源域管理）：

.. code-block:: c

   { .compatible = "sprd,sc9832e-mali", .data = &default_data, },

修改位置：
``drivers/gpu/drm/panfrost/panfrost_drv.c``

5. 内核配置（``sc9832e_defconfig``）
-------------------------------------

在 SC9832E 的默认内核配置文件中开启 Panfrost 驱动：

.. code-block:: kconfig

   CONFIG_DRM_PANFROST=y

修改位置：
``arch/arm64/configs/sc9832e_defconfig``

时钟体系说明
============

SC9832E 的 GPU 时钟由 sc9832e-clk.c 驱动管理，相关时钟定义如下：

``gpu_eb``（门控时钟）
  - 时钟名：``gpu-eb``
  - 父时钟：``clk_aon_apb``
  - 控制寄存器：偏移 ``0x0``，第 27 位
  - 所属控制器：``sprd,sc9832e-aonapb-gate``（设备树节点 ``aonapb_gate``）
  - 时钟 ID：``CLK_GPU_EB``（值为 26）

``nic_gpu_clk``（复合时钟：多路选择 + 分频）
  - 时钟名：``nic-gpu-clk``
  - 控制寄存器：偏移 ``0x2d8``
  - 父时钟可选：``twpll_256m``、``twpll_307m2``、``twpll_384m``、
    ``twpll_512m``、``gpll``
  - 所属控制器：``sprd,sc9832e-aon-clk``（设备树节点 ``aon_clk``）
  - 时钟 ID：``CLK_NIC_GPU``（值为 26）

Panfrost 驱动在初始化时会通过 ``devm_clk_get()`` 获取名为 ``core`` 的必选
时钟，以及名为 ``bus`` 的可选时钟，对应上述两个时钟。

如何在自定义板上启用 GPU
========================

若您在基于 SC9832E 的自定义板上使用本驱动，只需在您的板级 DTS 文件中覆写
``&gpu`` 节点：

.. code-block:: devicetree

   &gpu {
       status = "okay";
   };

如需电压调节器支持（DVFS），可在同一覆写节点中增加：

.. code-block:: devicetree

   &gpu {
       status = "okay";
       mali-supply = <&vddgpu>;  /* 指向您板上的 GPU 电源轨 */
   };

相关文件列表
============

.. list-table::
   :header-rows: 1
   :widths: 60 40

   * - 文件路径
     - 说明
   * - ``arch/arm64/boot/dts/sprd/sc9832e.dtsi``
     - SC9832E SoC 公共设备树（含 GPU 节点）
   * - ``arch/arm64/boot/dts/sprd/sp9832e-1h10.dts``
     - sp9832e-1h10 参考板设备树（启用 GPU）
   * - ``arch/arm64/configs/sc9832e_defconfig``
     - SC9832E 默认内核配置（启用 Panfrost）
   * - ``drivers/clk/sprd/sc9832e-clk.c``
     - SC9832E 时钟驱动（含 gpu_eb、nic_gpu_clk 定义）
   * - ``include/dt-bindings/clock/sprd,sc9832e-clk.h``
     - SC9832E 时钟 ID 头文件（CLK_GPU_EB、CLK_NIC_GPU）
   * - ``drivers/gpu/drm/panfrost/panfrost_drv.c``
     - Panfrost DRM 驱动（已加入 sc9832e-mali 匹配项）
   * - ``Documentation/devicetree/bindings/gpu/arm,mali-midgard.yaml``
     - Mali Midgard 设备树绑定（已加入 sprd,sc9832e-mali）
