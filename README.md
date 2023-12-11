# oslab5

# 实验报告
## 练习0：填写已有实验

本实验依赖实验2/3/4。请把你做的实验2/3/4的代码填入本实验中代码中有“LAB2”/“LAB3”/“LAB4”的注释相应部分。注意：为了能够正确执行lab5的测试应用程序，可能需对已完成的实验2/3/4的代码进行进一步改进。
1、alloc_proc
```c
static struct proc_struct *
alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
    proc->state = PROC_UNINIT;
    proc->pid = -1;
    proc->runs = 0;
    proc->kstack = 0;
    proc-> need_resched = 0;
    proc->parent = NULL;
    proc->mm = NULL;
    memset(&(proc->context), 0, sizeof(struct context));
    proc->tf = NULL;
    proc->cr3 = boot_cr3;
    proc->flags = 0;
    memset(&(proc->name), 0, PROC_NAME_LEN + 1);                
    proc->wait_state = 0;//进程控制块中新增的条目，初始化进程等待状态
    proc->cptr = proc->optr = proc->yptr = NULL;//进程相关指针初始化    
}
    return proc;
}
```
2、 do_fork
```c
int
do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    int ret = -E_NO_FREE_PROC;
    struct proc_struct *proc;
    if (nr_process >= MAX_PROCESS) {
        goto fork_out;
    }
    ret = -E_NO_MEM;

    if ((proc = alloc_proc()) == NULL)
        goto fork_out;
    proc->parent = current;
    //确保当前进程的wait_state为0
    assert(current->wait_state == 0);
    
    if (setup_kstack(proc) != 0)
        goto bad_fork_cleanup_proc;
    if (copy_mm(clone_flags, proc) != 0)
        goto bad_fork_cleanup_kstack;
    copy_thread(proc, stack, tf);
    bool intr_flag;
    local_intr_save(intr_flag);
    proc->pid = get_pid();
    hash_proc(proc);
    // 将新创建的进程插入到当前进程的子进程链表中
    set_links(proc);
    
    local_intr_restore(intr_flag);
    wakeup_proc(proc);
    ret = proc->pid;
 
fork_out:
    return ret;

bad_fork_cleanup_kstack:
    put_kstack(proc);
bad_fork_cleanup_proc:
    kfree(proc);
    goto fork_out;
}
```

## 练习1: 加载应用程序并执行（需要编码）
do_execv函数调用load_icode（位于kern/process/proc.c中）来加载并解析一个处于内存中的ELF执行文件格式的应用程序。你需要补充load_icode的第6步，建立相应的用户内存空间来放置应用程序的代码段、数据段等，且要设置好proc_struct结构中的成员变量trapframe中的内容，确保在执行此进程后，能够从应用程序设定的起始执行地址开始执行。需设置正确的trapframe内容。
请在实验报告中简要说明你的设计实现过程。
•	请简要描述这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。
```c
    //(6) setup trapframe for user environment
    // 设置用户环境下的中断栈帧
    struct trapframe *tf = current->tf;
    // Keep sstatus
    uintptr_t sstatus = tf->status;
    memset(tf, 0, sizeof(struct trapframe));
    tf->gpr.sp=USTACKTOP; 
    tf->epc = elf->e_entry; 
    tf->status = sstatus & ~(SSTATUS_SPP | SSTATUS_SPIE); 
    ret = 0;
```
为了能够恢复用户进程的状态，让用户进程能够从内核模式正确返回到用户模式，我们需要进行以下设置：

1. `tf->gpr.sp = USTACKTOP;`：这行代码设置了用户态的栈顶指针。在用户模式下，栈顶指针应该指向用户栈的顶部（USTACKTOP），这样用户程序才能正确地在栈上进行操作。

2. `tf->epc = elf->e_entry;`：这行代码设置了异常程序计数器（epc）。epc 应该指向用户程序的入口点，这样当从内核模式返回到用户模式时，CPU 会从这个地址开始执行用户程序。

3. `tf->status = sstatus & ~(SSTATUS_SPP | SSTATUS_SPIE);`：这行代码设置了用户进程的状态寄存器。在 RISC-V 架构中，SSTATUS 寄存器的 SPP 位用于保存之前的特权级别，SPIE 位用于保存之前的中断使能状态。这行代码清除了 SPP 和 SPIE 位，这样当从内核模式返回到用户模式时，特权级别会被设置为用户模式，中断会被禁用。



