# Rust 异步串口驱动模块设计与实现

## 背景和意义

- 外设
    - 独立于上层 OS
    - 用户对外设的操作：I/O
- 性能
    - 异步：I/O 密集型场景
    - 异步 I/O 方式：
        - WIN 32 Multi-thread Async API
        - Java Message Queue
        - C/C++ AC
        - Rust Future

## 概况

- 已有串口驱动实现：考虑内存安全、异步逻辑支持
    - C, Rust, C++, GO, 汇编 ...

- 趋势：Rust 编写 OS 组件

## 选题

### Rust 异步串口驱动模块设计与实现

为以下：

- 外设
    - 星光 2 开发板支持的外设
- Flash 闪存块设备
    - SD 卡

编写基于 Rust 的异步逻辑的驱动模块

## 目标和内容

- 目标：
    - Rust 异步支持
    - embassy repo 中异步运行实现
    - 环境：QEMU, 星光 2 开发板
    - 目标：跨操作系统/文件系统

- 内容：用 Rust 实现开发板上各种外设的异步驱动
    - 复现：在 QEMU 环境对 Alien OS 完成异步串口驱动模块的适配
    - 基础：在开发板上对 Alien OS 完成异步串口驱动模块的适配
    - 进阶：实现 SD 卡闪存块设备异步驱动，支持在开发板上运行文件系统并通过相应测例集

## 计划和进度安排

- 2024 年 12 月 - 2025 年 1 月上旬：调研确定选题
- 2025 年 1 月上旬：开题答辩
- 2025 年 1 月上旬 - 2025 年 2 月：继续学习 Rust、异步串口驱动
- 2025 年 2 月中旬：在 QEMU 上复现
- 2025 年 2 月中旬 - 3 月上旬：在开发板上成功跑异步串口驱动
- 2025 年 3 月上旬 - 4 月上旬：块设备异步驱动编写，中期答辩
- 2025 年 4 月上旬 - 5 月：块设备异步驱动，模拟与上板
- 2025 年 5 月 - 6 月：写论文，准备答辩
- 2025 年 6 月：进行毕设答辩

## 参考文献与资料

- 方兴,秦琦,刘维国. 多线程异步I/O模型. 舰船电子对抗,2005(04)
- 段楠. 异步非阻塞网络通讯技术研究. 现代计算机,2019(17)
- rCore-OS. ArceOS. https://github.com/rcore-os/arceos
- 林晨：https://github.com/BITcyman/Rust-os-learning
- Harris T. Special Topic: AC – Composable Asynchronous IO For Native Languages
- Zhu L , Huang L , Fu P ,et al. The upgrade to the EAST poloidal field power supply monitoring system
- Kwon G , Lee W , Lee T ,et al. Development of a real-time data archive system for a KSTAR real-time network
- Jan Axelson. Serial Port Complete: COM Ports, USB Virtual COM Ports, and Ports for Embedded Systems