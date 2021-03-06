Linux Process
================================================================================

基本概念
--------------------------------------------------------------------------------
进程: 程序执行的一个实例,目的就是担当分配系统资源(CPU时间,内存,文件等)的实体.

进程描述符
--------------------------------------------------------------------------------

为了管理进程,内核必须对每个进程所做的事情进行清楚的描述.例如:
进程优先级;给它分配了什么地址空间;允许它访问哪个文件等等.

路径: include/linux/sched.h
```
struct task_struct {
    volatile long state;    /* -1 unrunnable, 0 runnable, >0 stopped */

    void *stack;            // ==> "thread_info": 进程的基本信息.
    atomic_t usage;
    unsigned int flags;     /* per process flags, defined below */
    ......
    int exit_state;
    ......
    /* process credentials */  ==> 进程凭证
    /* objective and real subjective task credentials (COW) */
    const struct cred __rcu *real_cred;  /* ocontext */
    /* effective (overridable) subjective task credentials (COW) */
    const struct cred __rcu *cred;       /* scontext */
    ......
    struct tty_struct *tty; /* NULL if no tty */ ==> "tty_struct": 与进程相关的tty
    ......
    struct mm_struct *mm, *active_mm; // ==> "mm_struct": 指向内存区描述符的指针.
    ......
    /* filesystem information */
    struct fs_struct *fs;  // ==> "fs_struct": 文件系统信息
    /* open file information */
    struct files_struct *files;  // ==> "files_struct": 所有打开文件的信息
    /* namespaces */
    struct nsproxy *nsproxy; // 命名空间信息
    ......
    /* signal handlers */
    struct signal_struct *signal; // ==> "signal_struct": 所接收的信号
    ......
};
```

进程状态
--------------------------------------------------------------------------------

path: include/linux/sched.h

```
/* Task state bitmask. NOTE! These bits are also
 * encoded in fs/proc/array.c: get_task_state().
 *
 * We have two separate sets of flags: task->state
 * is about runnability, while task->exit_state are
 * about the task exiting. Confusing, but this way
 * modifying one set can't modify the other one by
 * mistake.
 */
#define TASK_RUNNING		0
#define TASK_INTERRUPTIBLE	1
#define TASK_UNINTERRUPTIBLE	2
#define __TASK_STOPPED		4
#define __TASK_TRACED		8
/* in tsk->exit_state */
#define EXIT_ZOMBIE		16
#define EXIT_DEAD		32
```

/* tsk->state */
* TASK_RUNNING: 进程是可执行的;它或者正在执行,或者在运行队列中等待执行.这是进程在用户空间中执行的唯一可能的状态.
* TASK_INTERRUPTIBLE: 进程正在睡眠,等待某些条件的达成(比如释放进程正在等待的资源).一旦这些条件达成,内核就会把进程状态设置为运行.
                      处于此状态的进程也会因为接收到信号而提前被唤醒并投入运行.
* TASK_UNINTERRUPTIBLE: 除了不对信号进行处理外,这个状态与TASK_INTERRUPTIBLE状态相同.
    例如,当进程在打开一个设备文件的时候,其相应的设备驱动程序开始探测相应的硬件设备时会用到这种状态.探测完成以前,设备驱动程序不能被中断.
* __TASK_TRACED: 被其它进程跟踪(ptrace).
* __TASK_STOPPED: 进程执行被暂停.当进程收到(SIGSTOP, SIGTSTP, SIGTTIN或SIGTTOU信号)后进入暂停状态.

/* tsk->state or tsk->exit_state */
* EXIT_ZOMBIE: 进程执行被终止,但是父进程还没发布wait4()或waitpid()系统掉哟你来返回有关死亡进程的信息.
               发布wait类系统调用前,内核不能丢弃包含在死进程描述符中的数据,因为父进程可能还需要它.
* EXIT_DEAD: 最终状态:父进程刚刚发出wait4()或waitpid()系统调用,因而进程由系统删除.为了防止其它执行线程在同一进程上执行
             wait()类系统调用而把进程状态由EXIT_ZOMBIE转为EXIT_DEAD状态.

进程描述符处理
--------------------------------------------------------------------------------

进程创建:
--------------------------------------------------------------------------------

## Sample

path: sample/dac/helloworld.c
```
#include <stdio.h>

int main(int argc, char* argv[])
{
    printf("hello world\n");
    return 0;
}
```

path: sample/fork_exec_helloworld.c
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#include <fcntl.h>
#include <unistd.h>