###•	请简要描述这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。
创建user_main进程后，由do_wait函数确认存在RUNNABLE的子进程后，调用schedule函数来运行此进程，调用proc_run切换当前进程为要运行的进程，切换页表，使用新进程的地址空间。然后调用switch_to()函数实现上下文切换。完成上下文切换后，forkrets再把传进来的进程的中断帧放在sp，跳转到__trapret。__trapret函数直接从中断帧里面恢复所有的寄存器，然后通过sret指令，跳转到user_main。user_main通过宏定义调用kernel_execve，通过ebreak 触发中断处理来实现内核态的系统调用，通过设置SYS_exec为系统调用号转发到syscall()，进而执行do_execve函数

do_execve检查虚拟内存空间的合法性，释放虚拟内存空间，加载应用程序，创建新的mm结构和页目录表。通过调用load_icode函数加载新的可执行程序并建立新的内存映射关系，并且设置用户环境下的中断帧，让用户进程能够从内核模式正确返回到用户模式。

Loade_icode执行完后，返回到exception_handler，继续执行kernel_execve_ret。
kernel_execve_ret会调整栈指针并复制上一个中断帧的内容到新的中断帧中。然后跳转到_trapret函数。

通过__trapret函数RESTORE_ALL，然后sret跳转到epc指向的函数。由于load_icode中设置用户环境下的中断帧，`tf->epc = elf->e_entry;` 将epc 指向用户程序的入口点，进而得以执行应用程序第一条指令。



##练习2: 父进程复制自己的内存空间给子进程（需要编码）
创建子进程的函数do_fork在执行中将拷贝当前进程（即父进程）的用户内存地址空间中的合法内容到新进程中（子进程），完成内存资源的复制。具体是通过copy_range函数（位于kern/mm/pmm.c中）实现的，请补充copy_range的实现，确保能够正确执行。
请在实验报告中简要说明你的设计实现过程。
•	如何设计实现Copy on Write机制？给出概要设计，鼓励给出详细设计。
```c
           // 获取进程 A 的页的内核虚拟地址，作为 memcpy 函数的源地址
            void* src_kvaddr = page2kva(page);
            // 获取进程 B 的新页的内核虚拟地址，作为 memcpy 函数的目标地址
            void* dst_kvaddr = page2kva(npage); 
            // 将进程 A 的页的内容复制到进程 B 的新页
            memcpy(dst_kvaddr, src_kvaddr, PGSIZE);
            // 在进程 B 的页表中插入新页，建立虚拟地址到物理地址的映射
            ret = page_insert(to, npage, start, perm);
```


### •	如何设计实现Copy on Write机制？给出概要设计，鼓励给出详细设计。
概要设计：

1. 页表共享：在创建新进程（例如在 fork 操作中）时，不立即复制父进程的内存，而是让父进程和子进程共享同一份物理内存，同时将页表项设置为只读。

2. 处理写操作：当进程试图写入共享的内存页时，由于页表项是只读的，会触发一个页面保护异常。在异常处理程序中，检测到这是一个 COW 页面，就进行实际的物理复制。

3. 页面复制：为写操作分配一个新的内存页，复制原来的页面内容到新的内存页，然后更新页表，将新的内存页映射到原来的虚拟地址，同时设置页表项为可写。

4. 更新页表：如果其他进程还在共享原来的内存页，就保持原来的页表项为只读。如果没有其他进程共享，就可以释放原来的内存页。

详细设计：
do fork 部分：在进行内存复制的部分，比如 copy_range 函数内部，不实际进行内存的复制，而是将子进程和父进程的虚拟页映射上同—个物理页面，然后在分别在这两个进程的虚拟页对应的PTE 部分将这个页置成是不可写的，同时利用 PTE 中的保留位将这个页设置成共享的页面，这样的话如果应用程序试图写某—个共享页就会产生页访间异常，从而可以将控制权交给操作系统进行处理；

