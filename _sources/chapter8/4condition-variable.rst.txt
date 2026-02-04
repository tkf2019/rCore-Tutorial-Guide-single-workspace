条件变量机制
=========================================

本节导读
-----------------------------------------

到目前为止，我们已经了解了操作系统提供的互斥锁和信号量。但应用程序在使用这两者时需要非常小心，
如果使用不当，就会产生效率低下、竞态条件、死锁或者其他一些不可预测的情况。为了简化编程、避免错误，
计算机科学家针对某些情况设计了一种更高层的同步互斥原语。具体而言，在有些情况下，
线程需要检查某一条件（condition）满足之后，才会继续执行。

我们来看一个例子，有两个线程 first 和 second 在运行，线程 first 会把全局变量 A 设置为
1，而线程 second 在 ``A != 0`` 的条件满足后，才能继续执行，如下面的伪代码所示：

.. code-block:: rust
    :linenos:

    static mut A: usize = 0;
    unsafe fn first() -> ! {
        A=1;
        ...
    }

    unsafe fn second() -> ! {
        while A==0 {
          // 忙等或睡眠等待 A==1
        };
        //继续执行相关事务
    }

在上面的例子中，如果线程 second 先执行，会忙等在 while 循环中，在操作系统的调度下，线程
first 会执行并把 A 赋值为 1 后，然后线程 second 再次执行时，就会跳出 while 循环，进行接下来的工作。
配合互斥锁，可以正确完成上述带条件的同步流程，如下面的伪代码所示：

.. code-block:: rust
    :linenos:

    static mut A: usize = 0;
    unsafe fn first() -> ! {
        mutex.lock();
        A=1;
        mutex.unlock();
        ...
    }

    unsafe fn second() -> ! {
        mutex.lock();
        while A==0 {
            mutex.unlock();
            // give other thread a chance to lock
            mutex.lock();
        };
        mutex.unlock();
        //继续执行相关事务
    }

这种实现能执行，但效率低下，因为线程 second 会忙等检查，浪费处理器时间。我们希望有某种方式让线程
second 休眠，直到等待的条件满足，再继续执行。于是，我们可以写出如下的代码：

.. code-block:: rust
    :linenos:

    static mut A: usize = 0;
    unsafe fn first() -> ! {
        mutex.lock();
        A=1;
        wakup(second);
        mutex.unlock();
        ...
    }

    unsafe fn second() -> ! {
        mutex.lock();
        while A==0 {
           wait();
        };
        mutex.unlock();
        //继续执行相关事务
    }

粗略地看，这样就可以实现睡眠等待了。但请同学仔细想想，当线程 second 在睡眠的时候， ``mutex``
是否已经上锁了？ 确实，线程 second 是带着上锁的 ``mutex`` 进入等待睡眠状态的。
如果这两个线程的调度顺序是先执行线程 second，再执行线程first，那么线程 second 会先睡眠且拥有
``mutex`` 的锁；当线程 first 执行时，会由于没有 ``mutex`` 的锁而进入等待锁的睡眠状态。
结果就是两个线程都睡了，都执行不下去，这就出现了 **死锁** 。

这里需要解决的两个关键问题： **如何等待一个条件？** 和 **在条件为真时如何向等待线程发出信号** 。
我们的计算机科学家给出了 **管程（Monitor）** 和 **条件变量（Condition Variables）**
这种巧妙的方法。接下来，我们就会深入讲解条件变量的设计与实现。

条件变量的基本思路
-------------------------------------------

管程有一个很重要的特性，即任一时刻只能有一个活跃线程调用管程中的过程，
这一特性使线程在调用执行管程中过程时能保证互斥，这样线程就可以放心地访问共享变量。
管程是编程语言的组成部分，编译器知道其特殊性，因此可以采用与其他过程调用不同的方法来处理对管程的调用.
因为是由编译器而非程序员来生成互斥相关的代码，所以出错的可能性要小。

