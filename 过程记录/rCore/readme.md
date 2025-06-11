# rCore 2025S 过程记录

本实验根据 [rCore-2025S 指导书](https://learningos.cn/rCore-Tutorial-Guide-2025S/) 以及[提供的初始实验仓库](https://github.com/LearningOS/rCore-Tutorial-Code-2025S)完成。

这里是[我的实现](https://github.com/hy-huang20/rCore-2025S)

## 目录

- [环境配置](#环境配置)
- [运行](#运行)
- [完成过程](#完成过程)

## 环境配置

不要使用 rCore-Tutorial-Book-v3 里面的实验仓库！尽量用最新的实验指导书和仓库！

- 之前基于 [rCore-Tutorial-Book-v3](https://rcore-os.cn/rCore-Tutorial-Book-v3/index.html) 写 rCore，使用链接中提供的[配置好的虚拟机](https://pan.baidu.com/share/init?surl=yQHtQIXQUbHCbyqSPtuqqQ&pwd=pcxf)运行，发现在环境配置方面存在诸多问题
    - 使用 VirtualBox NAT 模式，如需使用代理，需要在 host 机上设置 Allow LAN
    - 在虚拟机可以访问Google的情况下，make docker 仍然会遇到网络问题
    - 不使用 docker。在不做改动的情况下无法通过基础测例，提示 rustup nightly 版本问题
- 于是更换到最新的实验参考书 [rCore 2025S](https://learningos.cn/rCore-Tutorial-Guide-2025S/0setup-devel-env.html)
    - 使用本地 WSL2
    - make docker 仍然会出现问题，提示 pull 被拒绝。于是手动配置环境
    - 解压步骤中（qemu-7.0.0.tar.xz）会产生符号链接，像 NTFS 之类的文件系统支持符号链接，而如 exFAT 之类的无法支持，解压时会出现问题
    - 环境问题已经解决（将项目迁移到 NTFS 上），可以跑通 ch1~ch8 的基础测例
- [仓库链接](https://github.com/hy-huang20/rCore-2025S)

## 运行

实验指导书[第三章](https://learningos.cn/rCore-Tutorial-Guide-2025S/chapter3/5exercise.html#id5)给出了说明：

>默认情况下，makefile 仅编译基础测例 (BASE=1)，即无需修改框架即可正常运行的测例。 你需要在编译时指定 BASE=0 控制框架仅编译实验测例（在 os 目录执行 make run BASE=0）， 或指定 BASE=2 控制框架同时编译基础测例和实验测例。

如果使用 VS Code 开发，下载一个 rust-analyzer 插件会方便很多。如果在 wsl 上配置好 rust 环境并运行项目，而编程是在 windows 上完成，则为了 rust-analyzer 插件能够在编程时正常工作，需要在 windows 上也配置好 rust 环境。

## 完成过程

主要需要完成 5 个分支的编程实验：

- [ch3：多道程序与分时多任务](./chapters/ch3.md)
- [ch4：地址空间](./chapters/ch4.md)
- [ch5：进程及进程管理](./chapters/ch5.md)
- [ch6：文件系统与I/O重定向](./chapters/ch6.md)
- [ch8：并发](./chapters/ch8.md)

个人向难度评级为 ch8 > ch6 > ch4 > ch5 > ch3，其中第 4，6，8 都有一定难度。文档很详细，个人觉得难度主要是需要理解什么文件中的代码做什么事，我要实现/增加某个功能的话需要修改哪个文件中哪里的代码。

### 时间

|rCore-2025S|完成时间|任务数（旧+新）|
|---|---|---|
|ch3|20250503|1|
|ch4|20250511|2+2|
|ch5|20250512|3+2|
|ch6|20250519|5+3|
|ch8|20250603|1+1|
