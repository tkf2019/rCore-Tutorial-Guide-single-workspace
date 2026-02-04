内核态的线程管理
=========================================

线程概念
---------------------------------------------

这里会结合与进程的比较来说明线程的概念。到本章之前，我们看到了进程这一抽象，操作系统让进程拥有相互隔离的虚拟的地址空间，
让进程感到在独占一个虚拟的处理器。其实这只是操作系统通过时分复用和空分复用技术来让每个进程复用有限的物理内存和物理CPU。
而线程是在进程内中的一个新的抽象。在没有线程之前，一个进程在一个时刻只有一个执行点（即程序计数器 (PC)
寄存器保存的要执行指令的指针）。但线程的引入把进程内的这个单一执行点给扩展为多个执行点，即在进程中存在多个线程，
每个线程都有一个执行点。而且这些线程共享进程的地址空间，所以可以不必采用相对比较复杂的 IPC 机制（一般需要内核的介入），
而可以很方便地直接访问进程内的数据。

在线程的具体运行过程中，需要有程序计数器寄存器来记录当前的执行位置，需要有一组通用寄存器记录当前的指令的操作数据，
需要有一个栈来保存线程执行过程的函数调用栈和局部变量等，这就形成了线程上下文的主体部分。
这样如果两个线程运行在一个处理器上，就需要采用类似两个进程运行在一个处理器上的调度/切换管理机制，
即需要在一定时刻进行线程切换，并进行线程上下文的保存与恢复。这样在一个进程中的多线程可以独立运行，
取代了进程，成为操作系统调度的基本单位。

由于把进程的结构进行了细化，通过线程来表示对处理器的虚拟化，使得进程成为了管理线程的容器。
在进程中的线程没有父子关系，大家都是兄弟，但还是有个老大。这个代表老大的线程其实就是创建进程（比如通过
``fork`` 系统调用创建进程）时，建立的第一个线程，它的线程标识符（TID）为 ``0`` 。


线程模型与重要系统调用
----------------------------------------------

目前，我们只介绍本章实现的内核中采用的一种非常简单的线程模型。这个线程模型有三个运行状态：
就绪态、运行态和等待态；共享所属进程的地址空间和其他共享资源（如文件等）；可被操作系统调度来分时占用CPU执行；
可以动态创建和退出；可通过系统调用获得操作系统的服务。我们实现的线程模型建立在进程的地址空间抽象之上：
每个线程都共享进程的代码段和和可共享的地址空间（如全局数据段、堆等），但有自己的独占的栈。
线程模型需要操作系统支持一些重要的系统调用：创建线程、等待子线程结束等，来支持灵活的多线程应用。
接下来会介绍这些系统调用的基本功能和设计思路。


线程创建系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在一个进程的运行过程中，进程可以创建多个属于这个进程的线程，每个线程有自己的线程标识符（TID，Thread Identifier）。
系统调用 ``thread_create`` 的原型如下：

.. code-block:: rust
   :linenos:

   /// 功能：当前进程创建一个新的线程
   /// 参数：entry 表示线程的入口函数地址
   /// 参数：arg：表示线程的一个参数
   pub fn thread_create(entry: usize, arg: usize) -> isize

当进程调用 ``thread_create`` 系统调用后，内核会在这个进程内部创建一个新的线程，这个线程能够访问到进程所拥有的代码段，
堆和其他数据段。但内核会给这个新线程分配一个它专有的用户态栈，这样每个线程才能相对独立地被调度和执行。
在本教程的组件化实现中，线程切换与特权级切换由 ``tg-kernel-context`` 提供的“异界传送门”
（``MultislotPortal``）完成：每个线程的执行上下文中都包含它所属用户地址空间的页表根（``satp``），
内核在切换线程时根据该 ``satp`` 进入对应的用户地址空间运行；当发生 ``ecall``/异常时，会返回到内核继续处理。
为了让“进入/返回”过程在不同地址空间之间保持一致，内核会把传送门映射到每个用户地址空间的固定虚页位置
（见 ``ch8/src/main.rs`` 中的 ``PROTAL_TRANSIT`` 与 ``map_portal``）。

相比于创建进程的 ``fork`` 系统调用，创建线程不需要要建立新的地址空间，这是二者之间最大的不同。
另外属于同一进程中的线程之间没有父子关系，这一点也与进程不一样。

等待子线程系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当一个线程执行完代表它的功能后，会通过 ``exit`` 系统调用退出。内核在收到线程发出的 ``exit`` 系统调用后，
会结束该线程的调度实体，并记录退出码。此后，进程内的其他线程可以通过 ``waittid`` 等待该线程结束并获取其退出码。
系统调用 ``waittid`` 的原型如下：

.. code-block:: rust
   :linenos:

   /// 参数：tid表示线程id
   /// 返回值：如果线程不存在，返回-1；如果线程还没退出，返回-2；其他情况下，返回结束线程的退出码
   pub fn waittid(tid: usize) -> isize