管程虽然借助编译器提供了一种实现互斥的简便途径，但这还不够，还需要一种线程间的沟通机制。
首先是等待机制：由于线程在调用管程中某个过程时，发现某个条件不满足，那就在无法继续运行而被阻塞。
其次是唤醒机制：另外一个线程可以在调用管程的过程中，把某个条件设置为真，并且还需要有一种机制，
及时唤醒等待条件为真的阻塞线程。为了避免管程中同时有两个活跃线程，
我们需要一定的规则来约定线程发出唤醒操作的行为。目前有三种典型的规则方案：

- Hoare 语义：线程发出唤醒操作后，马上阻塞自己，让新被唤醒的线程运行。注：此时唤醒线程的执行位置还在管程中。
- Hansen 语义：是执行唤醒操作的线程必须立即退出管程，即唤醒操作只可能作为一个管程过程的最后一条语句。
  注：此时唤醒线程的执行位置离开了管程。
- Mesa 语义：唤醒线程在发出行唤醒操作后继续运行，并且只有它退出管程之后，才允许等待的线程开始运行。
  注：此时唤醒线程的执行位置还在管程中。

下面介绍一个基于 Mesa 语义的沟通机制。这种沟通机制的具体实现就是
**条件变量** 和对应的操作：wait 和 signal。线程使用条件变量来等待一个条件变成真。
条件变量其实是一个线程等待队列，当条件不满足时，线程通过执行条件变量的 wait
操作就可以把自己加入到等待队列中，睡眠等待（waiting）该条件。另外某个线程，当它改变条件为真后，
就可以通过条件变量的 signal 操作来唤醒一个或者多个等待的线程（通过在该条件上发信号），让它们继续执行。

早期提出的管程是基于 Concurrent Pascal 来设计的，其他语言如 C 和 Rust 等，并没有在语言上支持这种机制。
我们还是可以用手动加入互斥锁的方式来代替编译器，就可以在 C 和 Rust 的基础上实现原始的管程机制了。
在目前的 C 语言应用开发中，实际上也是这么做的。这样，我们就可以用互斥锁和条件变量，
来重现上述的同步互斥例子：

.. code-block:: rust
    :linenos:

    static mut A: usize = 0;
    unsafe fn first() -> ! {
        mutex.lock();
        A=1;
        condvar.wakup();
        mutex.unlock();
        ...
    }

    unsafe fn second() -> ! {
        mutex.lock();
        while A==0 {
           condvar.wait(mutex); //在睡眠等待之前，需要释放mutex
        };
        mutex.unlock();
        //继续执行相关事务
    }

有了上面的介绍，我们就可以实现条件变量的基本逻辑了。下面是条件变量的 wait 和 signal 操作的伪代码：

.. code-block:: rust
    :linenos:

    fn wait(mutex) {
        mutex.unlock();
        <block and enqueue the thread>;
        mutex.lock();
    }

    fn signal() {
        <unblock a thread>;
    }

条件变量的wait操作包含三步，1. 释放锁；2. 把自己挂起；3. 被唤醒后，再获取锁。条件变量的 signal
操作只包含一步：找到挂在条件变量上睡眠的线程，把它唤醒。

注意，条件变量不像信号量那样有一个整型计数值的成员变量，所以条件变量也不能像信号量那样有读写计数值的能力。
如果一个线程向一个条件变量发送唤醒操作，但是在该条件变量上并没有等待的线程，则唤醒操作实际上什么也没做。

实现条件变量
-------------------------------------------

使用 condvar 系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

我们通过例子来看看如何实际使用条件变量。下面是面向应用程序对条件变量系统调用的简单使用，
可以看到对它的使用与上一节介绍的信号量系统调用类似。 在这个例子中，主线程先创建了初值为 1
的互斥锁和一个条件变量，然后再创建两个线程 First 和 Second。线程 First 会先睡眠 10ms，而当线程
Second 执行时，会由于条件不满足执行条件变量的 wait 操作而等待睡眠；当线程 First 醒来后，通过设置
A 为 1，让线程 second 等待的条件满足，然后会执行条件变量的 signal 操作，从而能够唤醒线程 Second。
这样线程 First 和线程 Second 就形成了一种稳定的同步与互斥关系。

