内核与应用的地址空间
================================================

本节导读
------------------------------------------------------

前两节我们了解了 SV39 分页机制的原理和实现。本节将介绍如何为内核和应用程序创建独立的地址空间，这是实现内存隔离的关键。


两种地址空间设计
------------------------------------------------------

设计选择
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

有两种常见的地址空间设计方案：

**方案一：共享地址空间（传统设计）**

- 内核和应用共享同一个页表
- 内核映射在高地址，应用映射在低地址
- 通过 U 标志位区分内核页和用户页
- 优点：Trap 时无需切换页表，性能好
- 缺点：可能受 Meltdown 等侧信道攻击影响

**方案二：独立地址空间（本章采用）**

- 内核和每个应用各自拥有独立的页表
- Trap 时需要切换页表
- 优点：更安全，可防范侧信道攻击
- 缺点：切换开销略大

.. code-block:: text

    方案一（共享）：           方案二（独立）：

      +-------------+           内核地址空间    应用A地址空间
      |  内核空间   |          +-----------+   +-----------+
      +-------------+          |   内核    |   |   应用A   |
      |  应用空间   |          +-----------+   +-----------+
      +-------------+             satp_k         satp_a
          一个 satp


内核地址空间
------------------------------------------------------

内核地址空间的布局
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

内核地址空间使用 **恒等映射**，虚拟地址等于物理地址：

.. code-block:: text

    虚拟地址                                物理地址
    0x80200000  +----------------+  0x80200000
                |  .text (代码)   |
    0x80214000  +----------------+  0x80214000
                |  .rodata       |
    0x8021d000  +----------------+  0x8021d000
                |  .data/.bss    |
    0x811a4000  +----------------+  0x811a4000
                |  .boot (启动栈) |
    0x811aa000  +----------------+  0x811aa000
                |                |
                |  堆空间        |
                |                |
    0x81a00000  +----------------+  0x81a00000

    // 传送门（最高虚拟页）
    0x7FFFFFFFFF000  +----------+
                     |  Portal  |  --> 映射到 portal 物理地址
                     +----------+


创建内核地址空间
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在 ``ch4/src/main.rs`` 中：

.. code-block:: rust

    fn kernel_space(
        layout: linker::KernelLayout,
        memory: usize,
        portal: usize,
    ) -> AddressSpace<Sv39, Sv39Manager> {
        let mut space = AddressSpace::<Sv39, Sv39Manager>::new();
        
        // 1. 映射内核各个段（恒等映射）
        for region in layout.iter() {
            log::info!("{region}");  // 打印段信息
            use linker::KernelRegionTitle::*;
            let flags = match region.title {
                Text => "X_RV",      // 代码段：可执行、可读
                Rodata => "__RV",    // 只读数据：只读
                Data | Boot => "_WRV", // 数据段：可读写
            };
            let s = VAddr::<Sv39>::new(region.range.start);
            let e = VAddr::<Sv39>::new(region.range.end);
            space.map_extern(
                s.floor()..e.ceil(),
                PPN::new(s.floor().val()),  // 恒等映射：PPN = VPN
                VmFlags::build_from_str(flags),
            )
        }
        
        // 2. 映射堆空间（恒等映射）
        log::info!(
            "(heap) ---> {:#10x}..{:#10x}",
            layout.end(),
            layout.start() + memory
        );
        let s = VAddr::<Sv39>::new(layout.end());
        let e = VAddr::<Sv39>::new(layout.start() + memory);
        space.map_extern(
            s.floor()..e.ceil(),
            PPN::new(s.floor().val()),
            VmFlags::build_from_str("_WRV"),
        );
        
        // 3. 映射传送门（位于最高虚拟页）
        space.map_extern(
            PROTAL_TRANSIT..PROTAL_TRANSIT + 1,
            PPN::new(portal >> Sv39::PAGE_BITS),
            VmFlags::build_from_str("__G_XWRV"),  // G=全局，允许执行和读写
        );
        
        println!();
        
        // 4. 启用分页！
        unsafe { satp::set(satp::Mode::Sv39, 0, space.root_ppn().val()) };
        
        space
    }

.. note::

    ``satp::set`` 之后，所有的内存访问都会经过 MMU 转换。由于我们使用恒等映射，代码可以无缝继续执行。


应用地址空间
------------------------------------------------------

应用地址空间的布局
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

与内核不同，应用程序使用独立的虚拟地址布局：

