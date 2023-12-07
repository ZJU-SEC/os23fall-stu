# Lab 6: 实现 fork 机制

> 强烈建议大家先通读整篇实验指导，**完成思考题后再动手写代码**。

## 实验目的
* 为 task 加入 **fork** 机制，能够支持通过 **fork** 创建新的用户态 task 。

## 实验环境 
* Environment in previous labs.

## 背景知识

### `fork` 系统调用

`fork` 是 Linux 中的重要系统调用，它的作用是将进行了该系统调用的 task 完整地复制一份，并加入 Ready Queue。这样在下一次调度发生时，调度器就能够发现多了一个 task，从这时候开始，新的 task 就可能被正式从 Ready 调度到 Running，而开始执行了。需留意，fork 具有以下特点：

* 子 task 和父 task 在不同的内存空间上运行。
* 关于 fork 的返回值：
  * `fork` 成功时，父 task  返回值为子 task 的 pid，子 task 返回值为 0。
  * `fork` 失败时，父 task  返回值为 -1。
* 创建的子 task 需要**深拷贝** `task_struct`，并且调整自己的页表、栈 和 CSR 等信息，同时还需要复制一份在用户态会用到的内存（用户态的栈、程序的代码和数据等），并且将自己伪装成是一个因为调度而加入了 Ready Queue 的普通程序来等待调度。在调度发生时，这个新 task 就像是原本就在等待调度一样，被调度器选择并调度。
<!-- * 创建的子 task 需要拷贝父 task  `task_struct`、`pgd`、`mm_struct` 以及父 task 的 `user stack` 等信息。 -->
* Linux 中使用了 `copy-on-write` 机制，`fork` 创建的子 task 首先与父 task 共享物理内存空间，直到父子 task 有修改内存的操作发生时再为子 task 分配物理内存（因为逻辑较为复杂，不要求所有同学都实现，如果你觉得这个机制很有趣，可以在实验中完成 COW 机制）。

### `fork` 在 Linux 中的实际应用

Linux 的另一个重要系统调用是 `exec`，它的作用是将进行了该系统调用的 task 换成另一个 task 。这两个系统调用一起，支撑起了 Linux 处理多任务的基础。当我们在 shell 里键入一个程序的目录时，shell（比如 zsh 或 bash）会先进行一次 fork，这时候相当于有两个 shell 正在运行。然后其中的一个 shell 根据 `fork` 的返回值（是否为 0），发现自己和原本的 shell 不同，再调用 `exec` 来把自己给换成另一个程序，这样 shell 外的程序就得以执行了。

> 如果你对 fork 的更多细节感兴趣，可以在 Linux 命令行中执行以下指令：
> `man 2 fork`。
> GNU/Linux 提供了内容丰富的离线手册，它被分为若干章节，每个章节记录了不同的内容。其中，man 2，也就是手册的第二章，记录的是与 Linux 系统调用相关的帮助文档。



## 实验步骤

### 实现 fork()

#### sys_clone
Fork 在早期的 Linux 中就被指定了名字，叫做 `clone`,
```c
#define SYS_CLONE 220
```
我们在实验原理中说到，fork 的关键在于状态和内存的复制。我们不仅需要完整地**深拷贝**一份页表以及 VMA 中记录的用户态的内存，还需要复制内核态的寄存器状态和内核态的内存。并且在最后，需要将 task “伪装”成是因为调度而进入了 Ready Queue。

回忆一下我们是怎样使用 `task_struct` 的，我们并不是分配了一块刚好大小的空间，而是分配了一整个页，并将页的高处作为了 task 的内核态栈。

```
                    ┌─────────────┐◄─── High Address
                    │             │
                    │    stack    │
                    │             │
                    │             │
              sp ──►├──────┬──────┤
                    │      │      │
                    │      ▼      │
                    │             │
                    │             │
                    │             │
                    │             │
    4KB Page        │             │
                    │             │
                    │             │
                    │             │
                    ├─────────────┤
                    │             │
                    │             │
                    │ task_struct │
                    │             │
                    │             │
                    └─────────────┘◄─── Low Address
```

也就是说，内核态的所有数据，包括了栈、陷入内核态时的寄存器，还有上一次发生调度，调用 `switch_to` 时的 `thread_struct` 信息，都被存在了这短短的 4K 内存中。这给我们的实现带来了很大的方便，这 4K 空间里的数据就是我们需要的所有内核态数据了！

