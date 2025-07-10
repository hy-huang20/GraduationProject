# rust 异步驱动设计与实现

总体的计划安排 [实时更新中](#20250611)

## 20250709

### 进度

- 进行中：看 embassy 源码 [过程记录](https://github.com/hy-huang20/rust-os-learning/tree/main/%E8%BF%87%E7%A8%8B%E8%AE%B0%E5%BD%95/rust/rust%E5%BC%82%E6%AD%A5/Embassy)
    - 目前涉及到的 package:
        - embassy-time
        - embassy-time-driver
        - embassy_time_queue_utils
        - embassy-executor
        - embassy-executor-macros

### 后续

- 把 embassy-executor 执行过程理清楚，并完成源码阅读过程记录
    - embassy-time
    - embassy-executor
- 参考一下训练营中的 embassy 记录

## 20250701

### 进度

- 看 [Embassy Book](https://embassy.dev/book/)，[过程记录](https://github.com/hy-huang20/rust-os-learning/blob/main/%E8%BF%87%E7%A8%8B%E8%AE%B0%E5%BD%95/rust/rust%E5%BC%82%E6%AD%A5/Embassy/readme.md)
    - Introduction 节
    - System Description 节
        - Embassy Executor 节
        - Time-keeping 节
- 跑通了 Embassy Book 中的 [2 个 example](https://github.com/hy-huang20/rust-learning/commit/955c5bb61689788dcc2a45fe7d03b5bd940f7ea7)
- 问题：林晨纪要 0416 中几个链接似乎失效了
    - [baremental 仓库](https://github.com/zflcs/baremental/)失效
    - [仍然有效的链接](https://github.com/ATS-INTC/ats-intc/blob/30e4ea6a47de5ca8ec757764d544f3c8e58d9e14/src/waker.rs?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NTEzMzEwNDgsImZpbGVHVUlEIjoiWEtxNDI1eGIxbnQ0V3pBTiIsImlhdCI6MTc1MTMzMDc0OCwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoiYWxsOmFsbDoiLCJ1c2VySWQiOjMwNTEwODA2fQ.lwgSV9-tymBeP2w-owXr-mwaahMgADLQSUjYye6lQqA)
    - 解决：可以看[其它例子](https://github.com/ATS-INTC/ats-intc/blob/30e4ea6a47de5ca8ec757764d544f3c8e58d9e14/src/lib.rs)

### 后续

- 看 Timer 源码
- 异步串口例子（还可以参考[训练营的代码：2，5](https://shimo.im/docs/KlkKvREgExUZM2qd)）在 qemu 中跑起来
- 继续看 Embassy Book
- 学习[在 rCore 中引入异步运行时](https://github.com/lighkLife/new-blog/issues/1?accessToken=eyJhbGciOiJIUzI1NiIsImtpZCI6ImRlZmF1bHQiLCJ0eXAiOiJKV1QifQ.eyJleHAiOjE3NTEzMzEwNDgsImZpbGVHVUlEIjoiWEtxNDI1eGIxbnQ0V3pBTiIsImlhdCI6MTc1MTMzMDc0OCwiaXNzIjoidXBsb2FkZXJfYWNjZXNzX3Jlc291cmNlIiwicGFhIjoiYWxsOmFsbDoiLCJ1c2VySWQiOjMwNTEwODA2fQ.lwgSV9-tymBeP2w-owXr-mwaahMgADLQSUjYye6lQqA)

## 20250625

### 进度

- 完成 Write an OS in Rust/async-await 的阅读 [总结](https://github.com/hy-huang20/rust-os-learning/blob/main/%E8%BF%87%E7%A8%8B%E8%AE%B0%E5%BD%95/rust/rust%E5%BC%82%E6%AD%A5/async-await/readme.md)
- 代码
    - 参照上述文档实现了一个[不带 Waker 的 Executor](https://github.com/hy-huang20/rust-learning/commit/9a56e8c0d5e5e0022983daca3dd3390a859076a4)
    - 基于一个[已有的异步爬虫](https://gitee.com/taoqi-cat/asyn/tree/master/spider)进行了简单改进

### 后续

- 学习 [embassy](https://github.com/embassy-rs/embassy/tree/main/embassy-executor)

## 20250618

### 进度

- 可选：[rCore 文档](https://github.com/hy-huang20/rust-os-learning/blob/main/%E8%BF%87%E7%A8%8B%E8%AE%B0%E5%BD%95/rCore/readme.md)
    - ch4, ch5 添加详细内容
- 学习 Write an OS in Rust [async-await](https://os.phil-opp.com/async-await/)
    - [文档总结](https://github.com/hy-huang20/rust-os-learning/blob/main/%E8%BF%87%E7%A8%8B%E8%AE%B0%E5%BD%95/rust/rust%E5%BC%82%E6%AD%A5/async-await/readme.md) 进行中

### 后续

- 可选：rCore 文档
    - ch6 比较简略
- 学完 async-await 并总结文档
- 用 tokio 库写一个异步的爬虫
    - 训练营中已有现成代码，可问老师要
- 有时间的话开始看 [embassy](https://github.com/embassy-rs/embassy/tree/main/embassy-executor)
    - timer：内核里写异步

## 20250611

### 进度

- 20250610 已和老师约时间演示实验完成情况
- 将写 rCore 的过程进行总结，已经形成[初版文档](https://github.com/hy-huang20/rust-os-learning/blob/main/%E8%BF%87%E7%A8%8B%E8%AE%B0%E5%BD%95/rCore/readme.md)

### 后续

- 毕设后续计划，明确任务/步骤以及前后任务之间的依赖关系
    - 复现：QEMU 串口驱动
        - 学习 rust 异步
            - Write an OS in Rust
                - [async-await](https://os.phil-opp.com/async-await/)  
            - [Embassy](https://github.com/embassy-rs/embassy/tree/main/embassy-executor)
                - 具体的细节需要看林晨的文档中的 20240416 的纪要中的描述
                - [embassy 记录](https://github.com/BITcyman/Rust-os-learning/blob/main/embassy/embassy.md)
                - [embassy into rCore 记录](https://github.com/BITcyman/Rust-os-learning/blob/main/embassy/embassy-into-rcore.md)
        - rCore-N 异步串口驱动 
            - 已经可以跑通 [记录](https://github.com/hy-huang20/rust-os-learning/blob/main/%E8%BF%87%E7%A8%8B%E8%AE%B0%E5%BD%95/%E5%A4%8D%E7%8E%B0%E8%BF%87%E7%A8%8B/rCore-N%5BQEMU%5D/readme.md)
            - 原理学习 [参考](https://github.com/BITcyman/Rust-os-learning/blob/main/rCore-N.md)
        - 学习已有的[异步串口驱动 crate](https://github.com/BITcyman/async-uart-driver/commit/3d1265d17e6b2d6e1ce8df351f6e6d19d04136ce)
        - QEMU 上跑通异步串口驱动在 AlienOS 中的使用
    - 实现：串口驱动上板子
        - 学习星光二板子上已有的 [PAC](https://codeberg.org/weathered-steel/jh71xx-pac) 和 [HAL](https://codeberg.org/weathered-steel/jh71xx-hal)
            - [参考](https://github.com/BITcyman/Rust-os-learning/blob/main/Vision_Five2.md)
        - 已有的工作
            - QEMU 上，异步串口驱动在 AlienOS 中使用 [最新记录](https://github.com/BITcyman/Rust-os-learning/blob/main/driver/runtime.md)
                >将多创建的基址在 0x10005000 上的串口绑定到一个终端上，目前能够看到从串口中输出的 ABC 字符串，但在终端中向串口中输入字符时，唤醒 串口读协程 出错。目前在找bug。又遇到多个bug。
            - **未实现**板子上的异步串口驱动
    - 实现：块设备驱动
        - QEMU，开发板
        - 了解不足，还需学习调研
            - 从老师那里了解到已经有能跑的实现了，只需要将已有的改成异步的
    - 最终目标
        - 星光二开发板上，串口/块设备异步驱动，在 AlienOS 中使用

## 20250604

### 进度

完成了 [rCore-2025S 实验](https://github.com/hy-huang20/rCore-2025S)

- 完成了 rCore-2025S 指导书 ch3-ch8 的编程练习，可以通过测例
- 偶尔会出现 rustup toolchain 的报错：st_name (xxx) is past the end of string table of size xxxx
- 上述报错和代码实现无关

### 后续

- 复盘，将写 rCore 的过程总结成文档
- 约向老师检查 rCore
- Rust 异步驱动设计与实现
    - 复现：QEMU 串口驱动（6月）
        - 学习并复现[林晨学长的工作](https://github.com/BITcyman/Rust-os-learning)
    - 实现：开发板串口驱动
    - 实现：QEMU 块设备驱动
    - 实现：开发板块设备驱动