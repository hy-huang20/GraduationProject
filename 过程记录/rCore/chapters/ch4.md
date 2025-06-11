# ch4

## 编程实验

难度有些大

### 概述

```rust
// os/src/task/task.rs

pub struct TaskControlBlock {
    // ...
    /// Application address space
    pub memory_set: MemorySet,
    // ...
}
```

```rust
// os/src/mm/memory_set.rs

pub struct MemorySet {
    page_table: PageTable,
    areas: Vec<MapArea>,
}

pub struct MapArea {
    vpn_range: VPNRange,
    data_frames: BTreeMap<VirtPageNum, FrameTracker>,
    map_type: MapType,
    map_perm: MapPermission,
}
```

```rust
// os/src/mm/page_table.rs

/// page table structure
pub struct PageTable {
    root_ppn: PhysPageNum,
    frames: Vec<FrameTracker>,
}
```

简而言之就是每个 task 对应一个 tcb，每个 tcb 中有个 memory_set 属性表示地址空间，每个 memory_set 中有当前 task 的页表和一些 MapArea，每个 MapArea 是一段虚存映射。PageTable 里面的 frames 存储了页表所有节点对应的物理帧，这些帧用来存 pte；而 MapArea 里面的 data_frames 存储了页表三级节点中的最后一级（叶节点）指向的物理帧，这些帧用来存数据。

### 读写用户态数据

其实最需要注意的地方就是所有 syscall 函数形参中的地址都变成虚拟地址了，而且在这之后的每一章实验中都需要记住这一点。

由于此时用户态 task 的地址全是虚拟地址，所以不能直接用上一章的方法读写用户态数据了。由于上一章的一些 syscall 接口涉及到从内核对用户态 task 数据的访问，因此**需要重新实现**。

具体是需要将用户态 task 的虚拟地址，利用该 task 的页表转换为物理地址。如果要实现从 os 访问修改用户态的数据，需要用好 ``translated_byte_buffer`` 函数。比如下面就是为了方便内核向用户态 task 拷贝数据而实现的函数：

```rust
// os/src/syscall/process.rs

/// 为了方便内核向用户态进程拷贝数据而实现的函数
pub fn os_data_copy_to_user(os_ptr: *const u8, user_ptr: *const u8, data_len: usize) {
    let token = current_user_token();
    let user_buffers = translated_byte_buffer(token, user_ptr, data_len);
    let os_data_byte_arr: &[u8] = unsafe { 
        core::slice::from_raw_parts(os_ptr, data_len) 
    };
    let mut byte_idx: usize = 0;
    for buffer in user_buffers {
        buffer.copy_from_slice(&os_data_byte_arr[byte_idx..byte_idx+buffer.len()]);
        byte_idx += buffer.len();
    }
}
```

``translated_byte_buffer`` 已经由实验框架提供：

```rust
// os/src/mm/page_table.rs

/// Translate&Copy a ptr[u8] array with LENGTH len to a mutable u8 Vec through page table
pub fn translated_byte_buffer(token: usize, ptr: *const u8, len: usize) -> Vec<&'static mut [u8]> {
    let page_table = PageTable::from_token(token);
    let mut start = ptr as usize;
    let end = start + len;
    let mut v = Vec::new();
    while start < end {
        let start_va = VirtAddr::from(start);
        let mut vpn = start_va.floor();
        let ppn = page_table.translate(vpn).unwrap().ppn();
        vpn.step();
        let mut end_va: VirtAddr = vpn.into();
        end_va = end_va.min(VirtAddr::from(end));
        if end_va.page_offset() == 0 {
            v.push(&mut ppn.get_bytes_array()[start_va.page_offset()..]);
        } else {
            v.push(&mut ppn.get_bytes_array()[start_va.page_offset()..end_va.page_offset()]);
        }
        start = end_va.into();
    }
    v
}
```