除了内核态之外，你还需要**深拷贝**一份页表，并遍历页表中映射到 parent task 用户地址空间的页表项（为了减小开销，你需要根据 parent task 的 vmas 来 walk page table），这些应该由 parent task 专有的页，如果已经分配并映射到 parent task 的地址空间中了，就需要你另外分配空间，并从原来的内存中拷贝数据到新开辟的空间，然后将新开辟的页映射到 child task 的地址空间中。想想为什么只要拷贝那些已经分配并映射的页；那些本来应该被分配并映射，但是暂时还没有因为 Page Fault 而被分配并映射的页怎么办?

#### __ret_from_fork

让 fork 出来的 task 被正常调度是本实验**最重要**的部分。我们在 Lab2 中有一道思考题

> 2. 当线程第一次调用时, 其 `ra` 所代表的返回点是 `__dummy`。那么在之后的线程调用中 `context_switch` 中, `ra` 保存/恢复的函数返回点是什么呢? 请同学用 gdb 尝试追踪一次完整的线程切换流程, 并关注每一次 `ra` 的变换 (需要截图)。

经过了对这个问题的思考，我们可以认识到，当一个程序第一次被调度时，其实是可以指明返回到哪一个位置的指令的。例如我们当时执行的 `__dummy`, 就替代了正常从 `switch_to` 返回的执行流。这次我们同样使用这个技巧，通过修改 `task_struct->thread.ra`，让程序 `ret` 时，直接跳转到我们设置的 symbol `__ret_from_fork`。 

我们在 `_traps` 中的 `jal x1, trap_handler` 后面插入一个符号：

```asm
    .global _traps
_traps:
    ...
   jal x1, trap_handler
    .global __ret_from_fork
__ret_from_fork:
    ... ;利用 sp 从栈中恢复出寄存器的值 
    sret
```

继续回忆，我们的 `__switch_to` 逻辑的后半段，就是从 `task_struct->thread` 中恢复 callee-saved registers 的值，其中正包括了我们恢复寄存器值所需要的 sp。

自此我们知道，我们可以利用这两个寄存器，完成一个类似于 ROP (return oriented programming) 的操作。也就是说，我们通过控制 `ra` 寄存器，来控制程序的执行流，让它跳转到 context switch 的后半段；通过控制 `sp` 寄存器，从内核态的栈上恢复出我们在 `sys_clone` 时拷贝到新的 task 的栈上的，原本在 context switch 时被压入父 task 的寄存器值，然后通过 sret 直接跳回用户态执行用户态程序。

于是，父 task 的返回路径是这样的：`sys_clone->trap_handler->_traps->user program`，而我们新 `fork` 出来的 task，则要以这样的路径返回: `__switch_to->__ret_from_fork(in _traps)->user program`。

#### Code Skeleton

某知名体系结构课程老师说过，skeleton 是给大家参考用的，不是给大家直接抄的。接下来我们给大家的代码框架**理论上**可以直接运行，因为在写作实验文档前某助教刚刚自己完整实现了一次。但是我们的当前的框架是“最小的”——虽然一定能跑，但是同学们照着这个来写可能会有一些不方便。同学可以自行修改框架，来更好地贴合自己的实现。

我们要在存储所有 task 的数组 `task` 中寻找一个空闲的位置。我们用最简单的管理方式，将原本的 `task` 数组的大小开辟成 16，IDLE task 和 初始化时新建的 task 各占用一个，剩余 14 个全部赋值成 NULL。 如果 `task[pid] == NULL`, 说明这个 pid 还没有被占用，可以作为新 task 的 pid，并将 `task[pid]` 赋值为新的 `struct task_struct*`。由于我们的实验中不涉及 task 的销毁，所以这里的逻辑可以只管填充，不管擦除。

在实现中，你需要始终思考的问题是，怎么才能够**让新创建的 task 获得调度后，正确地跳转到 `__ref_from_fork`, 并且利用 `sp` 正确地从内存中取值**。为了简单起见，`sys_clone` 只接受一个参数 `pt_regs *`，下面是代码框架：

