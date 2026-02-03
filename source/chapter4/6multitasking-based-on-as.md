# 基于地址空间的分时多任务

## 本节导读

在前几节中，我们了解了如何创建内核和应用的地址空间。本节将介绍如何在独立地址空间之间进行切换，实现基于地址空间的分时多任务系统。

## 从第三章到第四章：上下文的演变

### 第三章：LocalContext

在第三章中，所有任务共享同一个地址空间，上下文切换只需要保存/恢复寄存器：

```rust
// ch3/src/task.rs

pub struct TaskControlBlock {
    ctx: LocalContext,    // 本地上下文
    pub finish: bool,
    stack: [usize; 256],
}

// 执行任务
pub unsafe fn execute(&mut self) {
    self.ctx.execute();  // 直接执行，不涉及地址空间切换
}
```

### 第四章：ForeignContext

在第四章中，每个进程有独立的地址空间，切换时还需要切换页表：

```rust
// kernel-context/src/foreign/mod.rs

/// 异界线程上下文。
/// 不在当前地址空间的线程。
pub struct ForeignContext {
    /// 目标地址空间上的线程上下文。
    pub context: LocalContext,
    /// 目标地址空间的 satp 值。
    pub satp: usize,
}
```

`ForeignContext` 比 `LocalContext` 多了一个 `satp` 字段，用于记录目标地址空间的页表根节点。

## 跨地址空间执行的挑战

### 问题分析

假设我们要从内核切换到应用执行，需要做以下事情：

1. 保存内核的寄存器
2. 切换页表（修改 `satp`）
3. 恢复应用的寄存器
4. `sret` 返回到应用态

问题来了：**切换页表后，内核代码还能执行吗？**

```
执行流程：
1. 内核代码在地址 0x80200100
2. 执行 csrw satp, app_satp  ← 切换到应用页表
3. 下一条指令在 0x80200104
4. 但应用页表可能没有映射 0x80200100！
   → CPU 找不到代码 → 异常！
```

### 解决方案：传送门

传送门（Portal）是一段特殊的代码，它被同时映射到内核和应用的地址空间的相同虚拟地址上：

```
地址空间切换时的执行流程：

内核代码 (0x80200xxx)
    │
    ↓ 跳转到传送门
传送门代码 (0x7FFFFFFFFF000)  ← 在两个地址空间都有映射！
    │ 切换 satp
    │ sret
    ↓
应用代码 (0x10xxx)
```

## 传送门的实现

### MultislotPortal

`kernel-context` 库提供了 `MultislotPortal` 来管理传送门：

```rust
// ch4/src/main.rs

// 传送门所在虚页（最高虚拟页）
const PROTAL_TRANSIT: VPN<Sv39> = VPN::MAX;  // 0x7FFFFFF

extern "C" fn rust_main() -> ! {
    // ...
    
    // 计算传送门需要的空间大小
    let portal_size = MultislotPortal::calculate_size(1);  // 1 个插槽
    let portal_layout = Layout::from_size_align(portal_size, 1 << Sv39::PAGE_BITS).unwrap();
    // 分配传送门的物理页
    let portal_ptr = unsafe { alloc(portal_layout) };
    
    // 建立内核地址空间（会映射传送门）
    let mut ks = kernel_space(layout, MEMORY, portal_ptr as _);
    
    // ...
}
```

### 传送门代码的工作原理

传送门代码执行以下步骤：

```asm
// kernel-context/src/foreign/mod.rs (简化版)

// 1. 保存当前状态到缓存
sd    a1, 1*8(a0)           // 保存 a1

// 2. 切换地址空间
ld    a1, 2*8(a0)           // 加载目标 satp
csrrw a1, satp, a1          // 交换 satp
sfence.vma                   // 刷新 TLB
sd    a1, 2*8(a0)           // 保存原 satp

// 3. 加载目标上下文
ld    a1, 3*8(a0)           // 加载 sstatus
csrw  sstatus, a1
ld    a1, 4*8(a0)           // 加载 sepc
csrw  sepc, a1

// 4. 设置返回时的 trap 入口
la    a1, 1f                // trap 入口就在后面
csrrw a1, stvec, a1
sd    a1, 5*8(a0)           // 保存原 stvec

// 5. 恢复寄存器并进入用户态
ld    a1, 1*8(a0)
ld    a0, (a0)
sret                        // 进入用户态！

// ---- 用户态发生 trap 后回到这里 ----
1:
// 6. 保存用户寄存器
csrrw a0, sscratch, a0
sd    a1, 1*8(a0)

// 7. 恢复内核地址空间
ld    a1, 2*8(a0)
csrrw a1, satp, a1
sfence.vma
sd    a1, 2*8(a0)

// 8. 恢复内核上下文并返回
ld    a1, 1*8(a0)
ld    a0, 5*8(a0)
csrw  stvec, a0
jr    a0                    // 跳回内核的 trap 处理
```

