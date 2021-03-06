锁与进程间通信
=======


`Linux`作为多任务系统, 能够同时运行几个进程. 通常, 各个进程必须尽可能保持独立, 避免彼此干扰. 这对于保护数据和确保系统稳定性都很有必要. 但有时候, 应用程序必须彼此通信.

举例来说 :

*	一个进程生成的数据传输到另一个进程时;

*	数据由多个进程共享时;

*	进程必须彼此等待时;

*	需要协调资源的使用时

我们可以使用`System V`引入的几种经典技术来处理这些情况, 这些技术证明了自身的价值, 现在已经是`Linux`的主要部分了. 用户空间应用程序和内核自身都面临此类情况, 特别是在多处理器系统上, 需要各种内核内部的机制进行处理.

如果几个进程共享一个资源, 则很容易彼此干扰, 必须防止这种情况. 因此内核不仅提供了共享数据的机制, 同样提供了协调对数据访问的机制. 内核仍然采用了来自`System V`的机制.

用户空间应用程序和内核自身都需要保护资源, 特别是后者. 在`SMP`系统上, 各个`CPU`可能同时处于核心态, 在理论上可以操作所有现存的数据结构. 为阻止CPU彼此干扰, 需要通过锁保护内核的某些范围. 锁可以确保每次只能有一个`CPU`访问被保护的范围.



#1	控制机制
-------



在讲述内核的各种进程间通信(`interprocess communication`, `IPC`)和数据同步机制之前, 我们简
单讨论一下相互通信的进程彼此干扰的可能情况, 以及如何防止. 我们的讨论只限于基本和核心的方面. 对于经典问题的详细解释和大量例子, 请参考市面上操作系统方面的通用教科书.


##1.1	竞态条件
-------

我们考虑系统通过两种接口从外部设备读取数据的情况. 独立的数据包以不定间隔通过两个接口到达, 存在不同文件中. 录数据包到达的次序, 文件名之后添加了一个号码, 明数据包的
序号. 通常的一系列文件名是`act1.fil`、`act2.fil`、`act3.fil`, 等等. 可使用一个独立的变量来简化两个进程的工作. 该变量保存在由两个进程共享的内存页中, 且指定了下一个未使用的序号(为简
明起见,在下文我称该变量为 counter).

在一个数据包到达时, 进程必须执行一些操作, 才能正确地保存数据.

1.	从接口读取数据

2.	用序号counter构造文件名,打开一个文件

3.	将序号加1

4.	将数据写入文件,然后关闭文件。

上述的软件系统会发生错误吗? 如果每个进程都严格遵守上述过程, 并在适当的位置对状态变量加1, 那么上述过程不仅适用于两个进程, 也可用于多个进程.

事实上, 大多数情况下上述过程都会正确运作, 但在某些情况下会出错, 而这也是分布式程序设计的真正困难所在. 我们设个陷阱, 分别将从接口读取数据的进程称作进程1和进程2.

我们给出的场景中, 已经保存了若干文件, 比如说总共有12文件. 因此`counter`的值是13. 显然是个"凶兆"......

进程1从接口接收一个刚到达的新数据块. 它忠实地用序号13构造文件名并打开一个文件, 而同时调度器被激活并确认该进程已经消耗了足够的`CPU`时间, 必须由另一个进程替换, 并且假定是进程2. 要注意, 此时进程1读取了`counter`的值,但尚未对`counter`加1.

在进程2开始运行后, 同样从其对应的接口读取数据, 并开始执行必要的操作以保存这些数据.

它会读取`counter`的值, 用序号13构造文件名打开文件, 将 `counter`加1, `counter`从13变为14. 接下来它将数据写入文件, 最后结束.

不久, 又轮到进程1再次运行. 它从上次暂停处恢复执行, 并将`counter`加1, `counter`从14变为15. 接下来它将数据写入到用序号13打开的文件, 当然, 在这样做的时候, 会覆盖进程2已经保存的数据.

这简直是祸不单行, 丢失了一个数据记录, 而且序号14也变得不可用了.

修改程序接收数据之后的处理步骤, 可以防止该错误. 举例来说,进程可以在读取`counter`的值之后立即将`counter`加1, 然后再去打开文件. 但再想想, 问题远远不会这么简单. 因为我们总是可以设计出一些导致致命错误的情形. 因此, 我们很快就意识到了 : 如果在读取`counter`的值和对其加1
之间发生调度, 则仍然会产生不一致的情况.