```c

uint64_t sys_clone(struct pt_regs *regs) {
    /*
     1. 参考 task_init 创建一个新的 task，将的 parent task 的整个页复制到新创建的 
        task_struct 页上(这一步复制了哪些东西?）。将 thread.ra 设置为 
        __ret_from_fork，并正确设置 thread.sp
        (仔细想想，这个应该设置成什么值?可以根据 child task 的返回路径来倒推)

     2. 利用参数 regs 来计算出 child task 的对应的 pt_regs 的地址，
        并将其中的 a0, sp, sepc 设置成正确的值(为什么还要设置 sp?)

     3. 为 child task 申请 user stack，并将 parent task 的 user stack 
        数据复制到其中。 (既然 user stack 也在 vma 中，这一步也可以直接在 5 中做，无需特殊处理)
        
     3.1. 同时将子 task 的 user stack 的地址保存在 thread_info->
        user_sp 中，如果你已经去掉了 thread_info，那么无需执行这一步

     4. 为 child task 分配一个根页表，并仿照 setup_vm_final 来创建内核空间的映射

     5. 根据 parent task 的页表和 vma 来分配并拷贝 child task 在用户态会用到的内存

     6. 返回子 task 的 pid
    */
}
```

<!-- // uint64_t sys_clone(struct pt_regs *regs) {
//     return do_fork(regs);
// }
// ```
// * 参考 `_trap` 中的恢复逻辑 在 `entry.S` 中实现 `ret_from_fork`，函数原型如下：
//     * 注意恢复寄存器的顺序：
//       * `a0` 应该最后被恢复
//       * `sp` 不用恢复
//       * 想想为什么?
//     * `_trap` 中是从 `stack` 上恢复，这里从 `trapframe` 中恢复
// ```c
// void ret_from_fork(struct pt_regs *trapframe);
// ```
// * 修改 `Page Fault` 处理: 在之前的 `Page Fault` 处理中，我们对用户栈 `Page Fault` 处理方法是自由分配一页作为用户栈并映射到`[USER_END - PAGE_SIZE, USER_END)` 的虚拟地址。但由 `fork` 创建的 task ，它的用户栈已经拷贝完毕，因此 `Page Fault` 处理时直接为该页建立映射即可 (可以通过  `thread_info->user_sp` 来进行判断)。
 -->

### 编译及测试
在测试时，由于大家电脑性能都不一样，如果出现了时钟中断频率比用户打印频率高很多的情况，可以减少用户程序里的 while 循环的次数来加快打印。这里的实例仅供参考，只要 OS 和用户态程序运行符合你的预期，那就是正确的。这里以我们给出的第三段 `main` 程序为例：

- 输出示例

```
    OpenSBI v0.9
     ____                    _____ ____ _____
    / __ \                  / ____|  _ \_   _|
   | |  | |_ __   ___ _ __ | (___ | |_) || |
   | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
   | |__| | |_) |  __/ | | |____) | |_) || |_
    \____/| .__/ \___|_| |_|_____/|____/_____|
          | |
          |_|

    Platform Name             : riscv-virtio,qemu
    Platform Features         : timer,mfdeleg
    Platform HART Count       : 1
    Firmware Base             : 0x80000000
    Firmware Size             : 100 KB
    Runtime SBI Version       : 0.2

    Domain0 Name              : root
    Domain0 Boot HART         : 0
    Domain0 HARTs             : 0*
    Domain0 Region00          : 0x0000000080000000-0x000000008001ffff ()
    Domain0 Region01          : 0x0000000000000000-0xffffffffffffffff (R,W,X)
    Domain0 Next Address      : 0x0000000080200000
    Domain0 Next Arg1         : 0x0000000087000000
    Domain0 Next Mode         : S-mode
    Domain0 SysReset          : yes

    Boot HART ID              : 0
    Boot HART Domain          : root
    Boot HART ISA             : rv64imafdcsu
    Boot HART Features        : scounteren,mcounteren,time
    Boot HART PMP Count       : 16
    Boot HART PMP Granularity : 4
    Boot HART PMP Address Bits: 54
    Boot HART MHPM Count      : 0
    Boot HART MHPM Count      : 0
    Boot HART MIDELEG         : 0x0000000000000222
    Boot HART MEDELEG         : 0x000000000000b109
    [S] buddy_init done!
    [S] Initialized: pid: 1, priority: 1, counter: 0
    [S] 2022 Hello RISC-V
    [S] Value of sstatus is 8000000000006002
    [S] Set schedule: pid: 1, priority: 1, counter: 4
    [S] Switch to: pid: 1, priority: 1, counter: 4
    [S] Supervisor Page Fault, scause: 000000000000000c
    [S] Supervisor Page Fault, scause: 000000000000000f, stval: 0000003ffffffff8, sepc: 0000000000010158
    [S] Supervisor Page Fault, scause: 000000000000000d, stval: 0000000000011a00, sepc: 000000000001017c
    [U] pid: 1 is running!, global_variable: 0
    [U] pid: 1 is running!, global_variable: 1
    [U] pid: 1 is running!, global_variable: 2
    [S] New task: 2
    [U-PARENT] pid: 1 is running!, global_variable: 3
    [U-PARENT] pid: 1 is running!, global_variable: 4
    [U-PARENT] pid: 1 is running!, global_variable: 5
    [S] Supervisor Mode Timer Interrupt
    [U-PARENT] pid: 1 is running!, global_variable: 6
    [U-PARENT] pid: 1 is running!, global_variable: 7
    [S] Supervisor Mode Timer Interrupt
    [U-PARENT] pid: 1 is running!, global_variable: 8
    [U-PARENT] pid: 1 is running!, global_variable: 9
    [S] Supervisor Mode Timer Interrupt
    [U-PARENT] pid: 1 is running!, global_variable: 10
    [U-PARENT] pid: 1 is running!, global_variable: 11
    [S] Supervisor Mode Timer Interrupt
    [S] Switch to: pid: 2, priority: 1, counter: 3
    [U-CHILD] pid: 2 is running!, global_variable: 3
    [U-CHILD] pid: 2 is running!, global_variable: 4
    [U-CHILD] pid: 2 is running!, global_variable: 5
    [S] Supervisor Mode Timer Interrupt
    [U-CHILD] pid: 2 is running!, global_variable: 6
    [U-CHILD] pid: 2 is running!, global_variable: 7
    [S] Supervisor Mode Timer Interrupt
    [U-CHILD] pid: 2 is running!, global_variable: 8
    [U-CHILD] pid: 2 is running!, global_variable: 9
    [S] Supervisor Mode Timer Interrupt
    [S] Set schedule: pid: 1, priority: 1, counter: 10
    [S] Set schedule: pid: 2, priority: 1, counter: 4
    [S] Switch to: pid: 2, priority: 1, counter: 4
    [U-CHILD] pid: 2 is running!, global_variable: 10
    
```