## 调度流程

### 调度线程

在 `ch4/src/main.rs` 中，我们创建了一个专门的调度线程：

```rust
extern "C" fn rust_main() -> ! {
    // ... 初始化代码 ...
    
    // 建立调度栈
    const PAGE: Layout = unsafe { 
        Layout::from_size_align_unchecked(2 << Sv39::PAGE_BITS, 1 << Sv39::PAGE_BITS) 
    };
    let pages = 2;
    let stack = unsafe { alloc(PAGE) };
    ks.map_extern(
        VPN::new((1 << 26) - pages)..VPN::new(1 << 26),
        PPN::new(stack as usize >> Sv39::PAGE_BITS),
        VmFlags::build_from_str("_WRV"),
    );
    
    // 创建调度线程
    let mut scheduling = LocalContext::thread(schedule as _, false);
    *scheduling.sp_mut() = 1 << 38;  // 设置栈指针
    
    // 执行调度线程
    unsafe { scheduling.execute() };
    
    // 如果调度线程因异常返回，说明出错了
    log::error!("stval = {:#x}", stval::read());
    panic!("trap from scheduling thread: {:?}", scause::read().cause());
}
```

**为什么需要调度线程？**

调度线程将内核的异常域与应用的异常域分离。如果在调度循环中发生内核异常，会返回到 `rust_main` 中处理，而不是影响到应用的执行。

### 调度循环

```rust
extern "C" fn schedule() -> ! {
    // 1. 初始化传送门
    let portal = unsafe { MultislotPortal::init_transit(PROTAL_TRANSIT.base().val(), 1) };
    
    // 2. 初始化系统调用
    syscall::init_io(&SyscallContext);
    syscall::init_process(&SyscallContext);
    syscall::init_scheduling(&SyscallContext);
    syscall::init_clock(&SyscallContext);
    
    // 3. 调度循环
    while !unsafe { PROCESSES.is_empty() } {
        // 获取第一个进程
        let ctx = unsafe { &mut PROCESSES[0].context };
        
        // 执行进程（会切换地址空间！）
        unsafe { ctx.execute(portal, ()) };
        
        // 处理 trap
        match scause::read().cause() {
            scause::Trap::Exception(scause::Exception::UserEnvCall) => {
                // 系统调用处理
                use syscall::{SyscallId as Id, SyscallResult as Ret};

                let ctx = &mut ctx.context;
                let id: Id = ctx.a(7).into();
                let args = [ctx.a(0), ctx.a(1), ctx.a(2), ctx.a(3), ctx.a(4), ctx.a(5)];
                
                match syscall::handle(Caller { entity: 0, flow: 0 }, id, args) {
                    Ret::Done(ret) => match id {
                        Id::EXIT => unsafe {
                            PROCESSES.remove(0);  // 进程退出
                        },
                        _ => {
                            *ctx.a_mut(0) = ret as _;  // 设置返回值
                            ctx.move_next();           // 移动到下一条指令
                        }
                    },
                    Ret::Unsupported(_) => {
                        log::info!("id = {id:?}");
                        unsafe { PROCESSES.remove(0) };  // 不支持的系统调用
                    }
                }
            }
            e => {
                // 其他异常（页错误等）
                log::error!(
                    "unsupported trap: {e:?}, stval = {:#x}, sepc = {:#x}",
                    stval::read(),
                    ctx.context.pc()
                );
                unsafe { PROCESSES.remove(0) };  // 终止进程
            }
        }
    }
    
    // 所有进程结束，关机
    system_reset(Shutdown, NoReason);
    unreachable!()
}
```

## 系统调用处理

### 跨地址空间访问用户数据

由于内核和应用使用不同的地址空间，系统调用处理时需要特别注意地址翻译：