.. code-block:: text

    应用虚拟地址空间：
                       +------------------+
    0x4000000000 (1G)  |     用户栈       |  ← sp 初始值
    0x3FFFFFFE000      +------------------+
                       |        ↓         |
                       |    （向下增长）   |
                       +------------------+
                       |                  |
                       |    （未映射）     |
                       |                  |
                       +------------------+
                       |    ELF 数据段    |
                       |    (.data/.bss)  |
                       +------------------+
                       |    ELF 代码段    |
                       |    (.text)       |
                       +------------------+
                       |                  |
    0x0                +------------------+


Process 结构体
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在 ``ch4/src/process.rs`` 中：

.. code-block:: rust

    /// 进程。
    pub struct Process {
        /// 跨地址空间的上下文
        pub context: ForeignContext,
        /// 进程的地址空间
        pub address_space: AddressSpace<Sv39, Sv39Manager>,
    }

注意与第三章 ``TaskControlBlock`` 的区别：

.. code-block:: rust

    // ch3: TaskControlBlock
    pub struct TaskControlBlock {
        ctx: LocalContext,       // 同地址空间上下文
        pub finish: bool,
        stack: [usize; 256],     // 固定大小栈
    }

    // ch4: Process
    pub struct Process {
        pub context: ForeignContext,  // 跨地址空间上下文
        pub address_space: AddressSpace<Sv39, Sv39Manager>,  // 独立地址空间
    }


从 ELF 文件创建进程
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ELF（Executable and Linkable Format）是一种标准的可执行文件格式。解析 ELF 文件可以获取：

- 程序入口地址
- 各段的虚拟地址、大小、权限
- 段数据

.. code-block:: rust

    // ch4/src/process.rs

    impl Process {
        pub fn new(elf: ElfFile) -> Option<Self> {
            // 1. 获取入口地址
            let entry = match elf.header.pt2 {
                HeaderPt2::Header64(pt2)
                    if pt2.type_.as_type() == header::Type::Executable
                        && pt2.machine.as_machine() == Machine::RISC_V =>
                {
                    pt2.entry_point as usize
                }
                _ => None?,  // 不是有效的 RISC-V 可执行文件
            };

            const PAGE_SIZE: usize = 1 << Sv39::PAGE_BITS;  // 4096
            const PAGE_MASK: usize = PAGE_SIZE - 1;         // 0xFFF

            // 2. 创建地址空间
            let mut address_space = AddressSpace::new();
            
            // 3. 遍历 ELF 的 Program Header，映射各个段
            for program in elf.program_iter() {
                // 只处理 LOAD 类型的段
                if !matches!(program.get_type(), Ok(program::Type::Load)) {
                    continue;
                }

                let off_file = program.offset() as usize;       // 段在文件中的偏移
                let len_file = program.file_size() as usize;    // 段在文件中的大小
                let off_mem = program.virtual_addr() as usize;  // 段的虚拟地址
                let end_mem = off_mem + program.mem_size() as usize;  // 段结束地址
                
                // 确保对齐
                assert_eq!(off_file & PAGE_MASK, off_mem & PAGE_MASK);

                // 4. 根据 ELF 标志设置页表标志
                let mut flags: [u8; 5] = *b"U___V";  // 用户态可访问
                if program.flags().is_execute() {
                    flags[1] = b'X';  // 可执行
                }
                if program.flags().is_write() {
                    flags[2] = b'W';  // 可写
                }
                if program.flags().is_read() {
                    flags[3] = b'R';  // 可读
                }
                
                // 5. 分配物理页，拷贝数据，建立映射
                address_space.map(
                    VAddr::new(off_mem).floor()..VAddr::new(end_mem).ceil(),
                    &elf.input[off_file..][..len_file],  // ELF 文件中的数据
                    off_mem & PAGE_MASK,                  // 页内偏移
                    VmFlags::from_str(unsafe { core::str::from_utf8_unchecked(&flags) }).unwrap(),
                );
            }
            
            // 6. 分配用户栈
            let stack = unsafe {
                alloc_zeroed(Layout::from_size_align_unchecked(
                    2 << Sv39::PAGE_BITS,   // 2 页 = 8KB
                    1 << Sv39::PAGE_BITS,
                ))
            };
            address_space.map_extern(
                VPN::new((1 << 26) - 2)..VPN::new(1 << 26),  // 虚拟地址接近 1GB
                PPN::new(stack as usize >> Sv39::PAGE_BITS),
                VmFlags::build_from_str("U_WRV"),  // 用户态、可读写
            );

            log::info!("process entry = {:#x}", entry);

            // 7. 创建上下文
            let mut context = LocalContext::user(entry);
            // 计算 satp 值：MODE=8（SV39），PPN=根页表物理页号
            let satp = (8 << 60) | address_space.root_ppn().val();
            // 设置栈指针为栈顶
            *context.sp_mut() = 1 << 38;  // 0x4000000000
            
            Some(Self {
                context: ForeignContext { context, satp },
                address_space,
            })
        }
    }