``translated_byte_buffer`` 会以向量的形式返回一组可以在内核空间中直接访问的字节数组切片。为什么以向量形式呢，因为访问可能会跨多个页，而向量中的每个元素（每个 slice）会包含至多一个页的数据，这是因为对数据的访问并不一定能够按页对齐。由于返回的是 slice 的可变引用，因此可以直接读写，十分方便。

### 虚存映射的建立与取消

```rust
// os/src/task/mod.rs

pub fn map_for_current_task(start_vpn: VirtPageNum, num_pages: usize, map_perm: MapPermission) -> isize {
    let task_id = get_current_task_id();
    let memory_set = &mut TASK_MANAGER.inner.exclusive_access().tasks[task_id].memory_set;
    let mut end_vpn = start_vpn;
    for _ in 0..num_pages {
        if let Some(pte) = memory_set.translate(end_vpn) {
            if pte.is_valid() { // vpn 已经被映射到了已经存在的物理页
                return -1;
            }
        }
        end_vpn.step();
    }
    let start_va = VirtAddr::from(start_vpn);
    let end_va = VirtAddr::from(end_vpn);
    // 一块新的 MapArea，会自动 map PageTable
    memory_set.insert_framed_area(start_va, end_va, map_perm);
    return 0;
}

pub fn unmap_for_current_task(start_vpn: VirtPageNum, num_pages: usize) -> isize {
    let task_id = get_current_task_id();
    let memory_set = &mut TASK_MANAGER.inner.exclusive_access().tasks[task_id].memory_set;
    let mut end_vpn = start_vpn;
    for _ in 0..num_pages {
        if let Some(pte) = memory_set.translate(end_vpn) {
            if !pte.is_valid() { // 不能解除不存在的映射
                return -1;
            }
            memory_set.unmap_from_page_table(end_vpn);
            end_vpn.step();
        } else { // 不能解除不存在的映射
            return -1;
        }
    }
    return 0;
}
```

实现 map/unmap 功能可以只需要使用 MemorySet 提供的接口来完成，具体来说上述代码使用到以下三个 pub 接口：

```rust
impl MemorySet {
    // ...

    /// 这个是自己添加的接口
    pub fn unmap_from_page_table(&mut self, vpn: VirtPageNum) {
        self.page_table.unmap(vpn);
    }

    /// Assume that no conflicts.
    pub fn insert_framed_area(
        &mut self,
        start_va: VirtAddr,
        end_va: VirtAddr,
        permission: MapPermission,
    ) {
        self.push(
            MapArea::new(start_va, end_va, MapType::Framed, permission),
            None,
        );
    }
    fn push(&mut self, mut map_area: MapArea, data: Option<&[u8]>) {
        map_area.map(&mut self.page_table);
        if let Some(data) = data {
            map_area.copy_data(&mut self.page_table, data);
        }
        self.areas.push(map_area);
    }

    /// Translate a virtual page number to a page table entry
    pub fn translate(&self, vpn: VirtPageNum) -> Option<PageTableEntry> {
        self.page_table.translate(vpn)
    }

    // ...
}
```

- ``insert_framed_area``

    每次 sys_mmap 的时候都会调用这个函数，因此每次通过 mmap 申请一块内存都会 push 一个新的 **MapArea** 进入 memory_set.areas。

    考虑 push 函数。当然，这里可以不用管 map_area.copy_data 这个调用，因为这里 push 函数形参 data 是 None。此外在 push 函数中调用了 map_area.map：

    ```rust
    
    impl MapArea {
        // ...

        pub fn map(&mut self, page_table: &mut PageTable) {
            for vpn in self.vpn_range {
                self.map_one(page_table, vpn);
            }
        }

        pub fn map_one(&mut self, page_table: &mut PageTable, vpn: VirtPageNum) {
            let ppn: PhysPageNum;
            match self.map_type {
                MapType::Identical => {
                    ppn = PhysPageNum(vpn.0);
                }
                MapType::Framed => { // 这里会走这个逻辑
                    let frame = frame_alloc().unwrap();
                    ppn = frame.ppn;
                    self.data_frames.insert(vpn, frame);
                }
            }
            let pte_flags = PTEFlags::from_bits(self.map_perm.bits).unwrap();
            page_table.map(vpn, ppn, pte_flags);
        }

        // ...
    }

    ```

    对于申请的内存的每个虚拟页面，都会通过 frame_alloc() 申请一个物理页帧 frame，首先将 frame 加到 map_area.data_frames 中（RAII），其次在 memory_set.page_table 中建立该虚拟页和 frame.ppn 物理页之间的联系，这样下次 memory_set.page_table.translate 就能够查到了。