```rust
// ch4/src/main.rs (impls 模块)

impl IO for SyscallContext {
    fn write(&self, caller: Caller, fd: usize, buf: usize, count: usize) -> isize {
        match fd {
            STDOUT | STDDEBUG => {
                // buf 是应用的虚拟地址，不能直接使用！
                const READABLE: VmFlags<Sv39> = VmFlags::build_from_str("RV");
                
                // 需要通过应用的页表进行地址翻译
                if let Some(ptr) = unsafe { PROCESSES.get_mut(caller.entity) }
                    .unwrap()
                    .address_space
                    .translate(VAddr::new(buf), READABLE)
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
            _ => {
                log::error!("unsupported fd: {fd}");
                -1
            }
        }
    }
}
```

### clock_gettime 系统调用

同样需要地址翻译：

```rust
impl Clock for SyscallContext {
    fn clock_gettime(&self, caller: Caller, clock_id: ClockId, tp: usize) -> isize {
        // tp 是应用提供的指针，需要翻译
        const WRITABLE: VmFlags<Sv39> = VmFlags::build_from_str("W_V");
        
        match clock_id {
            ClockId::CLOCK_MONOTONIC => {
                if let Some(mut ptr) = unsafe { PROCESSES.get(caller.entity) }
                    .unwrap()
                    .address_space
                    .translate(VAddr::new(tp), WRITABLE)  // 检查可写权限
                {
                    let time = riscv::register::time::read() * 10000 / 125;
                    *unsafe { ptr.as_mut() } = TimeSpec {
                        tv_sec: time / 1_000_000_000,
                        tv_nsec: time % 1_000_000_000,
                    };
                    0
                } else {
                    log::error!("ptr not writable");
                    -1
                }
            }
            _ => -1,
        }
    }
}
```

## 异常处理

### 页错误

当应用访问未映射或权限不足的地址时，会触发页错误：

```rust
// 调度循环中的异常处理
e => {
    log::error!(
        "unsupported trap: {e:?}, stval = {:#x}, sepc = {:#x}",
        stval::read(),  // 触发异常的地址
        ctx.context.pc() // 触发异常的指令地址
    );
    unsafe { PROCESSES.remove(0) };  // 终止进程
}
```

常见的页错误类型：
- `LoadPageFault`：读取未映射或不可读的地址
- `StorePageFault`：写入未映射或不可写的地址
- `InstructionPageFault`：执行未映射或不可执行的地址

## 与第三章的对比

| 特性 | 第三章 | 第四章 |
|------|--------|--------|
| 上下文类型 | `LocalContext` | `ForeignContext` |
| 执行方式 | `ctx.execute()` | `ctx.execute(portal, key)` |
| 地址空间切换 | 无 | 通过传送门自动切换 |
| 系统调用访问用户数据 | 直接使用指针 | 需要 `translate()` |
| 异常处理 | 直接使用 `stval` | 检查是否为页错误 |
| 调度方式 | 主函数循环 | 独立的调度线程 |

## 执行流程总结

```
                         rust_main
                             │
                             ↓
                    初始化内核地址空间
                    加载应用程序
                             │
                             ↓
                       schedule()
                             │
        ┌────────────────────┼────────────────────┐
        ↓                    ↓                    ↓
    Process 0            Process 1           Process N
        │                    │                    │
        ↓                    ↓                    ↓
  ctx.execute()        ctx.execute()        ctx.execute()
        │                    │                    │
        ↓                    ↓                    ↓
   [传送门]              [传送门]             [传送门]
   切换satp              切换satp             切换satp
        │                    │                    │
        ↓                    ↓                    ↓
   用户态执行            用户态执行           用户态执行
        │                    │                    │
      trap               trap                 trap
        │                    │                    │
        ↓                    ↓                    ↓
   [传送门]              [传送门]             [传送门]
   恢复satp              恢复satp             恢复satp
        │                    │                    │
        └────────────────────┼────────────────────┘
                             ↓
                     处理 trap/系统调用
                             │
                             ↓
                      继续调度循环
```

## 关键概念总结

| 概念 | 说明 |
|------|------|
| ForeignContext | 跨地址空间的上下文，包含 satp |
| MultislotPortal | 传送门，用于地址空间切换 |
| 调度线程 | 专门的线程，分离内核和应用的异常域 |
| 地址翻译 | 系统调用时将用户虚拟地址转为内核可访问的地址 |
| 页错误 | 访问未映射或权限不足的地址时触发 |

至此，我们实现了一个基于虚拟内存的分时多任务操作系统。每个应用程序都运行在独立的地址空间中，彼此隔离，安全性大大提高。
