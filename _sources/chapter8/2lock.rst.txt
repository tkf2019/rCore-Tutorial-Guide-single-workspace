锁机制
=========================================

本节导读
-----------------------------------------

.. chyyuu https://en.wikipedia.org/wiki/Lock_(computer_science)

到目前为止，我们已经实现了进程和线程，也能够理解在一个时间段内，会有多个线程在执行，这就是并发。
而且，由于线程的引入，多个线程可以共享进程中的全局数据。如果多个线程都想读和更新全局数据，
那么谁先更新取决于操作系统内核的抢占式调度和分派策略。在一般情况下，每个线程都有可能先执行，
且可能由于中断等因素，随时被操作系统打断其执行，而切换到另外一个线程运行，
形成在一段时间内，多个线程交替执行的现象。如果没有一些保障机制（比如互斥、同步等），
那么这些对共享数据进行读写的交替执行的线程，其期望的共享数据的正确结果可能无法达到。

所以，我们需要研究一种保障机制 --- 锁 ，确保无论操作系统如何抢占线程，调度和切换线程的执行，
都可以保证对拥有锁的线程，可以独占地对共享数据进行读写，从而能够得到正确的共享数据结果。
这种机制的能力来自于处理器的指令、操作系统系统调用的基本支持，从而能够保证线程间互斥地读写共享数据。
下面各个小节将从为什么需要锁、锁的基本思路、锁的不同实现方式等逐步展开讲解。

为什么需要锁
-----------------------------------------

上一小节已经提到，没有保障机制的多个线程，在对共享数据进行读写的过程中，可能得不到预期的结果。
我们来看看这个简单的例子：

.. code-block:: c
    :linenos:
    :emphasize-lines: 4

    // 线程的入口函数
    int a=0;
    void f() {
      a = a + 1;
    }

对于上述函数中的第 4 行代码，一般人理解处理器会一次就执行完这条简单的语句，但实际情况并不是这样。
我们可以用 GCC 编译出上述函数的汇编码：

.. code-block:: shell
    :linenos:

    $ riscv64-unknown-elf-gcc -o f.s -S f.c


可以看到生成的汇编代码如下：

.. code-block:: asm
    :linenos:
    :emphasize-lines: 18-23

    //f.s
      .text
      .globl	a
      .section	.sbss,"aw",@nobits
      .align	2
      .type	a, @object
      .size	a, 4
    a:
      .zero	4
      .text
      .align	1
      .globl	f
      .type	f, @function
    f:
      addi	sp,sp,-16
      sd	s0,8(sp)
      addi	s0,sp,16
      lui	a5,%hi(a)
      lw	a5,%lo(a)(a5)
      addiw	a5,a5,1
      sext.w	a4,a5
      lui	a5,%hi(a)
      sw	a4,%lo(a)(a5)
      nop
      ld	s0,8(sp)
      addi	sp,sp,16
      jr	ra


.. chyyuu 可以给上面的汇编码添加注释???

从中可以看出，对于高级语言的一条简单语句（C 代码的第 4 行，对全局变量进行读写），很可能是由多条汇编代码
（汇编代码的第 18~23 行）组成。如果这个函数是多个线程要执行的函数，那么在上述汇编代码第
18 行到第 23 行中的各行之间，可能会发生中断，从而导致操作系统执行抢占式的线程调度和切换，
就会得到不一样的结果。由于执行这段汇编代码（第 18~23 行））的多个线程在访问全局变量过程中可能导致竞争状态，
因此我们将此段代码称为临界区（critical section）。临界区是访问共享变量（或共享资源）的代码片段，
不能由多个线程同时执行，即需要保证互斥。

下面是有两个线程T0、T1在一个时间段内的一种可能的执行情况：

=====  =====  =======   =======   ===========   =========
时间     T0     T1        OS       共享变量a    寄存器a5
=====  =====  =======   =======   ===========   =========
1       L18      --       --         0          a的高位地址
2       --      --      切换         0              0
3       --      L18       --         0          a的高位地址
4       L20      --       --         0              1
5       --      --      切换         0           a的高位地址
6       --      L20       --         0              1
7       --      --      切换         0              1
8       L23     --       --          1              1
9       --      --      切换         1              1
10      --      L23      --          1              1
=====  =====  =======   =======   ===========   =========