page fault 部分：在 page fault 的 ISR 部分，新增加对当前的异常是否由千尝试写了某—个共享页面引起的，如果是的话，额外申请分配—个物理页面，然后将当前的共享页的内容复制过去，建立 出错的线性地址与新创建的物理页面的映射关系，将 PTE 设置设置成非共享的；然后查询原先共享的物理页面是否还是由多个其它进程共享使用的，如果不是的话，就将对应的虚地址的 PTE 进行修改，删掉共享标记，恢复写标记；这样的话 page fault 返回之后就可以正常完成对虚拟内存（原想的共享内存）的写操作了；



## 练习3: 阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现（不需要编码）
请在实验报告中简要说明你对 fork/exec/wait/exit函数的分析。并回答如下问题：
•	请分析fork/exec/wait/exit的执行流程。重点关注哪些操作是在用户态完成，哪些是在内核态完成？内核态与用户态程序是如何交错执行的？内核态执行结果是如何返回给用户程序的？
1.系统调用分析
—般来说，用户进程只能执行—般的指令,无法执行特权指令。采用系统调用机制为用户进程提供—个获得操作系统服务的统—接口层，简化用户进程的实现。
应用程序调用的 exit/fork/wait/getpid 等库函数最终都会调用 syscall 函数，只是调用的参数不同而已（分别是 SYS_exit / SYS_fork / SYS_wait / SYS_getid ）
当应用程序调用系统函数时，调用syscall后，CPU 根据操作系统建立的系统调用中断描述符，转入内核态，然后开始了操作系统系统调用的执行过程，在内核函数执行之前，会保留调用前的执行现场，之后操作系统就可以开始完成具体的系统调用，完成后调用 sret 返回用户态，并恢复现场。这样整个系统调用就执行完毕了。

1.1. fork
调用过程为：fork-> SYS_fork-> do_fork+wakeup_proc

首先当程序执行 fork 时，fork 使用了系统调用 SYS_fork，而系统调用 SYS_fork 则主要是由 do_fork 和
wakeup_proc 来完成的。do_fork() 完成的工作在练习 2 及 lab4 中已经做过详细介绍，这里再简单说
—下，主要是完成了以下工作：

①分配并初始化进程控制块（ alloc_proc 函数）; ②分配并初始化内核栈，为内核进程（线程）建立栈空间（ setup_stack
函数）; ③根据 clone_ﬂag 标志复制或共享进程内存管理结构（ copy_mm 函数）;
④设置进程在内核（将来也包括用户态）正常运行和调度所需的中断帧和执行上下文 （ copy_thread 函数）; ⑤为进程分配—个 PID（
get_pid() 函数）; ⑥把设置好的进程控制块放入 hash_list 和 proc_list 两个全局进程链表中;
⑦自此，进程已经准备好执行了，把进程状态设置为“就绪”态; ⑧设置返回码为子进程的 PID 号。

而 wakeup_proc 函数主要是将进程的状态设置为等待，即 proc->wait_state = 0。

1.2. exec
调用过程为：SYS_exec->do_execve

当应用程序执行的时候，会调用 SYS_exec 系统调用，而当 ucore 收到此系统调用的时候，则会使用
do_execve() 函数来实现，因此这里我们主要介绍 do_execve() 函数的功能，函数主要时完成用户进程的创建工作，同时使用户进程进入执行。
主要工作如下：

①首先为加载新的执行码做好用户态内存空间清空准备。如果 mm 不为 NULL，则设置页表为内核空间页表，且进—步判断 mm 的引用计数减 1
后是否为 0，如果为 0，则表明没有进程再需要此进程所占用的内存空间，为此将根据 mm
中的记录，释放进程所占用户空间内存和进程页表本身所占空间。最后把当前进程的 mm 内存管理指针为空。
②接下来是加载应用程序执行码到当前进程的新创建的用户态虚拟空间中。之后就是调用 load_icode 从而使之准备好执行。（具体
load_icode 的功能在练习 1 已经介绍的很详细了，这里不赘述了）

1.3. wait
调用过程为： SYS_wait->do_wait

do_wait 函数的实现过程：
当执行 wait 功能的时候，会调用系统调用 SYS_wait，而该系统调用的功能则主要由 do_wait 函数实现，主要工作就是父进程如何完成对子进程的最后回收工作，具体的功能实现如下：

