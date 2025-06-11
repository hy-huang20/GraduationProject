# ch6

## 编程实验

难度有些大。框架有些复杂，需要理解什么文件中什么代码做什么事，以及实现某个功能需要到哪里去修改，还有一些量的含义不要混淆了。

easy fs 架构：

![](./img/easy-fs-demo.png)

实验指导书是以自下而上的方式去介绍的，介绍了从块设备接口 BlockDevice 开始的各层负责的功能。

之前我混淆了 block_id 和 inode_id。其实二者之间存在以下可计算的对应关系：

```rust
fn cal_inode_id(block_id: usize, block_offset: usize, inode_area_start_block: usize) -> usize {
    let size_of_disk_inode = core::mem::size_of::<DiskInode>();
    let disk_inodes_per_block = BLOCK_SZ / size_of_disk_inode;
    let inode_id = (block_id - inode_area_start_block as usize) * disk_inodes_per_block + block_offset / size_of_disk_inode;
    inode_id
}
```