int main(int argc, char* argv[])
{
    int pid = fork();

    if (pid < 0) {
        fprintf(stderr, "fork a subprocess error: %d, %s",
                errno, strerror(errno));
        return EXIT_FAILURE;
    } else if (pid == 0) { /* child process */
        printf("execute child process: %d\n", getpid());
        execl("./helloworld", "helloworld", NULL);
        fprintf(stderr, "exec a subprocess error: %d, %s",
                errno, strerror(errno));
    }

    printf("execute parent process: %d\n", getpid());
    /* parent process */
    if (waitpid(pid, NULL, 0) < 0) {
        fprintf(stderr, "wait a subprocess error: %d, %s",
                errno, strerror(errno));
    }

    return EXIT_SUCCESS;
}
```

编译运行:

```
liminghao@liminghao-mi:~/leeminghao/selinux/sample/dac$ gcc helloworld.c -o helloworld
liminghao@liminghao-mi:~/leeminghao/selinux/sample/dac$ ./helloworld
hello world
liminghao@liminghao-mi:~/leeminghao/selinux/sample/dac$ gcc fork_exec_helloworld.c -o fork
liminghao@liminghao-mi:~/leeminghao/selinux/sample/dac$ ./fork
execute parent process: 22355
execute child process: 22356
hello world
```

## Fork
传统的Unix OS以统一的方式对待所有的进程:子进程复制父进程所拥有的所有的资源.
缺点: 创建慢且效率低下,因为子进程需要拷贝父进程的整个地址空间.实际上,子进程几乎不必读或修改父进程所有的资源,
   在很多情况下,子进程立即调用execve(),并清除父进程拷贝过来的地址空间.
现代内核:
* 写时复制技术: 允许父子进程读相同的物理页.只要两者中有一个试图写一个物理页,内核就把这个页内容拷贝到一个新的物理页.
  并把这个新的物理页分配给正在写的进程.
* 轻量级进程允许父子进程共享进程在内核的很多数据结构.如页表,打开文件表以及信号处理.
* vfork()系统调用创建的进程能共享其父进程的内存地址空间.为了防止父进程重写子进程需要的数据,阻止父进程的执行,一直到
  子进程退出或执行新的程序为止.


下面我们以bionic c库中的fork函数为例来讲解fork函数的执行过程:

#### fork

path: bionic/libc/bionic/fork.c
```
int  fork(void)
{
    int ret;
    ...
    ret = __fork();
    ...
    return ret;
}
```

#### __fork

path: bionic/libc/arch-arm/syscalls/__fork.S
```
ENTRY(__fork)
    mov     ip, r7          # 将r7寄存器的值保存到ip寄存器中.
    ldr     r7, =__NR_fork  # 将__NR_fork宏表示的值加载到r7寄存器中

    swi     #0
    mov     r7, ip
    cmn     r0, #(MAX_ERRNO + 1)
    bxls    lr
    neg     r0, r0
    b       __set_errno
END(__fork)
```

__NR_fork定义在kernel/arch/arm/include/asm/unistd.h中:
EABI是什么东西呢? ABI: Application Binary Interface, 应用二进制接口.
* 在较新的EABI规范中,是将系统调用号压入寄存器r7中;
* 而在老的OABI中则是执行的swi中断号的方式, 也就是说原来的调用方式(Old ABI)是通过跟随在swi指令中的调用号来进行的.

#### __NR_fork

path: kernel/arch/arm/include/asm/unistd.h
```
#define __NR_OABI_SYSCALL_BASE  0x900000

#if defined(__thumb__) || defined(__ARM_EABI__)
#define __NR_SYSCALL_BASE 0
#else
#define __NR_SYSCALL_BASE __NR_OABI_SYSCALL_BASE
#endif

/*
 * This file contains the system call numbers.
 */
#define __NR_restart_syscall  (__NR_SYSCALL_BASE+  0)
...
#define __NR_fork             (__NR_SYSCALL_BASE+  2)
```

**SWI软中断处理**:
在ARM平台上，使用了swi中断来实现系统调用的跳转. swi指令用于产生软件中断, 从而实现从用户模式变换到
管理模式, CPSR(Current Program Status Register), 程序状态寄存器,包含了条件标志位,中断禁止位,当前
处理器模式标志以及其他的一些控制和状态位)保存到管理模式的SPSR(Saved Program Status Register,程序
状态保存寄存器,用于保存CPSR的状态，以便异常返回后恢复异常发生时的工作状态),执行转移到swi向量,
在其他模式下也可使用swi指令，处理器同样地切换到管理模式。

指令格式如下:

```
swi {cond}  immed_24
```

其中：
immed_24 24位立即数，值为从0-16777215之间的整数。
使用swi指令时,通常使用以下两种方法进行参数传递,swi异常处理程序可以提供相关的服务,这两种方法均
是用户软件协定.swi异常中断处理程序要通过读取引起软件中断的swi指令,以取得24位立即数.

1).指令中24位的立即数指定了用户请求的服务类型,参数通过通用寄存器传递.如:

```
    MOV R0,#34
    SWI 12