ELF 加载过程图解
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: text

    ELF 文件：                        应用地址空间：
    +------------------+             +------------------+
    | ELF Header       |             |                  |
    | (入口地址等)      |             |     用户栈       |
    +------------------+             +------------------+ 1GB
    | Program Headers  |             |                  |
    | (段描述)         |             |                  |
    +------------------+             +------------------+
    | .text 段数据     | ─────────→  |    .text         |
    +------------------+  map()      +------------------+
    | .data 段数据     | ─────────→  |    .data         |
    +------------------+             +------------------+
                                     |                  |
                                     +------------------+ 0


传送门的映射
------------------------------------------------------

为什么需要传送门？
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当应用发生 Trap 时，CPU 会跳转到 ``stvec`` 指向的地址。但此时我们还在应用的地址空间中，需要先切换到内核地址空间，才能执行内核代码。

问题：如果 ``stvec`` 指向的代码在内核地址空间中，但 CPU 当前使用的是应用的页表，这段代码就无法被执行！

解决方案：将一段 "传送门" 代码同时映射到内核和应用的地址空间中，在同一个虚拟地址上：

.. code-block:: text

    内核地址空间：                   应用地址空间：
    +------------------+            +------------------+
    |                  |            |                  |
    +------------------+            +------------------+
    |                  |            |                  |
    +------------------+            +------------------+
    |     Portal       |            |     Portal       |
    +------------------+ 0x7FFF...  +------------------+ 0x7FFF...

    Portal 映射到相同的物理页！


传送门的建立
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在 ``ch4/src/main.rs`` 中：

.. code-block:: rust

    extern "C" fn rust_main() -> ! {
        // ...
        
        // 1. 分配传送门的物理页
        let portal_size = MultislotPortal::calculate_size(1);
        let portal_layout = Layout::from_size_align(portal_size, 1 << Sv39::PAGE_BITS).unwrap();
        let portal_ptr = unsafe { alloc(portal_layout) };
        
        // 2. 在内核地址空间中映射传送门
        let mut ks = kernel_space(layout, MEMORY, portal_ptr as _);
        let portal_idx = PROTAL_TRANSIT.index_in(Sv39::MAX_LEVEL);
        
        // 3. 加载应用时，将传送门映射到应用地址空间
        for (i, elf) in linker::AppMeta::locate().iter().enumerate() {
            if let Some(process) = Process::new(ElfFile::new(elf).unwrap()) {
                // 共享传送门的页表项！
                process.address_space.root()[portal_idx] = ks.root()[portal_idx];
                unsafe { PROCESSES.push(process) };
            }
        }
    }


地址空间的隔离
------------------------------------------------------

隔离的实现
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

地址空间隔离是这样实现的：

1. **物理隔离**：每个进程的 ELF 数据被拷贝到不同的物理页
2. **映射隔离**：每个进程有自己的页表，只映射自己的物理页
3. **权限控制**：页表项的 U 标志位控制用户态能否访问

.. code-block:: text

    进程 A 的页表：                   进程 B 的页表：
    VPN 0x10 → PPN 0x80300          VPN 0x10 → PPN 0x80400
    VPN 0x11 → PPN 0x80301          VPN 0x11 → PPN 0x80401
    ...                              ...

    虽然两个进程都使用虚拟地址 0x10000，
    但它们映射到不同的物理页！


内核数据的保护
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

应用的页表中不映射内核数据的物理页，因此应用无法访问内核数据：

.. code-block:: text

    应用尝试访问 0x80200000（内核代码地址）：
    1. MMU 在应用的页表中查找 VPN
    2. 找不到对应的页表项
    3. 触发 Page Fault 异常
    4. 内核终止这个应用


关键概念总结
------------------------------------------------------

.. list-table::
   :header-rows: 1

   * - 概念
     - 说明
   * - 恒等映射
     - 虚拟地址 = 物理地址，用于内核地址空间
   * - ELF 解析
     - 从 ELF 文件获取程序段信息
   * - Process
     - 包含 ForeignContext 和 AddressSpace
   * - ForeignContext
     - 包含 LocalContext 和 satp
   * - 传送门
     - 同时映射到内核和应用地址空间的代码
   * - 地址空间隔离
     - 每个进程只能访问自己的物理页

下一节我们将介绍如何基于地址空间实现分时多任务。
