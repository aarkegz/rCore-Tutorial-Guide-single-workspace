# 实现 SV39 多级页表机制（下）

## 本节导读

上一节我们了解了 SV39 的基本概念，本节将深入代码，看看如何：
1. 分配和管理物理页帧
2. 建立虚拟地址到物理地址的映射
3. 实现地址翻译

## 内核堆分配器

### 为什么需要堆分配？

在第三章中，我们使用固定大小的数组来存储任务：

```rust
// ch3: 固定大小的任务控制块数组
let mut tcbs = [TaskControlBlock::ZERO; APP_CAPACITY];
```

这种方式的问题是：
- 大小固定，无法动态调整
- 无法使用 `Vec`、`Box` 等动态数据结构
- 页表需要动态分配内存

在第四章中，我们引入了内核堆分配器 `kernel-alloc`：

```rust
// ch4: 可以使用 Vec 了！
static mut PROCESSES: Vec<Process> = Vec::new();
```

### 初始化堆分配器

在 `ch4/src/main.rs` 中初始化堆：

```rust
extern "C" fn rust_main() -> ! {
    let layout = linker::KernelLayout::locate();
    // bss 段清零
    unsafe { layout.zero_bss() };
    // 初始化控制台
    // ...
    
    // 初始化内核堆
    kernel_alloc::init(layout.start() as _);
    unsafe {
        kernel_alloc::transfer(core::slice::from_raw_parts_mut(
            layout.end() as _,         // 堆的起始地址：内核镜像结束处
            MEMORY - layout.len(),     // 堆的大小：总内存 - 内核占用
        ))
    };
    // 现在可以使用 alloc 库了！
}
```

内存布局：

```
+------------------+ 0x80200000 (layout.start())
|   内核代码       |
|   (.text)        |
+------------------+
|   只读数据       |
|   (.rodata)      |
+------------------+
|   数据段         |
|   (.data/.bss)   |
+------------------+ layout.end()
|                  |
|   内核堆         |  <-- kernel_alloc 管理这块区域
|   (用于动态分配)  |
|                  |
+------------------+ 0x80200000 + MEMORY
```

## 物理页帧管理

### Sv39Manager 实现

在 `ch4/src/main.rs` 的 `impls` 模块中，我们实现了 `Sv39Manager` 来管理物理页：

```rust
// ch4/src/main.rs (impls 模块)

/// 页表管理器，封装了根页表指针
#[repr(transparent)]
pub struct Sv39Manager(NonNull<Pte<Sv39>>);

impl Sv39Manager {
    /// 标记页面是由我们分配的（使用 RSW 保留位）
    const OWNED: VmFlags<Sv39> = unsafe { VmFlags::from_raw(1 << 8) };

    /// 分配 count 个连续的物理页，并清零
    #[inline]
    fn page_alloc<T>(count: usize) -> *mut T {
        unsafe {
            alloc_zeroed(Layout::from_size_align_unchecked(
                count << Sv39::PAGE_BITS,   // 总字节数 = count * 4096
                1 << Sv39::PAGE_BITS,       // 按页对齐 = 4096
            ))
        }
        .cast()
    }
}
```

### 实现 PageManager trait

`PageManager` 是 `kernel-vm` 定义的 trait，需要实现以下方法：

```rust
impl PageManager<Sv39> for Sv39Manager {
    /// 创建新的根页表
    fn new_root() -> Self {
        Self(NonNull::new(Self::page_alloc(1)).unwrap())
    }

    /// 获取根页表的物理页号
    fn root_ppn(&self) -> PPN<Sv39> {
        PPN::new(self.0.as_ptr() as usize >> Sv39::PAGE_BITS)
    }

    /// 获取根页表的指针
    fn root_ptr(&self) -> NonNull<Pte<Sv39>> {
        self.0
    }

    /// 将物理页号转换为当前地址空间的指针
    /// 由于内核使用恒等映射，物理地址 == 虚拟地址
    fn p_to_v<T>(&self, ppn: PPN<Sv39>) -> NonNull<T> {
        unsafe { NonNull::new_unchecked(VPN::<Sv39>::new(ppn.val()).base().as_mut_ptr()) }
    }

    /// 将指针转换为物理页号
    fn v_to_p<T>(&self, ptr: NonNull<T>) -> PPN<Sv39> {
        PPN::new(VAddr::<Sv39>::new(ptr.as_ptr() as _).floor().val())
    }

    /// 检查页表项是否由我们分配
    fn check_owned(&self, pte: Pte<Sv39>) -> bool {
        pte.flags().contains(Self::OWNED)
    }

    /// 分配物理页
    fn allocate(&mut self, len: usize, flags: &mut VmFlags<Sv39>) -> NonNull<u8> {
        *flags |= Self::OWNED;  // 标记为我们分配的
        NonNull::new(Self::page_alloc(len)).unwrap()
    }

    // ... deallocate, drop_root 方法
}
```

### 恒等映射

注意 `p_to_v` 和 `v_to_p` 的实现：物理地址等于虚拟地址。这是因为内核地址空间使用**恒等映射**（Identical Mapping）。

恒等映射的含义是：对于物理地址 `0x80200000`，它在内核地址空间中的虚拟地址也是 `0x80200000`。

这样做的好处：
- 在启用分页前后，内核代码可以无缝运行
- 方便内核访问任意物理地址

## 建立地址映射

### map_extern：映射到已有的物理页

`map_extern` 用于将虚拟地址范围映射到指定的物理页：

