Golang 的特色之一就是 goroutine ，使得程序员进行并发编程更加方便，适合用来进行服务器编程。作为后端开发工程师，有必要了解并发编程面临的场景和常见的解决方案。一般情况下，是怎样做高并发的编程呢？有那些经典的模型呢？

---
# 一切始于 C10k

*C10k* 就是 Client 10000，单机服务器同时服务1万个客户端。当然，现在的业务面临的是 C100k、C1000k 了。早期的服务器是基于进程/线程模型，每新来一个连接，就分配一个进程（线程）去处理这个连接。而进程（线程）在操作系统中，占有一定的资源。由于硬件的限制，进程（线程）的创建是有瓶颈的。另外进程（线程）的上下文切换也有成本：每次调度器调度线程，操作系统都要把线程的各种必要的信息，如程序计数器、堆栈、寄存器、状态等保存起来。

CPU 运算远远快于 I/O 操作。一般而言，常见的互联网应用（比如 Web）都是 I/O 密集型而非计算密集型。I/O 密集型是指，计算机 CPU 大量的时间都花在等待数据的输入输出，而不是计算。当 CPU 大部分时间都在等待 I/O 的时候，大部分计算资源都被浪费掉了。

显然，简单粗暴地开一个进程/线程去 handle 一个连接是不够的。为了达到高并发，应该好好考虑一下 I/O 策略。同样的硬件条件下，不同的设计产生的效果差别也会很大。在讨论几种 I/O 模型之前，先介绍一下同步/异步、阻塞/非阻塞的概念，以及操作系统的知识。

参考：

[The C10K problem](http://www.kegel.com/c10k.html)

---

## 同步/异步？阻塞/非阻塞？

同步，是调用者主动去查看调用的状态；异步，则是被调用者来通知调用者。例如在 Web 应用里，后端通过渲染模版的方式把 Web 页面发送给前端，是同步的方式。这里前端是调用者，每一次请求数据，都要把整个页面重新加载一次。而前端用 jQuery Ajex 向服务器请求数据，则是异步的，每次请求数据不需要把整个页面重新加载，局部刷新即可。

阻塞和非阻塞的区别是调用后是否立即返回。 A 调完 B，就在调用处等待（阻塞），直到 B 方法返回才继续执行剩下的代码，这就是阻塞调用。而非阻塞是 A 方法调用 B 方法，B 方法立即返回，A 可以继续执行下面的代码，不会被该调用阻塞。当某个方法被阻塞了，该方法所在的线程会被挂起，被操作系统的调度器放到阻塞队列，直到 A 等待的事件发生，才从阻塞态转到就绪态。

Unix 下的 I/O 模型也有同步/异步、阻塞/非阻塞的概念，可以查看我做的笔记：[UNIX 中的 I/O 模型](https://github.com/gobomb/myDoc/wiki/io-model)

---

## 进程、线程、协程

*进程* 是系统进行资源分配的一个独立单位。这些资源包括：用户的地址空间，实现进程（线程）间同步和通信的机制，已打开的文件和已申请到的I/O设备，以及一张由核心进程维护的地址映射表。内核通过 *进程控制块* （PCB，process control block）来感知进程。

*线程* 是调度和分派的基本单位。内核通过 *线程控制块* （TCB，thread control block）来感知线程。

线程本身不拥有系统资源，而是仅有一点必不可少的、能保证独立运行的资源，如TCB、程序计数器、局部变量、状态参数、返回地址等寄存器和堆栈。同一进程的所有线程具有相同的地址空间，线程可以访问进程拥有的资源。多个线程可并发执行，一个进程含有若干个相对独立的线程，但至少有一个线程。

线程的有不同的实现方式，分 *内核支持线程* （KST，Kernel Supported Threads）和 *用户级线程* (UST, User Supported Threads)。内核级线程的 TCB 保存在内核空间，其创建、阻塞、撤销、切换等活动也都是在内核空间实现的。用户级线程则是内核无关的，用户级线程的实现在用户空间，内核感知不到用户线程的存在。用户线程的调度算法可以是进程专用的，不会被内核调度，但同时，用户线程也无法利用多处理机的并行执行。而一个拥有多个用户线程的进程，一旦有一个线程阻塞，该进程所有的线程都会被阻塞。内核的切换需要转换到内核空间，而用户线程不需要，所以前者开销会更大。但用户线程也需要内核的支持，一般是通过运行时系统或内核控制线程来连接一个内核线程，有 1:1、1:n、n:m 的不同实现。

在分时操作系统中，处理机的调度一般基于时间片的轮转（RR, round robin)，多个就绪线程排成队列，轮流执行时间片。而为保证交互性和实时性，线程都是以抢占的方式（Preemptive Mode）来获得处理机。而抢占方式的开销是比较大的。有抢占方式就有非抢占方式（Nonpreemptiv Mode)，在非抢占式中，除非某正在运行的线程执行完毕、因系统调用（如 I/O 请求）发生阻塞或主动让出处理器，不会被调度或暂停。