## 思考题

下面用*斜体*标出的部分是这篇实验指导中的原文。请找到原文所在的位置，基于上下文来回答这些问题：

1. *参考 task_init 创建一个新的 task，将的 parent task 的整个页复制到新创建的 task_struct 页上*。这一步复制了哪些东西?
2. *将 thread.ra 设置为 `__ret_from_fork`，并正确设置 `thread.sp`*。仔细想想，这个应该设置成什么值？可以根据 child task 的返回路径来倒推。
3. *利用参数 regs 来计算出 child task 的对应的 pt_regs 的地址，并将其中的 a0, sp, sepc 设置成正确的值*。为什么还要设置 sp?

## 作业提交
你需要提交实验报告以及整个工程代码。在提交前请使用 `make clean` 清除所有构建产物。

## 更多测试样例

下面是同学提供的测试样例，不强制要求大家都运行一遍。但是如果想增强一下对自己写的代码的信心，可以尝试替换 `main` 并运行。如果你有其他适合用来测试的代码，欢迎为仓库做出贡献。

[lhjgg](https://frightenedfoxcn.github.io/blog/) 给出的样例：

```c
#define LARGE 1000

unsigned long something_large_here[LARGE] = {0};

int fib(int times) {
  if (times <= 2) {
    return 1;
  } else {
    return fib(times - 1) + fib(times - 2);
  }
}

int main() {
  for (int i = 0; i < LARGE; i++) {
    something_large_here[i] = i;
  }
  int pid = fork();
  printf("[U] fork returns %d\n", pid);

  if (pid == 0) {
    while(1) {
      printf("[U-CHILD] pid: %ld is running! the %dth fibonacci number is %d and the number @ %d in the large array is %d\n", getpid(), global_variable, fib(global_variable), LARGE - global_variable, something_large_here[LARGE - global_variable]);
      global_variable++;
      for (int i = 0; i < 0xFFFFFF; i++);
    }
  } else {
    while (1) {
      printf("[U-PARENT] pid: %ld is running! the %dth fibonacci number is %d and the number @ %d in the large array is %d\n", getpid(), global_variable, fib(global_variable), LARGE - global_variable, something_large_here[LARGE - global_variable]);
      global_variable++;
      for (int i = 0; i < 0xFFFFFF; i++);
    }
  }
}
```

> 有一个可能的bug就是两个线程算出来的同一个斐波那契数不一致，这时候就是用户栈切换的问题，其他样例应该测不出来. 删了几行之后的错误示范（
> ![1 ZE$IPCYT2}N)7~XIBG3_1](https://user-images.githubusercontent.com/57927141/205851029-d283ecd8-d2b2-4aa9-9af3-bdd94fe1aea9.png)