几个进程在访问资源时彼此干扰的情况通常称之为竞态条件(`race condition`). 在对分布式应用编程时, 这种情况是一个主要的问题, 因为竞态条件无法通过系统的试错法检测. 相反, 只有彻底研究源代码(深入了解各种可能发生的代码路径)并通过敏锐的直觉, 才能找到并消除竞态条件.

由于导致竞态条件的情况非常罕见, 因此需要提出一个问题 :  是否值得做一些(有时候是大量的)工作来保护代码避免竞态条件.

在某些环境中(飞机的控制系统、重要机械的监控、危险装备), 竞态条件是致命问题. 即使在日常软件项目中, 避免潜在的竞态条件也能大大提高程序的质量以及用户的满意度. 为改进Linux内核对多处理器的支持, 我们需要精确定位内核中暗藏竞态条件的范围,并提供适当的防护. 由于缺乏保护而导致的出乎意料的系统崩溃和莫名其妙的错误, 这些都是不可接受的.



##1.2	临界区
-------


这个问题的本质是 : 进程的执行在不应该的地方被中断, 从而导致进程工作得不正确. 显然, 一种可能的解决方案是标记出相关的代码段, 使之无法被调度器中断. 尽管这种方法原则上是可行的, 但有几个内在问题. 在某种情况下, 有问题的程序可能迷失在标记的代码段中无法退出, 因而无法放弃CPU, 进而导致计算机不可用. 因此我们必须立即放弃这种解决方案.


问题的解决方案不一定要求临界区是不能中断的. 只要没有其他的进程进入临界区, 那么在临界区中执行的进程完全是可以中断的. 这种严格的禁止条件, 可以确保几个进程不能同时改变共享的值, 我们称为互斥(`mutual exclusion`). 也就是说, 在给定时刻, 只有一个进程可以进入临界区代码.

有许多方法可以设计这种类别的互斥方法(不考虑技术实现问题). 但所有的设计都必须保证, 无论在何种情况下都要确保排他原则(`exclusion principle`). 这种保证决不能依赖于所涉及处理器的数目或速度. 如果存在这样的依赖(以至于解决方案只适用于特定硬件配置下的给定计算机系统), 那
么该方案将是不切实际的. 因为它无法提供通用的保护机制, 而这正是我们所需要的. 进程不应该允许彼此阻塞或永久停止. 尽管这里描述了一个可取的目标, 但它并不总是能够用技术手段实现, 读者从下文可以看到这一点. 经常需要程序员未雨绸缪,以避免问题的发生.


应用何种原理来支持互斥方法? 在多任务和多用户系统的历史上, 人们提出了许多不同的解决方案, 但都各有利弊. 一些解决方案只是纯理论的, 而另一些则已经在各种操作系统中付诸实践了. 下面我们将仔细讨论大多数系统采用的一种方案.


**信号量**


信号量(`semaphore`)是由`E. W. Dijkstra`在1965年设计. 初看起来, 它们对各种进程间通信问题提供了一种简单得令人吃惊的解答, 但对信号量的使用仍需要经验、直觉和谨慎.


实质上, 信号量只是受保护的特别变量, 能够表示为正负整数. 其初始值为1.


为操作信号量定义了两个标准操作 : `up`和`down`. 这两个操作分别用于控制关键代码范围的进入和退出, 且假定相互竞争的进程访问信号量机会均等.


在一个进程想要进入关键代码时, 它调用`down`函数. 这会将信号量的值减1, 即将其设置为0, 然后执行危险代码段.

在执行完操作之后, 调用`up`函数将信号量的值加1, 即重置为初始值. 信号量有下面两种特性.


1.	又一个进程试图进入关键代码段时, 首先也必须对信号量执行`down`操作. 因为第1个进程已经进入该代码段, 信号量的值此时为0. 这导致第2个进程在该信号量上"睡眠". 换句话说, 它会一直等待, 直至第1个进程退出相关的代码.

	在执行`down`操作时, 有一点特别重要. 即从应用程序的角度来看, 该操作应视为一个原子操作.

	它不能被调度器调用中断, 这意味着竞态条件是无法发生的. 从内核视角来看, 查询变量的值和修改变量的值是两个不同的操作, 但用户将二者视为一个原子操作.

	当进程在信号量上睡眠时, 内核将其置于阻塞状态, 且与其他在该信号量上等待的进程一同放到一个等待列表中.