而 *协程* （Coroutine）就是基于非抢占式的调度来实现的。进程、线程是操作系统级别的概念，而协程是编译器级别的，现在很多编程语言都支持协程，如 Erlang、Lua、Python、Golang。准确来说，协程只是一种用户态的轻量线程。它运行在用户空间，不受系统调度。它有自己的调度算法。在上下文切换的时候，协程在用户空间切换，而不是陷入内核做线程的切换，减少了开销。简单地理解，就是编译器提供一套自己的运行时系统（而非内核）来做调度，做上下文的保存和恢复，重新实现了一套“并发”机制。系统的并发是时间片的轮转，单处理器交互执行不同的执行流，营造不同线程同时执行的感觉；而协程的并发，是单线程内控制权的轮转。相比抢占式调度，协程是主动让权，实现协作。协程的优势在于，相比回调的方式，写的异步代码可读性更强。缺点在于，因为是用户级线程，利用不了多核机器的并发执行。

线程的出现，是为了分离进程的两个功能：资源分配和系统调度。让更细粒度、更轻量的线程来承担调度，减轻调度带来的开销。但线程还是不够轻量，因为调度是在内核空间进行的，每次线程切换都需要陷入内核，这个开销还是不可忽视的。协程则是把调度逻辑在用户空间里实现，通过自己（编译器运行时系统/程序员）模拟控制权的交接，来达到更加细粒度的控制。


参考：