```

2).指令中的24位立即数被忽略,用户请求的服务类型有寄存器R0的值决定,参数通过其他的通用寄存器传递. 如:

```
    MOV R0, #12
    MOV R1, #34
    SWI 0
```

在SWI异常处理程序中,取出SWI立即数的步骤为:
* 首先, 确定一起软中断的SWI指令时ARM指令还是Thumb指令,这可通过对SPSR访问得到;
* 然后, 取得该SWI指令的地址, 这可通过访问LR寄存器得到, 接着读出指令,分解出立即数(低24位).

系统会根据ABI的不同而将相应的系统调用表的基地址加载进tbl寄存器,接下来查找的过程:
ARM异常向量表:
#### vector_start

path: arch/arm/kernel/entry-armv.S中:
```
__vectors_start:
	W(b)	vector_rst
	W(b)	vector_und
	W(ldr)	pc, __vectors_start + 0x1000 @(__stubs_start == vector + 0x1000)
	W(b)	vector_pabt
	W(b)	vector_dabt
	W(b)	vector_addrexcptn
	W(b)	vector_irq
	W(b)	vector_fiq

```

#### vector_swi

对于swi软中断的中断函数入口表项如下:

path: arch/arm/kernel/entry-armv.S
```
__stubs_start:
	@ This must be the first word
	.word	vector_swi
```

最终,当使用swi触发软中断的时候将会调用vector_swi处的中断处理函数来处理对应的软件中断.
SWI Handler(swi软中断处理函数):

#### SWI handler

path: kernel/arch/arm/kernel/entry-common.S
```
/*=============================================================================
 * SWI handler
 *-----------------------------------------------------------------------------
 */

	.align	5
ENTRY(vector_swi)
        ...

	/*
         * Get the system call number.
         */

#if defined(CONFIG_OABI_COMPAT)
    ...
#elif defined(CONFIG_AEABI)
    ...