一般情况下，线程 T0 执行完毕后，再执行线程 T1，那么共享全局变量 ``a`` 的值为 2 。但在上面的执行过程中，
可以看到在线程执行指令的过程中会发生线程切换，这样在时刻 10 的时候，共享全局变量 ``a`` 的值为 1 ，
这不是我们预期的结果。出现这种情况的原因是两个线程在操作系统的调度下（在哪个时刻调度具有不确定性），
交错执行 ``a = a + 1`` 的不同汇编指令序列，导致虽然增加全局变量 ``a`` 的代码被执行了两次，
但结果还是只增加了 1 。这种多线程的最终执行结果不确定（indeterminate），取决于由于调度导致的、
不确定指令执行序列的情况就是竞态条件（race condition）。

如果每个线程在执行 ``a = a + 1`` 这个 C 语句所对应多条汇编语句过程中，不会被操作系统切换，
那么就不会出现多个线程交叉读写全局变量的情况，也就不会出现结果不确定的问题了。

所以，访问（特指写操作）共享变量代码片段，不能由多个线程同时执行（即并行）或者在一个时间段内都去执行
（即并发）。要做到这一点，需要互斥机制的保障。从某种角度上看，这种互斥性也是一种原子性，
即线程在临界区的执行过程中，不会出现只执行了一部分，就被打断并切换到其他线程执行的情况。即，
要么线程执行的这一系列操作/指令都完成，要么这一系列操作/指令都不做，不会出现指令序列执行中被打断的情况。


锁的基本思路
-----------------------------------------

要保证多线程并发执行中的临界区的代码具有互斥性或原子性，我们可以建立一种锁，
只有拿到锁的线程才能在临界区中执行。这里的锁与现实生活中的锁的含义很类似。比如，我们可以写出如下的伪代码：

.. code-block:: rust
    :linenos:

    lock(mutex);    // 尝试取锁
    a = a + 1;      // 临界区，访问临界资源 a
    unlock(mutex);  // 是否锁
    ...             // 剩余区

对于一个应用程序而言，它的执行是受到其执行环境的管理和限制的，而执行环境的主要组成就是用户态的系统库、
操作系统和更底层的处理器，这说明我们需要有硬件和操作系统来对互斥进行支持。一个自然的想法是，这个
``lock/unlock`` 互斥操作就是CPU提供的机器指令，那上面这一段程序就很容易在计算机上执行了。
但需要注意，这里互斥的对象是线程的临界区代码，而临界区代码可以访问各种共享变量（简称临界资源）。
只靠两条机器指令，难以识别各种共享变量，不太可能约束可能在临界区的各种指令执行共享变量操作的互斥性。
所以，我们还是需要有一些相对更灵活和复杂一点的方法，能够设置一种所有线程能看到的标记，
在一个能进入临界区的线程设置好这个标记后，其他线程都不能再进入临界区了。总体上看，
对临界区的访问过程分为四个部分：

1. 尝试取锁: 查看锁是否可用，即临界区是否可访问（看占用临界区标志是否被设置），如果可以访问，
   则设置占用临界区标志（锁不可用）并转到步骤 2 ，否则线程忙等或被阻塞;
2. 临界区: 访问临界资源的系列操作
3. 释放锁: 清除占用临界区标志（锁可用），如果有线程被阻塞，会唤醒阻塞线程；
4. 剩余区: 与临界区不相关部分的代码

根据上面的步骤，可以看到锁机制有两种：让线程忙等的忙等锁（spin lock），以及让线程阻塞的睡眠锁
（sleep lock）。锁的实现大体上基于三类机制：用户态软件、机器指令硬件、内核态操作系统。
下面我们介绍来 rCore 中基于内核态操作系统级方法实现的支持互斥的锁。

我们还需要知道如何评价各种锁实现的效果。一般我们需要关注锁的三种属性：

1. 互斥性（mutual exclusion），即锁是否能够有效阻止多个线程进入临界区，这是最基本的属性。
2. 公平性（fairness），当锁可用时，每个竞争线程是否有公平的机会抢到锁。
3. 性能（performance），即使用锁的时间开销。


内核态操作系统级方法实现锁 --- mutex 系统调用
----------------------------------------------


使用 mutex 系统调用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

如何能够实现轻量的可睡眠锁？一个自然的想法就是，让等待锁的线程睡眠，让释放锁的线程显式地唤醒等待锁的线程。
如果有多个等待锁的线程，可以全部释放，让大家再次竞争锁；也可以只释放最早等待的那个线程。
这就需要更多的操作系统支持，特别是需要一个等待队列来保存等待锁的线程。