一般情况下，创建线程的那条线程会通过 ``waittid`` 等待子线程结束并获取退出码。
在本章的实现中，线程退出只会结束当前线程；当一个进程内 **最后一个线程** 退出时，该进程才会被内核删除。
进程级别的资源回收仍由 ``waitpid``（``wait``/``waitpid`` 系列系统调用）完成。


进程相关的系统调用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在引入了线程机制后，进程相关的重要系统调用： ``fork`` 、 ``exec`` 、 ``waitpid`` 虽然在接口上没有变化，
但在它要完成的功能上需要有一定的扩展。首先，需要注意到把以前进程中与处理器执行相关的部分拆分到线程中。这样，在通过
``fork`` 创建进程其实也意味着要单独建立一个主线程来使用处理器，并为以后创建新的线程建立相应的线程控制块向量。
相对而言， ``exec`` 和 ``waitpid`` 这两个系统调用要做的改动比较小，还是按照与之前进程的处理方式来进行。总体上看，
进程相关的这三个系统调用还是保持了已有的进程操作的语义，并没有由于引入了线程，而带来大的变化。


应用程序示例
----------------------------------------------

我们刚刚介绍了 thread_create/waittid 两个重要系统调用，我们可以借助它们和之前实现的系统调用，
开发出功能更为灵活的应用程序。下面我们通过描述一个多线程应用程序 ``threads`` 的开发过程来展示这些系统调用的使用方法。


系统调用封装
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

本教程将系统调用的“用户态封装”放在 ``tg-syscall`` 组件中（``tg-syscall/src/user.rs``）。
其中 ``waittid`` 的封装方式与第三章的 ``waitpid`` 类似：当内核返回 ``-2``（目标线程仍在运行）时，
用户态主动 ``sched_yield`` 让出 CPU，避免忙等占用处理器。

.. code-block:: rust
   :linenos:

   pub fn waittid(tid: usize) -> isize {
       loop {
           match unsafe { syscall1(SyscallId::WAITID, tid) } {
               -2 => { sched_yield(); }
               exit_code => return exit_code,
           }
       }
   }

waittid 等待一个线程标识符的值为 tid 的线程结束。当返回值为 ``-2`` 时表示“线程存在但尚未退出”，
此时通过 ``sched_yield`` 让出 CPU，下一次被调度运行时再重试。


多线程应用程序 -- threads
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

多线程应用程序 -- threads 开始执行后，先调用 ``thread_create`` 创建了三个线程，加上进程自带的主线程，其实一共有四个线程。每个线程在打印了1000个字符后，会执行 ``exit`` 退出。进程通过 ``waittid`` 等待这三个线程结束后，最终结束进程的执行。下面是多线程应用程序 -- threads 的源代码：

.. code-block:: rust
   :linenos:

   // tg-user/src/bin/threads.rs

   #![no_std]
   #![no_main]

   #[macro_use]
   extern crate user_lib;
   extern crate alloc;

   use user_lib::{exit, thread_create, waittid};

   pub fn thread_a() -> isize {
       for _ in 0..1000 { print!("a"); }
       exit(1)
   }

   pub fn thread_b() -> isize {
       for _ in 0..1000 { print!("b"); }
       exit(2)
   }

   pub fn thread_c() -> isize {
       for _ in 0..1000 { print!("c"); }
       exit(3)
   }

   #[no_mangle]
   pub extern "C" fn main() -> i32 {
       let tids = [
           thread_create(thread_a as *const () as usize, 0),
           thread_create(thread_b as *const () as usize, 0),
           thread_create(thread_c as *const () as usize, 0),
       ];
       for tid in tids.iter() {
           let exit_code = waittid(*tid as usize);
           println!("thread#{} exited with code {}", tid, exit_code);
       }
       println!("main thread exited.");
       0
   }

线程管理的核心数据结构
-----------------------------------------------

本章在内核实现中明确区分两类对象：

- **进程（Process）**：资源容器，管理同一进程内各线程共享的资源（地址空间、文件描述符表、信号模块、同步原语等）；
- **线程（Thread）**：执行实体，管理执行上下文（寄存器上下文 + ``satp``）与线程标识符（TID）。

它们位于 ``ch8/src/process.rs``：

.. code-block:: rust
   :linenos:

   // ch8/src/process.rs
   pub struct Process {
       pub pid: ProcId,
       pub address_space: AddressSpace<Sv39, Sv39Manager>,
       pub fd_table: Vec<Option<Mutex<Fd>>>,
       pub signal: Box<dyn Signal>,
       pub semaphore_list: Vec<Option<Arc<Semaphore>>>,
       pub mutex_list: Vec<Option<Arc<dyn MutexTrait>>>,
       pub condvar_list: Vec<Option<Arc<Condvar>>>,
   }

   pub struct Thread {
       pub tid: ThreadId,
       pub context: ForeignContext,
   }

可以看到：同步原语列表（mutex/semaphore/condvar）归属于进程，符合“同一进程内线程共享资源”的直觉；而线程仅保存
TID 与执行上下文。

处理器与调度器
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