[《计算机操作系统》](https://book.douban.com/subject/26079463/)


---

# 并发模型

## 1. 单进（线）程·循环处理请求

单进程和单线程其实没有区别，因为一个进程至少有一个线程。循环处理请求应该是最初级的做法。当大量请求进来时，单线程一个一个处理请求，请求很容易就积压起来，得不到响应。这是无并发的做法。

---

## 2. 多进程

主进程监听和管理连接，当有客户请求的时候，fork 一个子进程来处理连接，父进程继续等待其他客户的请求。但是进程占用服务器资源是比较多的，服务器负载会很高。

Apache 是多进程服务器。有两种模式：

1. Prefork MPM : 使用多个子进程，但每个子进程不包含多线程。每个进程只处理一个连接。在许多系统上它的速度和worker MPM一样快，但是需要更多的内存。这种无线程的设计在某些性况下优于 worker MPM，因为它可在应用于不具备线程安全的第三方模块上（如 PHP3/4/5），且在不支持线程调试的平台上易于调试，另外还具有比worker MPM更高的稳定性。

2. Worker MPM : 使用多个子进程，每个子进程中又有多个线程。每个线程处理一个请求，该MPM通常对高流量的服务器是一个不错的选择。因为它比prefork MPM需要更少的内存且更具有伸缩性。


这种架构的最大的好处是隔离性，子进程万一 crash 并不会影响到父进程。缺点就是对系统的负担过重。

参考：

[web服务器apache架构与原理](https://www.cnblogs.com/fnng/archive/2012/11/08/2761713.html)

---

## 3. 多线程
和多进程的方式类似，只不过是替换成线程。主线程负责监听、`accept()`连接，子线程（工作线程）负责处理业务逻辑和流的读取。子线程阻塞，同一进程内的其他线程不会被阻塞。

缺点是：

  1. 会频繁地创建、销毁线程，这对系统也是个不小的开销。这个问题可以用线程池来解决。线程池是预先创建一部分线程，由线程池管理器来负责调度线程，达到线程复用的效果，避免了反复创建线程带来的性能开销，节省了系统的资源。

  2. 要处理同步的问题，当多个线程请求同一个资源时，需要用锁之类的手段来保证线程安全。同步处理不好会影响数据的安全性，也会拉低性能。

  3. 一个线程的崩溃会导致整个进程的崩溃。

多线程的适用场景是：提高响应速度，让IO和计算相互重叠，降低延时。虽然多线程不能提高绝对性能，但是可以提高平均响应性能。

这种其实是比较容易想到的，特别是对于刚刚学习多线程和操作系统的计算机学生而言。在请求量不高的时候，是足够的。来多少连接开多少线程，就看服务器的硬件性能能不能承受。但高并发并不是线性地堆砌硬件或加线程数就能达到的。100个线程也许能够达到1000的并发，但10000的并发下，线程数乘以10也许就不行，比如线程调度带来的开销、同步成为了瓶颈。

---

## 4. 单线程·回调（callback）和事件轮询

### Nginx

Nginx 采用的是多进程（单线程） & 多路IO复用模型:

1. Nginx 在启动后，会有一个 master 进程和多个相互独立的 worker 进程。

2. 接收来自外界的信号，向各 worker 进程发送信号，每个进程都有可能来处理这个连接

3. master 进程能监控 worker 进程的运行状态，当 worker 进程退出后(异常情况下)，会自动启动新的 worker 进程。

主进程（master 进程）首先通过 socket() 来创建一个 sock 文件描述符用来监听，然后fork生成子进程（workers 进程），子进程将继承父进程的 sockfd（socket 文件描述符），之后子进程 accept() 后将创建已连接描述符（connected descriptor）），然后通过已连接描述符来与客户端通信。

存在惊群现象：当连接进来时，所有子进程都将收到通知并“争着”与它建立连接。

Nginx 在 accept 上加一把互斥锁来应对惊群现象。

在每个 worker 进程里，Nginx 调用内核 epoll()函数来实现 I/O 的多路复用。


参考：

[初步探索Nginx高并发原理
](http://blog.csdn.net/stfphp/article/details/52936490)

---

### Node.js

Node.js 也是单线程模型。Node.js中所有的逻辑都是事件的回调函数，所以 Node.js始终在事件循环中，程序入口就是事件循环第一个事件的回调函数。事件的回调函数中可能会发出I/O请求或直接发射（ emit ）事件，执行完毕后返回事件循环。事件循环会检查事件队列中有没有未处理的事件，直到程序结束。Node.js的事件循环对开发者不可见，由 libev 库实现，libev 不断检查是否有活动的、可供检测的事件监听器，直到检查不到时才退出事件循环，程序结束。

Node.js 单线程能够实现非阻塞，是因为其底层实现有另一个线程在轮询事件队列，对于上层的开发者，只需考虑单线程，没有权限去开新的线程，也不需要考虑线程同步之类的问题。

这种机制的缺点是，会造成大量回调函数的嵌套，代码可读性不佳。因为没有多线程，在多核的机器上，也没办法实现并行执行。

 参考：

 [Node.js机制及原理理解初步](https://www.cnblogs.com/dengchj/p/7171978.html)

 [使用 libevent 和 libev 提高网络应用性能](https://www.ibm.com/developerworks/cn/aix/library/au-libev/index.html)

---

## 5. 协程

协程基于用户空间的调度器，具体的调度算法由具体的编译器和开发者实现，相比多线程和事件回调的方式，更加灵活可控。不同语言协程的调度方式也不一样，python是在代码里显式地`yield`进行切换，golang 则是用`go`语法来开启 goroutine，具体的调度由语言层面提供的运行时执行。

gorounte 的堆栈比较小，一般是几k，可以动态增长。线程的堆栈空间在 Windows 下默认 2M，Linux 下默认 8M。这也是 goroutine 单机支持上万并发的原因，因为它更廉价。

从堆栈的角度，进程拥有自己独立的堆和栈，既不共享堆，亦不共享栈，进程由操作系统调度。线程拥有自己独立的栈和共享的堆，共享堆，不共享栈，线程亦由操作系统调度(内核线程)。协程和线程一样共享堆，不共享栈，协程由程序员在协程的代码里显示调度。

在使用 goroutine 的时候，可以把它当作轻量级的线程来用，和多进程、多线程方式一样，主 goroutine 监听，开启多个工作 goroutine 处理连接。比起多线程的方式，优势在于能开更多的 goroutine，来处理连接。

goroutine 的底层实现，关键在于三个基本对象上，G(goroutine)，M(machine)，P (process)。M：与内核线程连接，代表内核线程；P：代表M运行G所需要的资源，可以把它看做一个局部的调度器，维护着一个goroutine队列；G:代表一个goroutine，有自己的栈。M 和 G 的映射，可以类比操作系统内核线程与用户线程的 m:n 模型。通过对 P 数量的控制，可以控制操作系统的并发度。

参考：

[如何理解 Golang 中“不要通过共享内存来通信，而应该通过通信来共享内存”？](https://www.zhihu.com/question/58004055)

[Golang源码探索(二) 协程的实现原理](http://www.cnblogs.com/zkweb/p/7815600.html)

[Goroutine（协程）为何能处理大并发？](https://studygolang.com/articles/4535)

[协程](https://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/0013868328689835ecd883d910145dfa8227b539725e5ed000)


---

# Actor 和 CSP 模型

传统的多线程编程，是用共享内存的方式来进行同步的。但当并行度变高，不确定性就增加了，需要用锁等机制保证正确性，但锁用得不好容易拉低性能。而且多线程编程也是比较困难的，不太符合人的思维习惯，很容易出错，会产生死锁。所以有一些新的编程模型来实现高并发，用消息传递来代替共享内存和锁。

于是就有了“Don’t communicate by sharing memory, share memory by communicating”（不要通过共享内存来通信，而应该通过通信来共享内存）的思想，Actor 和 CSP 就是两种基于这种思想的并发编程模型，学术界已有诸多论文加以阐述。也就是说，这是有数学证明的，了解这两种模型，能给高并发服务器的开发很多有益的启发。作为工程师，不一定要有理论创新，但要学会把理论成果用到自己的项目上面。

「Actor 模型的重点在于参与交流的实体,而 CSP 模型的重点在于用于交流的通道。」Java/Scala 有个库 akka，就是 Actor 模型的实现。而 golang 的协程机制则是 CSP 模型。


「Actor 模型推崇的哲学是“一切皆是参与者(actor)”，这与面向对象编程的“一切皆是对象”类似。」「Actor模型=数据+行为+消息。Actor模型内部的状态由自己的行为维护，外部线程不能直接调用对象的行为，必须通过消息才能激发行为，这样就保证Actor内部数据只有被自己修改。」

我的理解是，在模型内部，对数据的处理始终是单线程的，所以无需要考虑线程安全，无需加锁，外部可以是多线程，要操作数据需要向内部线程发送消息，内部线程一次只处理一次消息，一个消息代表一个处理数据的行为。内部线程和外部线程通过信箱（mailbox）来实现异步的消息机制。


CSP 与 Actor 类似，process（在 go 中则是 goroutine） 对应 acotor，也就是发送消息的实体。 channel 对应 mailbox，是传递消息的载体。区别在与一个 actor 只有一个 mailbox，actor 和 mailbox 是耦合的。channel 是作为 first-class 独立存在的（这在 golang 中很明显），channel 是匿名的。mailbox 是异步的，channel 一般是同步的（在 golang 里，channel 有同步模式，也可以设置缓冲区大小实现异步）。


参考：

[actor并发模型&基于共享内存线程模型](http://www.jdon.com/45516)

[为什么Actor模型是高并发事务的终极解决方案？](http://www.jdon.com/45728)

[如何深入浅出地解释并发模型中的 CSP 模型？](https://www.zhihu.com/question/26192499)

[并发编程：Actors模型和CSP模型](http://www.importnew.com/24226.html)

---

# 总结

高并发的关键在于实现异步非阻塞，更加高效地利用 CPU。多线程可以达到非阻塞，但占用资源多，切换开销大。协程用栈的动态增长、用户态的调度来避免多线程的两个问题。事件驱动用单线程的方式，避免了占用太多系统资源，不需要关心线程安全，但无法利用多核。具体要采用哪种模型，还是要看需求。模型或技术只是工具，条条大陆通罗马。

比较优雅的还是 CSP 和 Actor 模型，因为能够符合人的思维习惯，避免了锁的使用。个人觉得加锁和多线程的方式，很容易被滥用，这是一种从微观出发和线性的思维方式，不够高屋建瓴。不如用消息通信来的耦合性更低。

高并发编程很有必要性。一方面，很多应用都需要高并发支持，网络的用户越来越多，业务场景会越来越复杂，需要有稳定和高效的服务器支持。另一方面，现代的计算机性能都是比较高的，但如果软件设计得不够好，就不能够把性能都给发挥出来。这就很浪费了。

在写这篇文章的时候，我发现了很多有趣的开源源码和项目，值得进一步研究和阅读，但时间有限，暂时没有深入。接下来会继续了解一下，然后更新一些文章：

1. *libtask* golang 作者之一 Russ Cox 实现的 C 语言协程库，golang 的 goroutine 就参考了这个库的实现： https://swtch.com/libtask/

2. *libev* 事件驱动编程框架：http://software.schmorp.de/pkg/libev.html

3. *akka* scala 实现的 Actor 框架：https://akka.io/

这篇文章花了我快一周的时间，查了大量的资料，写作难度比我想象中大很多，而且还写得不好，引用了不少博客的说法，还没有办法自己组织出比较好的语言来阐述问题。有些点可能还理解错了，以后再改吧。不过好歹写完了。至少对主流的并发编程有了个感性的理解，也算是对自己的一个交代。

---
# update：函数式编程

2018.01.01

最近了解到，函数式编程也是一个可以用来解决并发问题的模型。

命令式语言和函数式语言的抽象不同。

命令式编程是对计算机硬件的抽象，关心的是解决问题的步骤。函数式编程是对数学的抽象，把问题转化为数学表达式。

函数性语言两个特征：数据不可变，不依赖保存或检索状态的操；无副作用，用相同的输入调用函数，总是返回相同的值。也因此，可以不依赖锁来做并发编程。

还没有学习函数式的语言，所以对函数式编程如何做到并发不是很理解。但能感受到，函数式语言是一个值得探寻的领域。

有一句话“软件的首要技术使命是管理复杂度。”（《代码大全》）。之所以存在这么多抽象，一方面是要有效地解决问题，另一方面，也是为了降低程序员的心智负担。编程模型其实就是程序员看待问题的方式。同样解决问题，当然是选择编程友好、符合人的思维习惯的编程模型比较好。“代码是写给人看的,不是写给机器看的”（SICP）。虽然机器一样能执行，但最终的目的是为了解放人，让人能把大部分精力花在刀刃上、花在创造性的工作上。



参考：

[函数式编程初探](http://www.ruanyifeng.com/blog/2012/04/functional_programming.html)

[什么是函数式编程思维？](https://www.zhihu.com/question/28292740)