我们先看看多线程应用程序如何使用mutex系统调用的：


.. code-block:: rust
   :linenos:
   :emphasize-lines: 8,15,21

   // tg-user/src/bin/race_adder_mutex_blocking.rs（节选）
   static mut A: usize = 0;

   unsafe fn f() -> isize {
       for _ in 0..PER_THREAD {
           mutex_lock(0);
           // 临界区：读 A -> 计算 -> 写 A
           // ...
           mutex_unlock(0);
       }
       exit(0)
   }

   pub extern "C" fn main() -> i32 {
       assert_eq!(mutex_create(true), 0); // 创建 ID 为 0 的阻塞互斥锁
       for _ in 0..THREAD_COUNT {
           thread_create(f as *const () as usize, 0);
       }
       // ...
       0
   }

在用户态看来，这是一组非常直接的接口（``mutex_create/mutex_lock/mutex_unlock``），它们由 ``tg-syscall``
组件在 ``tg-syscall/src/user.rs`` 中提供封装，底层通过 ``ecall`` 进入内核。

mutex 系统调用的实现
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

本章将 mutex 的“数据结构与互斥逻辑”下沉到组件 ``tg-sync``，而把 **阻塞/唤醒与调度** 放在 ``ch8`` 内核中完成。
这也是组件化实现里非常重要的一条分层原则：同步原语只维护队列/状态，不直接操作调度器。

核心数据结构
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在进程结构 ``Process`` 中维护互斥锁列表（同一进程内线程共享）：

.. code-block:: rust
   :linenos:

   // ch8/src/process.rs（节选）
   pub struct Process {
       // ...
       pub mutex_list: Vec<Option<Arc<dyn MutexTrait>>>,
   }

``tg-sync`` 中定义了 mutex 抽象与阻塞 mutex 的实现（等待队列里保存的是 ``ThreadId``）：

.. code-block:: rust
   :linenos:

   // tg-sync/src/mutex.rs（节选）
   pub trait Mutex: Sync + Send {
       fn lock(&self, tid: ThreadId) -> bool;
       fn unlock(&self) -> Option<ThreadId>;
   }

   pub struct MutexBlockingInner {
       locked: bool,
       wait_queue: VecDeque<ThreadId>,
   }

系统调用与阻塞/唤醒
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

内核通过实现 ``tg_syscall::SyncMutex`` trait 提供 mutex 相关系统调用（位于 ``ch8/src/main.rs``）。
其中的关键点有两条：

- **加锁失败**：``MutexBlocking::lock(tid)`` 会把当前线程 TID 入队并返回 ``false``；内核将系统调用返回值设为 ``-1``，
  并在 syscall 分发后把当前线程标记为 blocked（不再加入就绪队列），实现“睡眠等待”。
- **解锁唤醒**：``MutexBlocking::unlock()`` 会从等待队列中弹出一个 TID（若存在）并返回给内核；内核据此把该线程重新加入就绪队列。

.. code-block:: rust
   :linenos:

   // ch8/src/main.rs（节选）
   fn mutex_lock(&self, _caller: Caller, mutex_id: usize) -> isize {
       let processor: *mut ProcessorInner = PROCESSOR.get_mut() as *mut ProcessorInner;
       let tid = unsafe { (*processor).current().unwrap() }.tid;
       let current_proc = unsafe { (*processor).get_current_proc().unwrap() };
       let mutex = Arc::clone(current_proc.mutex_list[mutex_id].as_ref().unwrap());
       if !mutex.lock(tid) { -1 } else { 0 }
   }

   fn mutex_unlock(&self, _caller: Caller, mutex_id: usize) -> isize {
       let processor: *mut ProcessorInner = PROCESSOR.get_mut() as *mut ProcessorInner;
       let current_proc = unsafe { (*processor).get_current_proc().unwrap() };
       let mutex = Arc::clone(current_proc.mutex_list[mutex_id].as_ref().unwrap());
       if let Some(waking_tid) = mutex.unlock() {
           unsafe { (*processor).re_enque(waking_tid) };
       }
       0
   }

最后，在 ``rust_main`` 的 syscall 统一分发位置，内核会对 ``MUTEX_LOCK`` 这类“可能阻塞”的系统调用做统一处理：
返回值为 ``-1`` 表示获取失败，需要阻塞当前线程；否则将当前线程重新入队（suspend），等待下一轮调度继续执行。


