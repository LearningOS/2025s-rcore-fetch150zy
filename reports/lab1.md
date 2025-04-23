# ch3实验报告
## 总结
任务要求：在目前的多任务分时操作系统上添加一个新的系统调用来实现跟踪当前任务的系统调用的历史信息  

- trace_request 为 0 和 1 的情况比较简单
- trace_request 为 2 时要求实现一个返回当前任务 syscall 的次数 (只在功能上考虑)
    - 需要一个与 current_task 绑定的数据结构来存储不同 syscall 的次数统计信息，最简单的方式就是使用一个数组来存，最大的 syscall id 为 410，开 512 足够（
    - 然后就是这个数组该存在哪？TaskManagerInner 中的话，那么就需要开二维数组才能和 current_task 进行绑定。如果放到 TaskControlBlock 中的话，一维数组即可。当前的实现是放在 TaskManagerInner 中。
    - 何时更新 syscall_counts？最直观的想法是在 syscall 处直接递增？在 task/mod.rs 中封装一个 increment_syscall_count 函数即可。
    - 然后再封装一个 get_current_syscall_counts 用于获取当前任务的特定 syscall 次数

## 问答
### Q1
```bash
$ make test CHAPTER=2
[kernel] PageFault in application, bad addr = 0x0, bad instruction = 0x804003a4, kernel killed it.
[kernel] IllegalInstruction in application, kernel killed it.
[kernel] IllegalInstruction in application, kernel killed it.
[kernel] Panicked at src/syscall/fs.rs:11 called `Result::unwrap()` on an `Err` value: Utf8Error { valid_up_to: 3, error_len: Some(1) }
```
RustSBI version: RustSBI-QEMU Version 0.2.0-alpha.2  

- `bad_address`: 试图向 0x0 写入数据，导致了 pagefault
- `bad_instructions`: 试图在用户态程序中使用 sret (在不合适的上下文中)
- `bad_register`: 在用户态试图读取 CSR 寄存器 (sstatus)


### Q2
#### 1
sp 代表内核栈的栈顶  
task 切换时恢复任务上下文  
syscall 返回 (恢复用户态程序的上下文)

#### 2
```assembly
ld t0, 32*8(sp)   # 从栈空间加载 sstatus 到 t0 寄存器
ld t1, 33*8(sp)   # 从栈空间记载 sepc 到 t1 寄存器
ld t2, 2*8(sp)    # 从栈空间加载 sscratch 到 t2 寄存器
csrw sstatus, t0  # 将 t0 中的值写入到 sstatus 寄存器
csrw sepc, t1     # 将 t1 中的值写入到 sepc 寄存器
csrw sscratch, t2 # 将 t2 中的值写入到 sscratch 寄存器
```
其实就是从栈种恢复了 sstatus sepc sscratch 这三个 CSR 寄存器的内容  
要看这些寄存器对进入用户态有什么意义，从这些寄存器的功能上分析即可
- `sstatus`: 中断
- `sepc`: 程序发生异常或中断的 PC 值
- `sscratch`: 用户栈地址

#### 3
分析 x2 和 x4 寄存器的作用即可
- `x2`: 栈指针 (sp)，在处理 trap 时 sp 会被用来指向当前的栈顶，对用户态程序来说没意义
- `x4`: 线程指针 (tp)，同上

#### 4
- `sp`: user stack
- `sscratch`: kernel stack

#### 5
发生在 sret，sret 会从 S -> U

#### 6
- `sp`: kernel stack
- `sscratch`: user stack

#### 7
U -> S or S -> M 都是通过 ecall