本章使用 ``tg-task-manage`` 组件提供的线程/进程管理框架 ``PThreadManager`` 来维护：

- 线程就绪队列：选择下一条要运行的线程；
- 线程-进程从属关系（``tid -> pid`` 映射）；
- 进程父子关系与 wait/waittid 所需的退出码记录。

在本章内核中，``ch8/src/processor.rs`` 将其具体化为：

.. code-block:: rust
   :linenos:

   // ch8/src/processor.rs
   pub type ProcessorInner = PThreadManager<Process, Thread, ThreadManager, ProcManager>;
   pub static PROCESSOR: Processor = Processor::new();

并实现两个“后端”管理器：

- **ThreadManager**：用 ``BTreeMap<ThreadId, Thread>`` 保存全部线程实体，用 ``VecDeque<ThreadId>`` 作为就绪队列；
- **ProcManager**：用 ``BTreeMap<ProcId, Process>`` 保存全部进程实体。

线程管理机制的设计与实现
-----------------------------------------------

本章的线程管理机制由两部分协作完成：

- **系统调用抽象与分发**：由 ``tg-syscall`` 提供（traits + ``handle`` 分发器）；
- **线程/进程与 wait 关系维护**：由 ``tg-task-manage`` 提供（``PThreadManager`` + 关系表）。

内核启动时会在 ``ch8/src/main.rs`` 中初始化并注册系统调用实现：

.. code-block:: rust
   :linenos:

   // ch8/src/main.rs（节选）
   tg_syscall::init_process(&SyscallContext);
   tg_syscall::init_scheduling(&SyscallContext);
   tg_syscall::init_clock(&SyscallContext);
   tg_syscall::init_signal(&SyscallContext);
   tg_syscall::init_thread(&SyscallContext);
   tg_syscall::init_sync_mutex(&SyscallContext);

在每次用户态触发 ``ecall`` 返回内核后，内核读取寄存器得到 syscall id 与参数，然后调用
``tg_syscall::handle(...)`` 完成分发。

线程创建
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``thread_create`` 的内核实现位于 ``ch8/src/main.rs`` 中 ``SyscallContext`` 对 ``tg_syscall::Thread`` trait
的实现。其核心工作可以概括为：

- 为新线程找到一段尚未映射的用户栈虚拟地址区间；
- 分配物理页并映射进当前进程地址空间；
- 构造用户态初始上下文（入口点、栈指针、参数寄存器）；
- 通过 ``PROCESSOR``（``PThreadManager``）把新线程加入当前进程，并进入就绪队列等待调度。

.. code-block:: rust
   :linenos:

   // ch8/src/main.rs（节选）
   fn thread_create(&self, _caller: Caller, entry: usize, arg: usize) -> isize {
       let processor: *mut ProcessorInner = PROCESSOR.get_mut() as *mut ProcessorInner;
       let current_proc = unsafe { (*processor).get_current_proc().unwrap() };
       // 省略：扫描虚拟地址空间，找到未映射的栈位置 vpn
       // 省略：分配两页并 map_extern 到用户地址空间（U_WRV）
       let satp = (8 << 60) | current_proc.address_space.root_ppn().val();
       let mut context = tg_kernel_context::LocalContext::user(entry);
       *context.sp_mut() = /* 新栈顶 */;
       *context.a_mut(0) = arg;
       let thread = Thread::new(satp, context);
       let tid = thread.tid;
       unsafe { (*processor).add(tid, thread, current_proc.pid) };
       tid.get_usize() as _
   }

线程退出
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

线程通过 ``exit`` 系统调用退出。内核在处理 ``SyscallId::EXIT`` 时，会调用
``PThreadManager::make_current_exited(exit_code)`` 将当前线程从调度器中删除，并在关系表中记录退出码；
若该线程是进程内最后一个线程，则同时删除进程并维护父子关系（子进程会被转移到 0 号进程名下）。

等待线程结束
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``waittid`` 依赖 ``tg-task-manage`` 的关系表（``ProcThreadRel``）维护退出线程队列 ``dead_threads``：

- 若目标线程已退出：返回其退出码；
- 若目标线程仍在运行：返回 ``-2``（用户态会 ``sched_yield`` 并重试）；
- 若目标线程不存在：返回 ``-1``。

对应的内核实现位于 ``SyscallContext::waittid``（``ch8/src/main.rs``），其本质是调用
``PThreadManager::waittid(ThreadId)`` 查询关系表。


线程执行中的特权级切换和调度切换
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

本章的“进入用户态执行/从用户态返回内核”由 ``tg-kernel-context`` 的 ``ForeignContext`` 与
``MultislotPortal`` 协作完成：内核调度到某个线程时，调用
``task.context.execute(portal, ())`` 进入该线程的用户态上下文；当发生 ``ecall``/异常时返回到内核，
内核在 ``rust_main`` 的调度循环中根据返回原因（如 ``UserEnvCall``）完成系统调用分发，并根据结果将当前线程
设置为 **就绪（suspend，重新入队）** / **阻塞（blocked，不入队）** / **退出（exited，删除实体）**。