.. code-block:: rust
    :linenos:

    // tg-user/src/bin/test_condvar.rs（节选）
    static mut A: usize = 0;
    const CONDVAR_ID: usize = 0;
    const MUTEX_ID: usize = 0;

    unsafe fn first() -> isize {
        sleep(10);
        println!("First work, Change A --> 1 and wakeup Second");
        mutex_lock(MUTEX_ID);
        A = 1;
        condvar_signal(CONDVAR_ID);
        mutex_unlock(MUTEX_ID);
        exit(0)
    }

    unsafe fn second() -> isize {
        println!("Second want to continue,but need to wait A=1");
        mutex_lock(MUTEX_ID);
        while (&raw const A).read_volatile() == 0 {
            condvar_wait(CONDVAR_ID, MUTEX_ID);
        }
        mutex_unlock(MUTEX_ID);
        exit(0)
    }

    pub extern "C" fn main() -> i32 {
        assert_eq!(condvar_create() as usize, CONDVAR_ID);
        assert_eq!(mutex_create(true) as usize, MUTEX_ID);
        // create threads ...
        0
    }

其中 ``condvar_*`` 接口由 ``tg-syscall`` 在 ``tg-syscall/src/user.rs`` 中提供封装，底层通过 ``ecall`` 进入内核。


实现 condvar 系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

本章同样遵循组件化分层：条件变量的数据结构由 ``tg-sync`` 提供，而“阻塞/唤醒/调度”由内核统一处理。

核心数据结构
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

条件变量列表归属于进程（同一进程内线程共享）：

.. code-block:: rust
   :linenos:

   // ch8/src/process.rs（节选）
   pub struct Process {
       // ...
       pub condvar_list: Vec<Option<Arc<Condvar>>>,
   }

``tg-sync`` 的 ``Condvar`` 内部维护等待队列（保存 ``ThreadId``）：

.. code-block:: rust
   :linenos:

   // tg-sync/src/condvar.rs（节选）
   pub struct CondvarInner {
       pub wait_queue: VecDeque<ThreadId>,
   }

   impl Condvar {
       pub fn new() -> Self { /* ... */ }
       pub fn signal(&self) -> Option<ThreadId> { /* pop_front */ }
   }

系统调用与实现现状
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

内核通过实现 ``tg_syscall::SyncMutex`` trait 提供 ``condvar_*`` 系列系统调用（位于 ``ch8/src/main.rs``）：

- ``condvar_create``：在当前进程的 ``condvar_list`` 中分配一个新的 ``Condvar``；
- ``condvar_signal``：调用 ``condvar.signal()``，若返回某个等待线程的 TID，则 ``PROCESSOR.re_enque(tid)`` 重新入队；
- ``condvar_wait``：当前实现使用 ``tg-sync`` 中的一个 **简化版** 辅助函数 ``wait_with_mutex``，
  其行为是“先释放 mutex 并唤醒一个等待该 mutex 的线程，然后当前线程立刻尝试重新获取 mutex”。

.. note::

   上述 ``wait_with_mutex`` 并未完整实现本节前面介绍的 Mesa/Hoare 等条件变量语义（例如：把线程真正挂到 condvar 的等待队列并睡眠，
   以及被 signal 唤醒后再竞争 mutex）。它是为了便于通过教学测例而提供的最小实现；后续若要实现更接近真实 OS 的 condvar，
   应当让 ``condvar_wait`` 将线程加入 ``CondvarInner.wait_queue`` 并阻塞，``condvar_signal`` 再把对应线程唤醒回就绪队列。

Hansen, Per Brinch (1993). "Monitors and concurrent Pascal: a personal history". HOPL-II:
The second ACM SIGPLAN conference on History of programming languages. History of Programming
Languages. New York, NY, USA: ACM. pp. 1–35. doi:10.1145/155360.155361. ISBN 0-89791-570-4.