2.	在进程退出关键代码段时, 执行`up`操作. 这不仅会将信号量的值加1(恢复为1), 而且还会选择一个在该信号量上睡眠的进程. 该进程在恢复执行后, 完成`down`操作将信号量减1(变为0), 此后即可安全地开始执行关键代码.

	如果没有内核的支持, 这个过程是不可能的, 因为用户空间库无法保证`down`操作不被中断. 在讲解对应函数的实现之前, 首先必须讨论内核自身用于保护关键代码段的机制。这些机制是用户程序使用保护措施的基础.

	信号量在用户层可以正常工作, 原则上也可以用于解决内核内部的各种锁问题. 但事实上不是这样 : 性能是内核最首先的一个目标, 虽然信号量初看起来容易实现, 但其开销对内核来说过大. 这也是内核中提供了许多不同的锁和同步机制的原因,这些我将在下文讨论.


#2	内核锁机制
-------


内核可以不受限制地访问整个地址空间. 在多处理器系统上(或类似地, 在启用了内核抢占的单处理器系统上), 这会引起一些问题. 如果几个处理器同时处于核心态, 则理论上它们可以同时访问同一个数据结构, 这刚好造成了前一节讲述的问题.

在第一个提供了SMP功能的内核版本中, 该问题的解决方案非常简单, 即每次只允许一个处理器处于核心态. 因此, 对数据未经协调的并行访问被自动排除了. 令人遗憾的是, 该方法因为效率不高, 很快被废弃了.

现在, 内核使用了由锁组成的细粒度网络, 来明确地保护各个数据结构. 如果处理器A在操作数据结构X, 则处理器B可以执行任何其他的内核操作, 但不能操作X.

内核为此提供了各种锁选项, 分别优化不同的内核数据使用模式.



| 锁 | 描述 |
|:--:|:---:|
| 原子操作 | 这些是最简单的锁操作. 它们保证简单的操作, 诸如计数器加1之类, 可以不中断地原子执行. 即使操作由几个汇编语句组成, 也可以保证 |
| 自旋锁 | 这些是最常用的锁选项. 它们用于短期保护某段代码, 以防止其他处理器的访问. 在内核等待自旋锁释放时,会重复检查是否能获取锁,而不会进入睡眠状态(忙等待). 当然, 如果等待时间较长, 则效率显然不高 |
| 信号量 | 这些是用经典方法实现的. 在等待信号量释放时, 内核进入睡眠状态, 直至被唤醒. 唤醒后, 内核才重新尝试获取信号量. 互斥量是信号量的特例, 互斥量保护的临界区, 每次只能有一个用户进入 |
| 读者/写者锁 | 这些锁会区分对数据结构的两种不同类型的访问. 任意数目的处理器都可以对数据结构进行并发读访问, 但只有一个处理器能进行写访问. 事实上, 在进行写访问时, 读访问是无法进行的 |


以下各节详细讨论了这些选项的实现和使用. 这些锁的部署遍及内核源代码各处,锁已经成为内核开发一个非常重要的方面, 无论是基础的核心内核代码还是设备驱动程序. 尽管如此, 当我在本书中讨论特定的内核代码时, 大多数情况下仍然会省略锁操作, 除非使用锁的方式很不常见, 或者锁有特殊的功能需求需要满足. 但是, 如果锁很重要, 为什么在其他章节中我们要忽略内核的这方面内容?


大多数读者几乎都会认为本书讲解已经非常详细了, 如果再把所有子系统中与锁有关的内容加以详细讨论, 则将大大超出本书的范围. 但更重要的一点是, 大多数情况下, 讨论某个特定机制的工作原理时, 对锁的讨论将干扰那些实质性的内容, 并使之复杂化. 而我的重点就是向读者讲述这些实质性的内容.

要完全理解锁的用法, 需要逐行熟悉所有受锁影响的内核代码, 而本书并没有详细讨论这部分内容, (实际上也不应该这么做).