①如果 pid!=0，表示只找—个进程 id 号为 pid 的退出状态的子进程，否则找任意—个处千退出状态的子进程; ②
如果此子进程的执行状态不为 PROC_ZOMBIE，表明此子进程还没有退出，则当前进程设置执行状态为
PROC_SLEEPING（睡眠），睡眠原因为 WT_CHILD (即等待子进程退出)，调用 schedule()
函数选择新的进程执行，自己睡眠等待，如果被唤醒，则重复跳回步骤 1 处执行; ③ 如果此子进程的执行状态为
PROC_ZOMBIE，表明此子进程处千退出状态，需要当前进程(即子进程的父进程)完成对子进程的最终回收工作，即首先把子进程控制块从两个进程队列
proc_list 和 hash_list
中删除，并释放子进程的内核堆栈和进程控制块。自此，子进程才彻底地结束了它的执行过程，它所占用的所有资源均已释放。

1.4. exit
调用过程为：SYS_exit->exit
do_exit 函数的实现过程：

当执行 exit 功能的时候，会调用系统调用 SYS_exit，而该系统调用的功能主要是由 do_exit 函数实现。具体过程如下：

①先判断是否是用户进程，如果是，则开始回收此用户进程所占用的用户态虚拟内存空间;（具体 的回收过程不作详细说明）
②设置当前进程的中hi性状态为 PROC_ZOMBIE，然后设置当前进程的退出码为
error_code。表明此时这个进程已经无法再被调度了，只能等待父进程来完成最后的回收工作（主要是回收该子进 程的内核栈、进程控制块）
③如果当前父进程已经处千等待子进程的状态，即父进程的 wait_state 被置为
WT_CHILD，则此时就可以唤醒父进程，让父进程来帮子进程完成最后的资源回收工作。
④如果当前进程还有子进程,则需要把这些子进程的父进程指针设置为内核线程 init，且各个子进程指针需要插入到 init
的子进程链表中。如果某个子进程的执行状态是 PROC_ZOMBIE，则需要唤醒 init 来完成对此子进程的最后回收工作。 ⑤执行
schedule() 调度函数，选择新的进程执行


### •	请分析fork/exec/wait/exit的执行流程。重点关注哪些操作是在用户态完成，哪些是在内核态完成？内核态与用户态程序是如何交错执行的？内核态执行结果是如何返回给用户程序的？
fork 执行完毕后，如果创建新进程成功，则出现两个进程，—个是子进程，—个是父进程。在子进程中，fork 函数返回 0，在父进程中，fork 返回新创建子进程的进程 ID。我们可以通过 fork 返回的值来判断当前进程是子进程还是父进程。fork 不会影晌当前进程的执行状态，但是会将子进程的状态标记为RUNNALB，使得可以在后续的调度中运行起来；
exec 完成用户进程的创建工作。首先为加载新的执行码做好用户态内存空间清空准备。接下来的
—步是加载应用程序执行码到当前进程的新创建的用户态虚拟空间中。exec 不会影晌当前进程的执行状态，但是会修改当前进程中执行的程序；
wait 是等待任意子进程的结束通知。wait_pid 函数等待进程 id 号为 pid 的子进程结束通知。这两个函数最终访间 sys_wait 系统调用接口让 ucore 来完成对子进程的最后回收工作。wait 系统调用取决千是否存在可以释放资源（ZOMBIE）的子进程，如果有的话不会发生状态的改变，如果没有 的话会将当前进程置为 SLEEPING 态，等待执行了 exit 的子进程将其唤醒；
exit 会把—个退出码 error_code 传递给 ucore，ucore 通过执行内核函数 do_exit 来完成对当前进程的退出处理，主要工作简单地说就是回收当前进程所占的大部分内存资源，并通知父进程完成 最后的回收工作。exit 会将当前进程的状态修改为 ZOMBIE 态，并且会将父进程唤醒（修改为RUNNABLE），然后主动让出 CPU 使用权；

```c
                              |
                              | alloc_proc
                              |
                              |
                              V
                           UNINIT
                              |
                              | wakeup
                              |
                              |
            schedule          V              wakeup
RUNNING<------------------>RUNNABLE  <————————————————
 |                            |                      +
 |                            | do_kill              |
 |                            |                      |
 |                            |                      |
 |       do_exit              V                      |
+ ———————————————————————>   ZOMBIE              SLEEPING
 |                                                   |
 |                                                   |
 |                             do_wait               |
+————————————————————————————————————————————————————+


```