```rust
// kernel-vm/src/space/mod.rs

pub fn map_extern(&mut self, range: Range<VPN<Meta>>, pbase: PPN<Meta>, flags: VmFlags<Meta>) {
    // 记录这个虚拟地址范围
    self.areas.push(range.start..range.end);
    let count = range.end.val() - range.start.val();
    let mut root = self.root();
    // 使用 Mapper 遍历并填充页表
    let mut mapper = Mapper::new(self, pbase..pbase + count, flags);
    root.walk_mut(Pos::new(range.start, 0), &mut mapper);
}
```

使用示例（内核地址空间）：

```rust
// 映射内核代码段（恒等映射）
// 虚拟地址 0x80200000 -> 物理地址 0x80200000
let s = VAddr::<Sv39>::new(0x80200000);
let e = VAddr::<Sv39>::new(0x80214000);
space.map_extern(
    s.floor()..e.ceil(),              // 虚拟页范围
    PPN::new(s.floor().val()),        // 物理页号 = 虚拟页号（恒等映射）
    VmFlags::build_from_str("X_RV"),  // 可执行、可读
);
```

### map：分配新页并映射

`map` 用于分配新的物理页，拷贝数据，并建立映射：

```rust
// kernel-vm/src/space/mod.rs

pub fn map(
    &mut self,
    range: Range<VPN<Meta>>,   // 虚拟页范围
    data: &[u8],               // 要拷贝的数据
    offset: usize,             // 数据在页内的偏移
    mut flags: VmFlags<Meta>,  // 页表标志
) {
    let count = range.end.val() - range.start.val();
    let size = count << Meta::PAGE_BITS;
    
    // 分配物理页
    let page = self.page_manager.allocate(count, &mut flags);
    
    // 拷贝数据
    unsafe {
        use core::slice::from_raw_parts_mut as slice;
        let mut ptr = page.as_ptr();
        slice(ptr, offset).fill(0);           // 前面填零
        ptr = ptr.add(offset);
        slice(ptr, data.len()).copy_from_slice(data);  // 拷贝数据
        ptr = ptr.add(data.len());
        slice(ptr, page.as_ptr().add(size).offset_from(ptr) as _).fill(0);  // 后面填零
    }
    
    // 建立映射
    self.map_extern(range, self.page_manager.v_to_p(page), flags)
}
```

这个方法在加载应用程序时非常有用，可以将 ELF 文件的数据拷贝到新分配的页中。

## 地址翻译

### translate 方法

当内核需要访问应用程序的数据时（例如系统调用传入的指针），需要将应用的虚拟地址翻译为内核可以访问的地址：

```rust
// kernel-vm/src/space/mod.rs

pub fn translate<T>(&self, addr: VAddr<Meta>, flags: VmFlags<Meta>) -> Option<NonNull<T>> {
    let mut visitor = Visitor::new(self);
    // 遍历页表
    self.root().walk(Pos::new(addr.floor(), 0), &mut visitor);
    visitor
        .ans()
        // 检查权限
        .filter(|pte| pte.flags().contains(flags))
        // 转换为指针
        .map(|pte| unsafe {
            NonNull::new_unchecked(
                self.page_manager
                    .p_to_v::<u8>(pte.ppn())  // 物理页号 -> 虚拟地址（内核空间）
                    .as_ptr()
                    .add(addr.offset())        // 加上页内偏移
                    .cast(),
            )
        })
}
```

### 翻译过程图解

```
应用虚拟地址 0x12345678
         │
         ↓
    +---------+
    | 遍历应用 |
    | 的页表   |
    +---------+
         │
         ↓ 找到 PTE，提取物理页号
    +---------+
    | PPN =   |
    | 0x81234 |
    +---------+
         │
         ↓ p_to_v (恒等映射)
    +---------+
    | 内核虚拟 |
    | 0x81234 |
    | 000     |
    +---------+
         │
         ↓ + 页内偏移 0x678
    +---------+
    | 最终地址 |
    | 0x81234 |
    | 678     |
    +---------+
```

## 使用示例

### 在系统调用中翻译地址

```rust
// ch4/src/main.rs (impls 模块)

impl IO for SyscallContext {
    fn write(&self, caller: Caller, fd: usize, buf: usize, count: usize) -> isize {
        const READABLE: VmFlags<Sv39> = VmFlags::build_from_str("RV");
        
        // buf 是应用的虚拟地址，需要翻译
        if let Some(ptr) = unsafe { PROCESSES.get_mut(caller.entity) }
            .unwrap()
            .address_space
            .translate(VAddr::new(buf), READABLE)  // 翻译地址
        {
            // ptr 现在是内核可以访问的指针
            print!("{}", unsafe {
                core::str::from_utf8_unchecked(core::slice::from_raw_parts(
                    ptr.as_ptr(),
                    count,
                ))
            });
            count as _
        } else {
            log::error!("ptr not readable");
            -1
        }
    }
}
```

## 与第三章的对比

| 特性 | 第三章 | 第四章 |
|------|--------|--------|
| 物理页分配 | 固定栈数组 | 动态堆分配 |
| 页表管理 | 无 | `Sv39Manager` 实现 `PageManager` |
| 地址映射 | 无 | `map()` 和 `map_extern()` |
| 访问用户数据 | 直接使用指针 | 需要 `translate()` 翻译 |
| 数据结构 | 只能用固定数组 | 可以用 `Vec`、`Box` 等 |

## 关键概念总结

| 概念 | 说明 |
|------|------|
| kernel-alloc | 内核堆分配器，提供动态内存分配 |
| Sv39Manager | 管理 SV39 页表的结构体 |
| PageManager | 物理页管理的 trait |
| 恒等映射 | 内核地址空间中，虚拟地址 = 物理地址 |
| map_extern | 映射到已有的物理页 |
| map | 分配新物理页并映射 |
| translate | 将虚拟地址翻译为可访问的指针 |

下一节我们将看看如何为内核和应用程序创建地址空间。