`Linux`的源代码很容易得到, 书中完全没有必要加入读者很容易查看的内容, 而且这些内容在Linux的后续版本中有很多细节很可能会发生改变. 实质上, 书的作用在于使读者牢固理解那些不那么可能改变的概念, 这比复制源代码好得多. 尽管如此, 本章仍然会向读者提供所有必要的内容, 以理解具体的子系统如何实现针对并发操作的保护措施, 以及对这些机制的设计和工作原理的阐释. 读者要准备好投入源代码之中, 阅读并修改代码.


#3	System V进程间通信
-------



`Linux`使用`System V(SysV)`引入的机制, 来支持用户进程的进程间通信和同步. 内核通过系统调用提供了各种例程, 使得用户库(通常是C标准库)能够实现所需的操作.


除了信号量之外, `SysV`的进程间通信方案还包括进程间的消息交换和共享内存区域, 如下所述.



##3.1	System V机制
-------


`System V UNIX`的3种进程间通信(`IPC`)机制(信号量、消息队列、共享内存)反映了3种相去甚远的概念, 不过三者却有一个共同点. 它们都使用了全系统范围的资源, 可以由几个进程同时共享.


对于IPC机制而言, 这看起来似乎是合理的, 但不应该视作理所当然. 举例来说, 该机制最初的设计目标, 可能只是为了让程序的各个线程或`fork`产生的结构能够访问共享的`SysV`对象.


在各个独立进程能够访问`SysV IPC`对象之前, `IPC`对象必须在系统内唯一标识. 为此, 每种`IPC`结构在创建时分配了一个号码. 凡知道这个魔数的各个程序, 都能够访问对应的结构. 如果独立的应用程序需要彼此通信, 则通常需要将该魔数永久地编译到程序中. 一种备选方案是动态地产生一个保
证唯一的魔数(静态分配的号码无法保证唯一). 标准库提供了几个完成此工作的函数(详细信息请参见相关的系统程序设计手册).

在访问`IPC`对象时, 系统采用了基于文件访问权限的一个权限系统. 每个`IPC`对象都有一个用户`ID`和一个组`ID`, 依赖于产生`IPC`对象的程序在何种`UID/GID`之下运行. 读写权限在初始化时分配. 类似于普通的文件, 这些控制了3种不同用户类别的访问 : 所有者、组、其他. 这些工作具体如何完成, 详细信息请参考对应的系统程序设计手册.

要创建一个授予所有可能访问权限的信号量(所有者、组、其他用户都有读写权限), 则必须指定标志0666.