- ``translate``

    查 task 的页表通过调用该 task 对应的 tcb 的 memory_set 属性的 translate 函数，该函数实际上会调用 memory_set.page_table 的 translate 函数，这是 PageTable 提供的不经过 MMU 而是手动查页表的方法，以方便实现：

    ```rust
    // os/src/mm/page_table.rs

    impl PageTable {
        /// Temporarily used to get arguments from user space.
        pub fn from_token(satp: usize) -> Self {
            Self {
                root_ppn: PhysPageNum::from(satp & ((1usize << 44) -    1)),
                frames: Vec::new(),
            }
        }
        fn find_pte(&self, vpn: VirtPageNum) -> Option<&    PageTableEntry> {
            let idxs = vpn.indexes();
            let mut ppn = self.root_ppn;
            let mut result: Option<&PageTableEntry> = None;
            for i in 0..3 {
                let pte = &ppn.get_pte_array()[idxs[i]];
                if i == 2 {
                    result = Some(pte);
                    break;
                }
                if !pte.is_valid() {
                    return None;
                }
                ppn = pte.ppn();
            }
            result
        }
        pub fn translate(&self, vpn: VirtPageNum) -> Option<PageTableEntry> {
            self.find_pte(vpn)
                .map(|pte| {pte.clone()})
        }
    }
    ```

    注意 PageTable::translate 中的这个 Option::map 函数，它接收一个``闭包``（函数）对 Option 内部的值进行转换，同时保持 Option 的上下文（即 Some 或 None 的状态）。

    于是 translate 返回的 pte 是以拷贝形式返回，不涉及所有权转移，也无法通过修改返回的 pte 影响到原 pte 的值。

- ``unmap_from_page_table``

    MemorySet 中原本**没有**这个接口，是我为了方便实现擅自加进去的。简单地调用了 memory_set.page_table 的 unmap 函数：

    ```rust
    // os/src/mm/page_table.rs

    /// Assume that it won't oom when creating/mapping.
    impl PageTable {
        // ...

        /// remove the map between virtual page number and physical page number
        #[allow(unused)]
        pub fn unmap(&mut self, vpn: VirtPageNum) {
            let pte = self.find_pte(vpn).unwrap();
            assert!(pte.is_valid(), "vpn {:?} is invalid before unmapping", vpn);
            *pte = PageTableEntry::empty();
        }

        // ...
    }
    ```

    这样下次再通过调用 memory_set.page_table 的 translate 就会返回一个为 PageTableEntry::empty 的 pte，自然是 !pte.is_valid() 的

其实很多接口已经设计地很好了可以直接用了，最好在看懂的前提下合理运用。当然，直接绕过这些**上层接口**直接调用 frame_alloc() 且自己设置 pte 的内容似乎并不影响测例的通过。但是如果直接调用 frame_alloc() 的话所分配的 frame 默认不会被记录到 MapArea 以及 PageTable 中。而将 frame 添加到 MapArea 或 PageTable 中的用意是利用 RAII 思想，将这些 frame 的生命周期绑定到逻辑段 MapArea 或页表 PageTable 下，以便逻辑段和页表被回收之后这些之前分配的物理页帧也会自动地同时被回收。