#elif defined(CONFIG_ARM_THUMB)
    /* Legacy ABI only, possibly thumb mode. */
    tst	r8, #PSR_T_BIT			@ this is SPSR from save_user_regs
    # r7 = r7 + (#__NR_SYSCALL_BASE)
    addne scno, r7, #__NR_SYSCALL_BASE	@ put OS number in
    ldreq scno, [lr, #-4]
#else
    ...
#endif
        ...

	enable_irq

	get_thread_info tsk

	adr	tbl, sys_call_table		@ load syscall table pointer

        ...

	cmp	scno, #NR_syscalls		@ check upper syscall limit
	adr	lr, BSYM(ret_fast_syscall)	@ return address

        # 从这里执行sys_call_table中fork对应的系统调用函数.
	ldrcc	pc, [tbl, scno, lsl #2]		@ call sys_* routine

        ...
ENDPROC(vector_swi)
```

sys_call_table 在内核中是个跳转表,这个表中存储的是一系列的函数指针,这些指针就是系统调用函数的
指针,sys_call_table的定义如下所示:

#### sys_call_table

path: kernel/arch/arm/kernel/entry-common.S
```
	.type	sys_call_table, #object
ENTRY(sys_call_table)
#include "calls.S"
```

将会从sys_call_table表中取出fork对应在内核态要执行的函数.

path: kernel/arch/arm/kernel/call.S
```
/* 0 */		CALL(sys_restart_syscall)
		CALL(sys_exit)
		CALL(sys_fork_wrapper)
```

#### sys_fork_wrapper

path: kernel/arch/arm/kernel/entry-common.S
```
sys_fork_wrapper:
		add	r0, sp, #S_OFF # 指定参数.
		b	sys_fork
ENDPROC(sys_fork_wrapper)
```

最终fork要执行的函数是sys_fork:

#### sys_fork

path: kernel/arch/arm/kernel/sys_arm.c
```
/* Fork a new task - this creates a new program thread.
 * This is called indirectly via a small wrapper
 */
asmlinkage int sys_fork(struct pt_regs *regs)
{
#ifdef CONFIG_MMU
	return do_fork(SIGCHLD, regs->ARM_sp, regs, 0, NULL, NULL);
#else
	/* can not support in nommu mode */
	return(-EINVAL);
#endif
}
```

sys_fork函数最终是调用do_fork来实现一个进程的创建:

#### do_fork

path: kernel/kernel/fork.c
```
/*
 *  Ok, this is the main fork-routine.
 *
 * It copies the process, and if successful kick-starts
 * it and waits for it to finish using the VM if required.
 */
long do_fork(unsigned long clone_flags,
           unsigned long stack_start,
           struct pt_regs *regs,
           unsigned long stack_size,
           int __user *parent_tidptr,
           int __user *child_tidptr)
{
        struct task_struct *p;
        int trace = 0;
        long nr;

        ......

        /* copy_process()复制进程描述符,该函数返回刚创建的task_struct描述符的地址. */
        p = copy_process(clone_flags, stack_start, regs, stack_size,
                         child_tidptr, NULL, trace);
        ......
}
```

#### copy_process

path: kernel/kernel/fork.c
```
/*
 * This creates a new process as a copy of the old one,
 * but does not actually start it yet.
 *
 * It copies the registers, and all the appropriate
 * parts of the process environment (as per the clone
 * flags). The actual kick-off is left to the caller.
 */
static struct task_struct *copy_process(unsigned long clone_flags,
    unsigned long stack_start,
    struct pt_regs *regs,
    unsigned long stack_size,
    int __user *child_tidptr,
    struct pid *pid,
    int trace)
{
        int retval;
        struct task_struct *p;
        int cgroup_callbacks_done = 0;

        /* 1. 检查clone_flags是否合法 */

        /* 2. 通过调用security_task_create(clone_flags)函数以及稍后的security_task_alloc(p)
         * 函数执行所有附加的安全检查.
         */
         retval = security_task_create(clone_flags);

        /* 3. 调用dup_task_struct为子进程获取进程描述符.
         */
         p = dup_task_struct(current);

        /* 4.得到的进程与父进程内容几乎完全一致，初始化新创建进程 */
        ......
        retval = -EAGAIN;
        if (atomic_read(&p->real_cred->user->processes) >=
           task_rlimit(p, RLIMIT_NPROC)) {
           if (!capable(CAP_SYS_ADMIN) && !capable(CAP_SYS_RESOURCE) &&
               p->real_cred->user != INIT_USER)
                    goto bad_fork_free;
        }
        current->flags &= ~PF_NPROC_EXCEEDED;
        /* 为新建的进程拷贝credentials */
        retval = copy_creds(p, clone_flags);
        if (retval < 0)
            goto bad_fork_free;

        return p;

        ......
}
```

当进程由于中断或系统调用从用户态转换到内核态时,进程所使用的栈也要从用户栈切换到内核栈.
通过内核栈获取栈尾thread_info,就可以获取当前进程描述符task_struct.每个进程的thread_info结构在
它的内核栈的尾端分配.

dup_task_struct的实现如下所示:

#### dup_task_struct

path: kernel/kernel/fork.c
```
static struct task_struct *dup_task_struct(struct task_struct *orig)
{
    struct task_struct *tsk;
    struct thread_info *ti;
    unsigned long *stackend;
    int node = tsk_fork_get_node(orig);
    int err;

    prepare_to_copy(orig);

    /* 1.执行alloc_task_struct_node()宏,为新进程获取进程描述符,并将描述符地址保存在tsk局部变量中.
     */
     tsk = alloc_task_struct_node(node);
     if (!tsk)
        return NULL;

    /* 2.执行alloc_thread_info_node宏以获取一块空闲内存区,用来存放新进程的thread_info结构和
     * 内核栈,并将这些块内存区字段的地址存在局部变量ti中.这块内存区的大小是8KB或4KB.
     */
     ti = alloc_thread_info_node(tsk, node);
     if (!ti) {
        free_task_struct(tsk);
        return NULL;
     }

    /* 3.将current进程描述符的内容复制到tsk所指向的task_struct结构中,然后把tsk->stack置为ti. */
     err = arch_dup_task_struct(tsk, orig);
     if (err)
        goto out;
     tsk->stack = ti;

    /* 4.把current进程的thread_info描述符的内容复制到ti中,然后把ti->task置为tsk.
     * 意味着子进程和父进程共享内核堆栈.
     */
     setup_thread_stack(tsk, orig);

     clear_user_return_notifier(tsk);
     clear_tsk_need_resched(tsk);
     stackend = end_of_stack(tsk);
     *stackend = STACK_END_MAGIC;    /* for overflow detection */

#ifdef CONFIG_CC_STACKPROTECTOR
     tsk->stack_canary = get_random_int();
#endif

    /*
     * One for us, one for whoever does the "release_task()" (usually
     * parent)
     */
    /* 5.把新进程描述符使用计数器(tsk->usage)置为2, 用来表示进程描述符正在被使用而且其
     * 相应的进程状态处于活动状态(进程状态既不是EXIT_ZOMBIE,也不是EXIT_DEAD).
     */
     atomic_set(&tsk->usage, 2);
#ifdef CONFIG_BLK_DEV_IO_TRACE
   tsk->btrace_seq = 0;
#endif
    tsk->splice_pipe = NULL;

    account_kernel_stack(ti, 1);

    return tsk;  /* 返回进程描述符 */

out:
    free_thread_info(ti);
    free_task_struct(tsk);
    return NULL;
}
```

#### copy_creds

https://github.com/leeminghao/doc-linux/blob/master/security/credentials/Credentials.md

**内核线程**: 内核线程(kernel_thread)或叫守护进程(daemon).在操作系统中占据相当大的比例,当Linux操作系统启动以后,
可以用”ps -ef”命令查看系统中的进程,这时会发现很多以"d"结尾的进程名,确切说名称显示里面加"[]"的,这些进程就是内核线程.
系统的启动是从硬件->内核->用户态进程的,pid的分配是一个往前的循环的过程,所以随系统启动的内核线程的pid往往很小.
内核线程与普通进程的区别:
* 内核线程只运行在内核态,而普通进程既可以运行在内核态也可以运行在用户态.
* 因为内核线程只运行在内核态,它们只能使用大于PAGE_OFFSET的线性地址空间.另一方面,不管在用户态还是内核态.普通进程都可以
  用4GB线性地址空间.

## 总结:

#### 父进程创建了一个子进程之后,父进程和子进程相同部分如下:

* 实际用户ID, 实际组ID, 有效用户ID, 有效组ID
* 附加组ID
* 进程组ID
* 会话ID
* 控制终端
* 设置用户ID标志和设置组ID标志
* 当前工作目录
* 根目录
* 文件模式创建屏蔽字
* 信号屏蔽和安排
* 针对任一打开文件描述符在执行时关闭(clone-on-exec)标志
* 环境
* 连接的共享存储段
* 存储映射
* 资源限制

_注意_: Linux内核中的如下几个概念

* 进程组

```
Shell 上的一条命令行形成一个进程组
每个进程属于一个进程组
每个进程组有一个领头进程
进程组的生命周期到组中最后一个进程终止, 或加入其他进程组为止
getpgrp: 获得进程组 id, 即领头进程的 pid
setpgid: 加入进程组和建立新的进程组
前台进程组和后台进程组

#include <unistd.h>

int setpgid (pid_t pid, pid_t pgid);
pid_t getpgid (pid_t pid);
int setpgrp (void);
pid_t getpgrp (void);

进程只能将自身和其子进程设置为进程组 id.
某个子进程调用exec函数之后, 就不能再将该子进程的id作为进程组 id.
```

* 会话

```
一次登录形成一个会话
一个会话可包含多个进程组, 但只能有一个前台进程组.
setsid 可建立一个新的会话

#include <unistd.h>

pid_t setsid(void);

如果调用进程不是进程组的领头进程, 该函数才能建立新的会话.
调用setsid之后, 进程成为新会话的领头进程.
进程成为新进程组的领头进程.
进程失去控制终端
```

* 控制终端

```
会话的领头进程打开一个终端之后, 该终端就成为该会话的控制终端 (SVR4/Linux)
与控制终端建立连接的会话领头进程称为控制进程 (session leader)
一个会话只能有一个控制终端
产生在控制终端上的输入和信号将发送给会话的前台进程组中的所有进程
终端上的连接断开时 (比如网络断开或 Modem 断开), 挂起信号将发送到控制进程(session leader)
```

#### 父进程和子进程之间的区别:
* fork的返回值
* 进程ID不同
* 两个进程具有不同进程ID.
* 子进程的tms_utime, tms_stime, tms_cutime和tms_ustime均被设置为0
* 父进程设置的文件锁不会被子进程继承
* 子进程的为处理闹钟(alarm)被清除
* 子进程为处理信号集设置为空集