| System V机制 | 描述 |
|:-----------:|:----:|
| System V`信号量 | 信号量的使用主要是用来保护共享资源，使得资源在一个时刻只有一个进程(线程)所拥有. 信号量的值为正的时候，说明它空闲。所测试的线程可以锁定而使用它。若为0，说明它被占用，测试的线程要进入睡眠队列中，等待被唤醒 |
| 消息队列 | 消息队列提供了一种从一个进程向另一个进程发送一个数据块的方法. 每个数据块都被认为含有一个类型, 接收进程可以独立地接收含有不同类型的数据结构. 我们可以通过发送消息来避免命名管道的同步和阻塞问题. 但是消息队列与命名管道一样, 每个数据块都有一个最大长度的限制 |
| 共享内存 | 共享内存就是允许两个不相关的进程访问同一个逻辑内存. 共享内存是在两个正在运行的进程之间共享和传递数据的一种非常有效的方式. 不同进程之间共享的内存通常安排为同一段物理内存. 进程可以将同一段共享内存连接到它们自己的地址空间中, 所有进程都可以访问共享内存中的地址, 就好像它们是由用C语言函数malloc分配的内存一样. 而如果某个进程向共享内存写入数据, 所做的改动将立即影响到可以访问同一 |


##3.2	`System V`信号量
-------


`System V`信号量在`sem/sem.c`实现, 对应的头文件是 `<sem.h>`. 这种信号量与上文讲述的内核信号量没有任何关系.

`System V`的信号量接口决不直观, 因为信号量的概念已经远超其实际定义了. 信号量不再当作是用于支持原子执行预定义操作的简单类型变量. 相反,一个`System V`信号量现在是指一整套信号量, 可以允许几个操作同时进行(尽管用户看上去它们是原子的). 当然可以请求只有一个信号量的信号
量集合, 并定义函数模拟原始信号量的简单操作. 以下示例程序说明了信号量的使用方式.



##3.3	消息队列
-------


进程之间通信的另一个方法是交换消息. 这是使用消息队列机制完成的,其实现基于`System V`模型. 就涉及的数据结构而言, 消息队列和信号量有某些共同点.

产生消息并将其写到队列的进程通常称之为发送者, 而一个或多个其他进程(逻辑上称之为接收者)则从队列获取信息. 各个消息包含消息正文和一个(正)数, 以便在消息队列内实现几种类型的消息. 接收者可以根据该数字检索消息, 例如, 可以指定只接受编号1的消息, 或接受编号不大于5的消息. 在消息已经读取后, 内核将其从队列删除. 即使几个进程在同一信道上监听, 每个消息仍然只能由一个进程读取.

同一编号的消息按先进先出次序处理. 放置在队列开始的消息将首先读取. 但如果有选择地读取消息, 则先进先出次序就不再适用.


##3.4	共享内存
-------

共享内存是进程间通信的最后一个概念, 从用户和内核的角度来看, 它的实现使用了与上述两种机制类似的结构. 与信号量和消息队列相比, 共享内存没有本质性的不同.

*	应用程序请求的IPC对象, 可以通过魔数和当前命名空间的内核内部ID访问.

*	对内存的访问, 可能受到权限系统的限制.

*	可以使用系统调用分配与`IPC`对象关联的内存, 具备适当授权的所有进程, 都可以访问该内存.

内核的实现采用了与前述两种对象非常类似的概念.

同样, 在`smd_ids`全局变量的`entries`数组中保存了 `kern_ipc_perm`和`shmid_kernel`的组合, 以
便管理`IPC`对象的访问权限. 对每个共享内存对象都创建一个伪文件, 通过`shm_file`连接到`shmid_kernel`的实例. 内核使用`shm_file->f_mapping`指针访问地址空间对象(`struct address_space`), 用于创建匿名映射. 还需要设置所涉及各进程的页表, 使得各个进程都能够访问与该IPC对象相关的内存区域.


#4	其他IPC机制
-------



##4.1	信号
-------


与`SysV`机制相比, 信号是一种比较原始的通信机制. 尽管提供的选项较少, 但是它们非常有用.



其底层概念非常简单, `kill`命令根据`PID`向进程发送信号. 信号通过`-s sig`指定, 是一个正整数, 最大长度取决于处理器类型.

该命令有两种最常用的变体 :


*	一种是`kill`不指定信号, 实际上是要求进程结束(进程可以忽略该信号);


*	另一种是`kill -9`, 等价于在死刑批准上签字(导致某些进程死亡).


过去, 32位系统最多支持32个信号, 该限制现在已经提高了, `kill`手册页上列出的所有信号都已经支持. 不过, 经典的信号占用了信号列表中前32个位置. 接下来是针对实时进程引入的新信号.

进程必须设置处理程序例程来处理信号. 这些例程在信号发送到进程时调用(但有几个信号的行为无法修改, 如`SIGKILL`). 如果没有显式设置处理程序例程, 内核则使用默认的处理程序实现.

信号引入了几种特性, 必须永远切记. 进程可以决定阻塞特定的信号(有时称之为信号屏蔽).


如果发生这种情况, 会一直忽略该信号, 直至进程决定解除阻塞. 因而, 进程是否能感知到发送的信号, 是不能保证的.

在信号被阻塞时, 内核将其放置到待决列表上. 如果同一个信号被阻塞多次, 则在待决列表中只放置一次. 不管发送了多少相同的信号, 在进程删除阻塞之后, 都只会接收到一个信号.


`SIGKILL`信号无法阻塞, 也不能通过特定于进程的处理程序处理. 之所以不能修改该信号的行为, 是因为它是从系统删除失控进程的最后手段. 它与`SIGTERM`信号不同, 后者可以通过用户定义的信号处理程序处理, 实际上只是向进程发出的一个客气的请求, 要求进程尽快停止工作而已. 如果已经为
该信号设置了处理程序, 那么程序就有机会保存数据或询问用户是否确实想要退出程序.

`SIGKILL`不会提供这种机会, 因为内核会立即强行终止进程.

`init`进程属于特例. 内核会忽略发送给该进程的`SIGKILL`信号. 因为该进程对整个系统尤其重要, 不能强制结束该进程, 即使无意结束也不行.



##4.2	管道
-------

管道和套接字是流行的进程间通信机制. 我在这里只概述这两个概念的工作方式, 因为二者都大量使用了内核的其他子系统. 管道使用了虚拟文件系统对象, 而套接字使用了各种网络函数以及虚拟文件系统.


`shell`用户可能比较熟悉管道, 在命令行上可以如下使用 :

```cpp
prog | ghostscript | lpr-
```


这里将一个进程的输出用作另一个进程的输入, 管道负责数据的传输. 顾名思义, 管道是用于交换数据的连接. 一个进程向管道的一端供给数据, 另一个在管道另一端取出数据, 供进一步处理. 几个进程可以通过一系列管道连接起来.


在通过`shell`产生管道时, 总有一个读进程和一个写进程.


应用程序必须调用`pipe`系统调用产生管道. 该调用返回两个文件描述符, 分别用于管道的两端, 即分别用于管道的读和写. 由于两个描述符存在于同一个进程中, 进程最初只能向自身发送消息, 所以这不怎么实用.


管道是进程地址空间中的数据对象, 在用`fork`或`clone` 复制进程时同样会被复制. 使用管道通信的程序就利用了这种特征. 在`exec`系统调用用另一个程序替换子进程之后, 两个不同的应用程序之间就建立了一条通信链路(必须把管道描述符重定向到标准输入和输出, 或者调用`dup`系统调用, 以确保`exec`调用时不会关闭文件描述符).



##4.3	套接字
-------


套接字对象在内核中初始化时也返回一个文件描述符, 因此可以像普通文件一样处理. 但不同于管道, 套接字可以双向使用, 还可以用于与通过网络连接的远程系统通信(这并不意味着套接字无法用于支持本地系统上两个进程之间的通信).


套接字的实现是内核中相当复杂的一部分, 因为需要大量抽象机制来隐藏通信的细节. 从用户的角度来看, 同一系统上两个本地进程之间的通信或分别处于两个不同大陆的两台计算机上运行的应用程序之间的通信, 它们没有太大差别.


在内核版本2.6.26开发期间, 信号量特定于体系结构的实现, 已经替换为一种通用形式. 当然, 与优化代码相比,通用实现的执行效率稍差, 但由于信号量在内核中并未广泛应用(互斥量常见得多), 实际上没什么问题.

`struct semaphore`的定义已经移到 `include/linux/semaphore.h`, 所有相关操作在`kernel/semaphore.c`中实现. 最重要的是, 信号量API没有改变, 如此使用信号量的现存代码无需修改.

在内核`2.6.26`开发期间引入的另一个改变是自旋锁的实现. 根据定义我们认为这种锁一般都处于非竞争状态, 因此内核没有提供在多个等待者之间实现公平的机制. 也就是说, 如果有多个进程在等待自旋锁, 那么在锁被当前持有者释放之后, 等待进程的运行次序是未定义的. 但测量结果表明, 在
处理器数目较多的计算机上这种做法可能导致不公平问题, 例如有8个CPU的系统. 当今这种计算机并不罕见,因此修改了自旋锁的实现,使得多个等待者获取锁的顺序与到达的顺序相同. API同样没有变化,因此使用自旋锁的现存代码也无需修改.



#5	小结
-------



尽管几年前多处理器系统仍然很罕见, 但近来半导体工程取得的成就彻底改变了这一点. 由于多核CPU的出现, SMP计算机不再仅限于数值计算和超级计算等专业领域, 也出现在了普通的桌面计算机上. 这对内核提出了一些很独特的挑战 : 内核的多个实例可以同时运行, 而这需要协调对共享数据结构的操作. 内核对此提供了一整套机制, 些机制从简单快速的自旋锁
到强大的RCU机制, 可以在保证性能的同时确保并行操作的正确性. 选择适当的解决方案非常重要, 我也讨论过需要选择适当的设计, 通过细粒度锁来保证性能的同时, 在较小型的计算机上不增加过多的开销.


在用户层进程彼此通信时, 也会出现与内核类似的问题. 除了提供允许独立进程通信的机制之外, 内核还必须向进程提供同步机制. 在本章中, 我将讨论了在`Linux`内核里, 如何实现原本是`System V UNIX`中的这些机制.