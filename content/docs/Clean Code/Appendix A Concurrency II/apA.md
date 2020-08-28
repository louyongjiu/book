# 附录 A 并发编程 II

by Brett L. Schuchert

This appendix supports and amplifies the Concurrency chapter on page 177. It is written as a series of independent topics and you can generally read them in any order. There is some duplication between sections to allow for such reading.

> 本附录扩充了“并发编程”一章的内容，由一组相互独立的主题组成，你可以按随意顺序阅读。为了实现这样的阅读方式，节与节之间存在一些重复内容。

## A.1 CLIENT/SERVER EXAMPLE 客户端/服务器的例子

Imagine a simple client/server application. A server sits and waits listening on a socket for a client to connect. A client connects and sends a request.

> 想象一个简单的客户端/服务器应用程序。服务器在一个套接字上等待接受来自客户端的连接请求。客户端连接到服务器并发送请求。

### A.1.1 The Server 服务器

Here is a simplified version of a server application. Full source for this example is available starting on page 343, Client/Server Nonthreaded.

> 下面是服务器应用程序的简化版本代码。在后文“客户端/服务器非多线程版本”一节中有完整的代码。

```java
ServerSocket serverSocket = new ServerSocket(8009);

while (keepProcessing) {
    try {
        Socket socket = serverSocket.accept();
        process(socket);
    } catch (Exception e) {
        handle(e);
    }
}
```

This simple application waits for a connection, processes an incoming message, and then again waits for the next client request to come in. Here’s client code that connects to this server:

> 这个简单的应用等待连接请求，处理接收到的新消息，再等待下一个客户端请求。下面是连接到服务器的客户端代码：

```java
private void connectSendReceive(int i) {
    try {
        Socket socket = new Socket(“localhost”, PORT);
        MessageUtils.sendMessage(socket, Integer.toString(i));
        MessageUtils.getMessage(socket);
        socket.close();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

How well does this client/server pair perform? How can we formally describe that performance? Here’s a test that asserts that the performance is “acceptable”:

> 这对客户端/服务器程序运行得如何呢？怎样才能正式地描述其性能？下面是断言其性能“可接受”的测试：

```java
@Test(timeout = 10000)
public void shouldRunInUnder10Seconds() throws Exception {
    Thread[] threads = createThreads();
    startAllThreadsw(threads);
    waitForAllThreadsToFinish(threads);
}
```

The setup is left out to keep the example simple (see “ClientTest.java” on page 344). This test asserts that it should complete within 10,000 milliseconds.

> 为了让例子够简单，设置过程被忽略了（见后文 ClientText.java 部分）。测试断言程序应该在 10000 毫秒内完成。

This is a classic example of validating the throughput of a system. This system should complete a series of client requests in ten seconds. So long as the server can process each individual client request in time, the test will pass.

> 这是个验证系统吞吐量的典型例子。系统应该在 10 秒钟以内完成一组客户端请求。只要服务器能在时限内处理每个客户端请求，测试就通过了。

What happens if the test fails? Short of developing some kind of event polling loop, there is not much to do within a single thread that will make this code any faster. Will using multiple threads solve the problem? It might, but we need to know where the time is being spent. There are two possibilities:

> 如果测试失败会怎样？缺少了某些事件轮询机制，在单个线程上也没什么可让代码更快的手段。使用多线程能解决问题吗？可能会，我们先得了解什么地方耗费时间。下面是两种可能：

- I/O—using a socket, connecting to a database, waiting for virtual memory swapping, and so on.
- Processor—numerical calculations, regular expression processing, garbage collection, and so on.

---

> - I/O——使用套接字、连接到数据库、等待虚拟内存交换等；
> - 处理器——数值计算、正则表达式处理、垃圾回收等。

Systems typically have some of each, but for a given operation one tends to dominate. If the code is processor bound, more processing hardware can improve throughput, making our test pass. But there are only so many CPU cycles available, so adding threads to a processor-bound problem will not make it go faster.

> 以上在系统中都会部分存在，但对于特定的操作，其中之一会起主导作用。如果代码运行速度主要与处理器有关，增加处理器硬件就能提升吞吐量，从而通过测试。但 CPU 运算周期是有上限的，因此，只是增加线程的话并不会提升受处理器限制的代码的速度。

On the other hand, if the process is I/O bound, then concurrency can increase efficiency. When one part of the system is waiting for I/O, another part can use that wait time to process something else, making more effective use of the available CPU.

> 另一方面，如果吞吐量与 I/O 有关，则并发编程能提升运行效率。当系统的某个部分在等待 I/O，另一部分就可以利用等待的时间处理其他事务，从而更有效地利用了 CPU 能力。

### A.1.2 Adding Threading 添加线程代码

Assume for the moment that the performance test fails. How can we improve the throughput so that the performance test passes? If the process method of the server is I/O bound, then here is one way to make the server use threads (just change the processMessage):

> 假定性能测试失败了。如何才能提高吞吐量、通过性能测试呢？如果服务器的 process 方法与 I/O 有关，就有个办法让服务器利用线程（只需要修改 processMessage）：

```java
void process(final Socket socket) {
    if (socket == null)
        return;

    Runnable clientHandler = new Runnable() {
        public void run() {
            try {
                String message = MessageUtils.getMessage(socket);
                MessageUtils.sendMessage(socket, “Processed: ” + message);
                closeIgnoringException(socket);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    };

    Thread clientConnection = new Thread(clientHandler);
    clientConnection.start();
}
```

Assume that this change causes the test to pass;1 the code is complete, correct?

> 假设修改后测试通过了。代码是否完整、正确了呢？

### A.1.3 Server Observations 观察服务器端

The updated server completes the test successfully in just over one second. Unfortunately, this solution is a bit naive and introduces some new problems.

> 修改了的服务器成功通过测试，只花费了一秒多钟时间。不幸的是，这种解决手段有点一厢情愿，而且导致了新问题产生。

How many threads might our server create? The code sets no limit, so the we could feasibly hit the limit imposed by the Java Virtual Machine (JVM). For many simple systems this may suffice. But what if the system is meant to support many users on the public net? If too many users connect at the same time, the system might grind to a halt.

> 服务器应该创建多少个线程？代码没有设置上限，所以我们很有可能达到 Java 虚拟机（JVM）的限制。对于许多简单系统来说这无所谓。但如果系统要支持公众网络上的众多用户呢？如果有太多用户同时连接，系统就有可能挂掉。

But set the behavioral problem aside for the moment. The solution shown has problems of cleanliness and structure. How many responsibilities does the server code have?

> 不过先把性能问题放到一边吧。这种手段还有整洁性和结构上的问题。服务器代码有多少种权责呢？

- Socket connection management
- Client processing
- Threading policy
- Server shutdown policy

---

> - 套接字连接管理；
> - 客户端处理；
> - 线程策略；
> - 服务器关闭策略。

Unfortunately, all these responsibilities live in the process function. In addition, the code crosses many different levels of abstraction. So, small as the process function is, it needs to be repartitioned.

> 这些权责不幸全在 process 函数中。而且，代码跨越多个抽象层级。所以，即便 process 函数这么短小，还是需要再加以切分。

The server has several reasons to change; therefore it violates the Single Responsibility Principle. To keep concurrent systems clean, thread management should be kept to a few, well-controlled places. What’s more, any code that manages threads should do nothing other than thread management. Why? If for no other reason than that tracking down concurrency issues is hard enough without having to unwind other nonconcurrency issues at the same time.

> 服务器有几个修改的原因，所以它违反了单一权责原则。要保持并发系统整洁，应该将线程管理代码约束于少数几处控制良好的地方。而且，管理线程的代码只应该做管理线程的事。为什么？即便无需同时考虑其他非多线程代码，跟踪并发问题都已经足够困难了。

If we create a separate class for each of the responsibilities listed above, including the thread management responsibility, then when we change the thread management strategy, the change will impact less overall code and will not pollute the other responsibilities. This also makes it much easier to test all the other responsibilities without having to worry about threading. Here is an updated version that does just that:

> 如果为上述每个权责（包括线程管理权责在内）创建单独的类，当改动线程管理策略时，就会对整个代码产生较小影响，不至于污染其他权责。这样一来，也能在不担心线程问题的前提下测试所有其他权责。下面是修改过的版本：

```java
public void run() {
    while (keepProcessing) {
        try {
        ClientConnection clientConnection = connectionManager.awaitClient();
        ClientRequestProcessor requestProcessor
            = new ClientRequestProcessor(clientConnection);
        clientScheduler.schedule(requestProcessor);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    connectionManager.shutdown();
}
```

This now focuses all things thread-related into one place, clientScheduler. If there are concurrency problems, there is just one place to look:

> 所有与线程相关的东西都放到了 clientScheduler 里面。如果出现并发问题，只要看这个地方就好了：

```java
public interface ClientScheduler {
    void schedule(ClientRequestProcessor requestProcessor);
}
```

The current policy is easy to implement:

> 并发策略易于实现：

```java
public class ThreadPerRequestScheduler implements ClientScheduler {
    public void schedule(final ClientRequestProcessor requestProcessor) {
        Runnable runnable = new Runnable() {
            public void run() {
                requestProcessor.process();
            }
        };

        Thread thread = new Thread(runnable);
        thread.start();
    }
}
```

Having isolated all the thread management into a single place, it is much easier to change the way we control threads. For example, moving to the Java 5 Executor framework involves writing a new class and plugging it in (Listing A-1).

> 把所有线程管理隔离到一个位置，修改控制线程的方式就容易多了。例如，移植到 Java 5 Executor 框架就只需要编写一个新类并插进来即可（如代码清单 A-1 所示）。

Listing A-1 ExecutorClientScheduler.java

> 代码清单 A-1 ExecutorClientScheduler.java

```java
import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

public class ExecutorClientScheduler implements ClientScheduler {
    Executor executor;

    public ExecutorClientScheduler(int availableThreads) {
        executor = Executors.newFixedThreadPool(availableThreads);
    }

    public void schedule(final ClientRequestProcessor requestProcessor) {
        Runnable runnable = new Runnable() {
            public void run() {
                requestProcessor.process();
            }
        };
        executor.execute(runnable);
    }
}
```

### A.1.4 Conclusion 小结

Introducing concurrency in this particular example demonstrates a way to improve the throughput of a system and one way of validating that throughput through a testing framework. Focusing all concurrency code into a small number of classes is an example of applying the Single Responsibility Principle. In the case of concurrent programming, this becomes especially important because of its complexity.

> 本例介绍的并发编程，演示了一种提高系统吞吐量的方法，以及一种通过测试框架验证吞吐量的方法。将全部并发代码放到少数类中，是应用单一权责原则的范例。对于并发编程，因其复杂性，这一点尤其重要。

## A.2 POSSIBLE PATHS OF EXECUTION 执行的可能路径

Review the method incrementValue, a one-line Java method with no looping or branching:

> 复查没有循环或条件分支的单行 Java 方法 incrementValue：

```java
public class IdGenerator {
    int lastIdUsed;

    public int incrementValue() {
    return ++lastIdUsed;
    }
}
```

Ignore integer overflow and assume that only one thread has access to a single instance of IdGenerator. In this case there is a single path of execution and a single guaranteed result:

> 忽略整数溢出的情形，假定只有单个线程能访问 IdGenerator 的单个实体。这种情况下，只有一种执行路径和一个确定的结果：

- The value returned is equal to the value of lastIdUsed, both of which are one greater than just before calling the method.

> - 返回值等于 lastIdUsed 的值，两者都比调用方法前大 1。

What happens if we use two threads and leave the method unchanged? What are the possible outcomes if each thread calls incrementValue once? How many possible paths of execution are there? First, the outcomes (assume lastIdUsed starts with a value of 93):

> 如果使用两个线程、不修改方法的话会发生什么？如果每个线程都调用一次 incrementValue，可能得到什么结果呢？有多少种可能执行路径？首先来看结果（假定 lastIdUsed 初始值为 93）：

- Thread 1 gets the value of 94, thread 2 gets the value of 95, and lastIdUsed is now 95.
- Thread 1 gets the value of 95, thread 2 gets the value of 94, and lastIdUsed is now 95.
- Thread 1 gets the value of 94, thread 2 gets the value of 94, and lastIdUsed is now 94.

---

> - 线程 1 得到 94，线程 2 得到 95，lastIdUsed 为 95；
> - 线程 1 得到 95，线程 2 得到 94，lastIdUsed 为 95；
> - 线程 1 得到 94，线程 2 得到 94，lastIdUsed 为 94。

The final result, while surprising, is possible. To see how these different results are possible, we need to understand the number of possible paths of execution and how the Java Virtual Machine executes them.

> 最后一个结果尽管令人吃惊，也是有可能出现的。要想明白为何可能出现这些结果，就需要理解可能执行路径的数量以及 Java 虚拟机是如何执行这些路径的。

### A.2.1 Number of Paths 路径数量

To calculate the number of possible execution paths, we’ll start with the generated byte-code. The one line of java (return ++lastIdUsed;) becomes eight byte-code instructions. It is possible for the two threads to interleave the execution of these eight instructions the way a card dealer interleaves cards as he shuffles a deck.2 Even with only eight cards in each hand, there are a remarkable number of shuffled outcomes.

> 为了算出可能执行路径的数量，我们从生成的字节码开始研究。那行 Java 代码（return++lastIdUsed;）变成了 8 个字节码指令。两个线程有可能交错执行这 8 个指令，就像庄家在洗牌时交错牌张一样。即便每只手上只有 8 张牌，洗牌得到的结果数量也很可观。

For this simple case of N instructions in a sequence, no looping or conditionals, and T threads, the total number of possible execution paths is equal to

> 对于指令系列中有 N 个指令和 T 个线程、没有循环或条件分支的简单情况，总的可能执行路径数量等于

![](figures/apA/322equ01.jpg)

Calculating the Possible Orderings

> 计算可能执行次序

This comes from an email from Uncle Bob to Brett:

> 以下摘自鲍勃大叔给 Brett 的一封电子邮件：

With N steps and T threads there are `T * N` total steps. Prior to each step there is a context switch that chooses between the T threads. Each path can thus be represented as a string of digits denoting the context switches. Given steps A and B and threads 1 and 2, the six possible paths are 1122, 1212, 1221, 2112, 2121, and 2211. Or, in terms of steps it is A1B1A2B2, A1A2B1B2, A1A2B2B1, A2A1B1B2, A2A1B2B1, and A2B2A1B1. For three threads the sequence is 112233, 112323, 113223, 113232, 112233, 121233, 121323, 121332, 123132, 123123, ….

> 对于 N 步指令和 T 个线程，总共有`T*N`个步骤。在执行每步指令之前，会有在 T 个线程中选择其一的环境开关。因而每条路径都能以一个数字字符串的形式来表示该环境开关。对于步骤 A、B 及线程 1 和 2，可能有 6 条可能路径：1122、1212、1221、2112、2121 和 2211。或者以指令步骤表示为 A1B1A2B2、A1A2B1B2、A1A2B2B1、A2A1B1B2、A2A1B2B1 及 A2B2A1B1。对于三个线程，执行序列就是 112233、112323、113223、113232、112233、121233、121323、121332、123132、123123……

One characteristic of these strings is that there must always be N instances of each T. So the string 111111 is invalid because it has six instances of 1 and zero instances of 2 and 3.

> 这些字符串的特征之一是每个 T 总会出现 N 次。所以字符串 111111 是无效的，因为里面有 6 个 1，而 2 和 3 则未出现过。

So we want the permutations of N 1’s, N 2’s, … and N T’s. This is really just the permutations of `N * T` things taken `N * T` at a time, which is (`N * T`)!, but with all the duplicates removed. So the trick is to count the duplicates and subtract that from (`N * T`)!.

> 所以要排列组合 N1、N2……直至 NT。这其实就是`N * T`对应`N * T`的排列，即(`N * T`)!，但要剔除重复的情形。所以，巧妙之处就在于计算重复次数并从(`N * T`)!中剔除掉。

Given two steps and two threads, how many duplicates are there? Each four-digit string has two 1s and two 2s. Each of those pairs could be swapped without changing the sense of the string. You could swap the 1s or the 2s both, or neither. So there are four isomorphs for each string, which means that there are three duplicates. So three out of four of the options are duplicates; alternatively one of four of the permutations are NOT duplicates. `4! * .25 = 6`. So this reasoning seems to work.

> 对于两步指令和两个线程，有多少重复呢？每个四位数字符串中都有两个 1 和两个 2。每个这种配对都可以在不影响字符串意义的前提下调换。可以同时调换全部 1 和 2，也可以都不调换。所以每个字符串就有四种同构形态，即存在 3 次重复。所以四分之三的路径是重复的；而四分之一的排列则不重复。`4!*.25=6`。这样计算看来可行。

How many duplicates are there? In the case where N = 2 and T = 2, I could swap the 1s, the 2s, or both. In the case where N = 2 and T = 3, I could swap the 1s, the 2s, the 3s, 1s and 2s, 1s and 3s, or 2s and 3s. Swapping is just the permutations of N. Let’s say there are P permutations of N. The number of different ways to arrange those permutations are `P**T`.

> 有多少重复呢？对于 N=1 且 T=2 的情形，我可以调换 1，调换 2，或两者都调换。对于 N=2 且 T=3 的情形，我可以调换 1、2、3，1 和 2，1 和 3，或 2 和 3。调换只是 N 的排列组合罢了。设有 N 的 P 种排列组合。排列组合的方式总共有`P**T`种。

So the number of possible isomorphs is `N!**T`. And so the number of paths is `(T*N)!/(N!**T)`. Again, in our T = 2, N = 2 case we get 6 (24/4).

> 所以可能的同构形态数量为`N!**T`。路径的数量就是`(T*N)!/(N!**T)`。对于 T=2 且 N=2 的情况，结果就是 6（即 24/4）。

For N = 2 and T = 3 we get 720/8 = 90.

> 对于 N=2 且 T=3，结果是 720/8=90。

For N = 3 and T = 3 we get 9!/6^3 = 1680.

> 对于 N=3 且 T=3，结果是 9!/6^3=1680。

For our simple case of one line of Java code, which equates to eight lines of byte-code and two threads, the total number of possible paths of execution is 12,870. If the type of lastIdUsed is a long, then every read/write becomes two operations instead of one, and the number of possible orderings becomes 2,704,156.

> 对于一行 Java 代码（等同于 8 行字节码）和两个线程的简单情况，可能执行路径的总数量就是 12870。如果 lastIdUsed 的类型为 long，每次读/写操作都变成了两次操作，而可能的次序高达 2704156 种。

What happens if we make one change to this method?

> 如果改动一下该方法会怎样？

```java
public synchronized void incrementValue() {
    ++lastIdUsed;
}
```

The number of possible execution pathways becomes two for two threads and N! in the general case.

> 这样一来，对于两个线程的情况，可能执行路径的数量就是 2，即 N!。

### A.2.2 Digging Deeper 深入挖掘

What about the surprising result that two threads could both call the method once (before we added synchronized) and get the same numeric result? How is that possible? First things first.

> 两个线程都调用方法一次（在添加 synchronize 之前）、得到同一结果数字的惊异结果又怎样呢？怎么可能出现这种情况？一样一样来。

What is an atomic operation? We can define an atomic operation as any operation that is uninterruptable. For example, in the following code, line 5, where 0 is assigned to lastid, is atomic because according to the Java Memory model, assignment to a 32-bit value is uninterruptable.

> 什么是原子操作？可以把原子操作定义为不可中断的操作。例如，在下列代码的第 5 行，0 被赋值给 lastid，就是一个原子操作。因为依据 Java 内存模型，32 位值的赋值操作是不可中断的。

```java
01: public class Example {
02:    int lastId;
03:
04:    public void resetId() {
05:        value = 0;
06:    }
07:
08:    public int getNextId() {
09:        ++value;
10:    }
11:}
```

What happens if we change type of lastId from int to long? Is line 5 still atomic? Not according to the JVM specification. It could be atomic on a particular processor, but according to the JVM specification, assignment to any 64-bit value requires two 32-bit assignments. This means that between the first 32-bit assignment and the second 32-bit assignment, some other thread could sneak in and change one of the values.

> 如果把 lastId 的类型从 int 改为 long 会怎样？第 5 行还是原子操作吗？如果不考虑 JVM 规约，则有可能根据处理器不同而不同。不过，根据 JVM 规约，64 位值的赋值需要两次 32 位赋值。这意味着在第一次和第二次 32 位赋值之间，其他线程可能插进来，修改其中一个值。

What about the pre-increment operator, ++, on line 9? The pre-increment operator can be interrupted, so it is not atomic. To understand, let’s review the byte-code of both of these methods in detail.

> 第 9 行的前递增操作符++又怎样呢？前递增操作符可以被中断，所
> 以它不是原子的。为了理解这点，仔细复查一下这些方法的字节码吧。

Before we go any further, here are three definitions that will be important:

> 在更进一步之前，有三个重要的定义：

- Frame—Every method invocation requires a frame. The frame includes the return address, any parameters passed into the method and the local variables defined in the method. This is a standard technique used to define a call stack, which is used by modern languages to allow for basic function/method invocation and to allow for recursive invocation.
- Local variable—Any variables defined in the scope of the method. All nonstatic methods have at least one variable, this, which represents the current object, the object that received the most recent message (in the current thread), which caused the method invocation.
- Operand stack—Many of the instructions in the Java Virtual Machine take parameters. The operand stack is where those parameters are put. The stack is a standard last-in, first-out (LIFO) data structure.

---

> - 框架——每个方法调用都需要一个框架。该框架包括返回地址、传入方法的参数，以及方法中定义的本地变量。这是定义一个调用堆栈的标准技术，现代编程语言用来实现基本函数/方法调用和递归调用；
> - 本地变量——方法作用范围内定义的每个变量。所有非静态方法至少有一个变量 this，代表当前对象，即接收导致方法调用的（当前线程内）大多数最新消息的对象；
> - 运算对象栈——Java 虚拟机中的许多指令都有参数。运算对象栈是放置参数的地方。堆栈是个标准的后入先出（LIFO）数据结构。

Here is the byte-code generated for resetId():

> 下面是 restId() 的字节码，如表 A-1 所示。

![](figures/apA/0324tab01.jpg)

![](figures/apA/0325tab01.jpg)

These three instructions are guaranteed to be atomic because, although the thread executing them could be interrupted after any one of them, the information for the PUTFIELD instruction (the constant value 0 on the top of the stack and the reference to this one below the top, along with the field value) cannot be touched by another thread. So when the assignment occurs, we are guaranteed that the value 0 will be stored in the field value. The operation is atomic. The operands all deal with information local to the method, so there is no interference between multiple threads.

> 这三个指令确保是原子的，因为尽管执行它们的线程可能在其中任何一个指令后被打断，但 PUTFIELD 指令（堆栈顶部的常量值 0 和顶端之下的 this 引用及其字段值）的信息并不能为其他线程所触及。所以，当赋值操作发生时，值 0 一定将存储到字段值中。该操作是原子的。操作对象都处理对于方法而言是本地的信息，故在多个线程之间并无冲突。

So if these three instructions are executed by ten threads, there are 4.38679733629e+24 possible orderings. However, there is only one possible outcome, so the different orderings are irrelevant. It just so happens that the same outcome is guaranteed for longs in this case as well. Why? All ten threads are assigning a constant value. Even if they interleave with each other, the end result is the same.

> 所以，如果这三个指令由 10 个线程执行，就会有 4.38679733629e+24 种可能的执行次序。不过，只会有一种可能的结果，所以执行次序不同无关紧要。对于本例中的 long 常量，总是有同一种运算结果。为什么？因为 10 个线程的赋值操作都是针对一个常量的。即便它们互相干涉，结果也是一样。

With the ++ operation in the getNextId method, there are going to be problems. Assume that lastId holds 42 at the beginning of this method. Here is the byte-code for this new method:

> 方法 getNextId 中的++操作就会有问题了。假定 lastId 在方法开始时的值为 42.下面是新方法的字节码，如表 A-2 所示。

![](figures/apA/0325tab02.jpg)

Imagine the case where the first thread completes the first three instructions, up to and including GETFIELD, and then it is interrupted. A second thread takes over and performs the entire method, incrementing lastId by one; it gets 43 back. Then the first thread picks up where it left off; 42 is still on the operand stack because that was the value of lastId when it executed GETFIELD. It adds one to get 43 again and stores the result. The value 43 is returned to the first thread as well. The result is that one of the increments is lost because the first thread stepped on the second thread after the second thread interrupted the first thread.

> 设想第一个线程完成了前三个操作，直到执行完 GETFIELD，然后被打断。第二个线程接手并完成整个方法调用，lastId 的值递增 1；得到的值为 43。第一个线程再从中断处继续执行；操作对象栈中的值还是 42，因为那就是该线程执行 GETFIELD 时的 lastId 值。线程给 lastId 加 1，得到 43，存储这个结果。第一个线程也得到了值 43。结果就是其中一个递增操作丢失了，因为第一个线程在被第二个线程打断后又踏入了第二个线程中。

Making the getNexId() method synchronized fixes this problem.

> 将 getNextId( )方法修改为同步方法就能修正这个问题。

### A.2.3 Conclusion 小结

An intimate understanding of byte-code is not necessary to understand how threads can step on each other. If you can understand this one example, it should demonstrate the possibility of multiple threads stepping on each other, which is enough knowledge.

> 理解线程之间如何互相干涉，并不一定要精通字节码。如果你能看明白这个例子，它应该已经展示了多个线程之间互相干涉的可能性，这已经足够了。

That being said, what this trivial example demonstrates is a need to understand the memory model enough to know what is and is not safe. It is a common misconception that the ++ (pre- or post-increment) operator is atomic, and it clearly is not. This means you need to know:

> 这个小例子说明，有必要尽量理解内存模型，明白什么是安全的，什么是不安全的。有一种普遍的误解，认为++（前递增或后递增）操作符是原子的，其实并非如此。你必须知道：

- Where there are shared objects/values
- The code that can cause concurrent read/update issues
- How to guard such concurrent issues from happening

---

> - 什么地方有共享对象/值；
> - 哪些代码会导致并发读/写问题；
> - 如何防止这种并发问题发生。

## A.3 KNOWING YOUR LIBRARY 了解类库

### A.3.1 Executor Framework Executor 框架

As demonstrated in the ExecutorClientScheduler.java on page 321, the Executor framework introduced in Java 5 allows for sophisticated execution using thread pools. This is a class in the java.util.concurrent package.

> 如前文 ExecutorClientScheduler.java 所演示的那样，Java 5 中引入的 Executor 框架支持利用线程池进行复杂的执行。那就是 java.util.concurrent 包中的一个类。

If you are creating threads and are not using a thread pool or are using a hand-written one, you should consider using the Executor. It will make your code cleaner, easier to follow, and smaller.

> 如果在创建线程时没有使用线程池或自行编写线程池，可以考虑使用 Executor。它能让代码更整洁，易于理解，且更加短小。

The Executor framework will pool threads, resize automatically, and recreate threads if necessary. It also supports futures, a common concurrent programming construct. The Executor framework works with classes that implement Runnable and also works with classes that implement the Callable interface. A Callable looks like a Runnable, but it can return a result, which is a common need in multithreaded solutions.

> Executor 框架将把线程放到池中，自动调整其大小，并在必要时重建线程。它还支持 future，一种通用的并发编程构造。Executor 能与实现了 Runnable 的类协同工作，也能与实现了 Callable 接口的类协同工作。Callback 看来就像是 Runnable，但它能返回一个结果，那在多线程解决方案中是普遍的需求。

A future is handy when code needs to execute multiple, independent operations and wait for both to finish:

> 当代码需要执行多个相互独立的操作并等待这些操作结束时，future 刚好就手：

```java
public String processRequest(String message) throws Exception {
    Callable<String> makeExternalCall = new Callable<String>() {

        public String call() throws Exception {
            String result = “”;
            // make external request
            return result;
        }
    };

    Future<String> result = executorService.submit(makeExternalCall);
    String partialResult = doSomeLocalProcessing();
    return result.get() + partialResult;
}
```

In this example, the method starts executing the makeExternalCall object. The method continues other processing. The final line calls result.get(), which blocks until the future completes.

> 在本例中，方法开始执行 makeExternalCall 对象。然后该方法继续其他操作。最后一行代码调用 result.get()，在 future 代码执行完成前，这个操作是锁定的。

### A.3.2 Nonblocking Solutions 非锁定的解决方案

The Java 5 VM takes advantage of modern processor design, which supports reliable, nonblocking updates. Consider, for example, a class that uses synchronization (and therefore blocking) to provide a thread-safe update of a value:

> Java5 虚拟机利用了现代处理器支持可靠、非锁定更新的设计优点。例如，考虑某个使用同步（从而也是锁定的）来提供线程安全地更新一个值的类：

```java
public class ObjectWithValue {
    private int value;
    public void synchronized incrementValue() { ++value; }
    public int getValue() { return value; }
}
```

Java 5 has a series of new classes for situations like this: AtomicBoolean, AtomicInteger, and AtomicReference are three examples; there are several more. We can rewrite the above code to use a nonblocking approach as follows:

> Java5 有一系列用于此类情况的新类，例如 AtomicBoolean、 AtomicInteger 和 AtomicReference 等；还有另外一些。我们可以重写上面的代码，使用非锁定的手段，如下所示：

```java
public class ObjectWithValue {
    private AtomicInteger value = new AtomicInteger(0);

    public void incrementValue() {
    value.incrementAndGet();
    }
    public int getValue() {
        return value.get();
    }
}
```

Even though this uses an object instead of a primitive and sends messages like incrementAndGet() instead of ++, the performance of this class will nearly always beat the previous version. In some cases it will only be slightly faster, but the cases where it will be slower are virtually nonexistent.

> 即便使用了对象而非直接操作，使用了 incrementAndGet() 这样的信息发送方式而非++操作，这个类的性能还是几乎总能胜过上一版本。在某些情况下只会快一点点，但较慢的情形却几乎不存在。

How is this possible? Modern processors have an operation typically called Compare and Swap (CAS). This operation is analogous to optimistic locking in databases, whereas the synchronized version is analogous to pessimistic locking.

> 怎么会这样？现代处理器拥有一种通常称为比较交换（Compare and Swap，CAS）的操作。这种操作类似于数据库中的乐观锁定，而其同步版本则类似于保守锁定。

The synchronized keyword always acquires a lock, even when a second thread is not trying to update the same value. Even though the performance of intrinsic locks has improved from version to version, they are still costly.

> 关键字 synchronized 总是要求上锁，即便第二个线程并不更新同一值时也如此。尽管这种固有锁的性能一直在提升，但仍然代价昂贵。

The nonblocking version starts with the assumption that multiple threads generally do not modify the same value often enough that a problem will arise. Instead, it efficiently detects whether such a situation has occurred and retries until the update happens successfully. This detection is almost always less costly than acquiring a lock, even in moderate to high contention situations.

> 非上锁的版本假定多个线程通常并不频繁修改同一个值，导致问题产生。它高效地侦测这种情形是否发生，并不断尝试，直至更新成功。这种侦测行为几乎总是比上锁来得划算，在争用激烈的情况下也是如此。

How does the Virtual Machine accomplish this? The CAS operation is atomic. Logically, the CAS operation looks something like the following:

> 虚拟机如何实现这种机制？CAS 的操作是原子的。逻辑上，CAS 操作看起来像这样：

```java
int variableBeingSet;

void simulateNonBlockingSet(int newValue) {
    int currentValue;
    do {
        currentValue = variableBeingSet
    } while(currentValue != compareAndSwap(currentValue, newValue));
}

int synchronized compareAndSwap(int currentValue, int newValue) {
    if(variableBeingSet == currentValue) {
        variableBeingSet = newValue;
        return currentValue;
    }
    return variableBeingSet;
}
```

When a method attempts to update a shared variable, the CAS operation verifies that the variable getting set still has the last known value. If so, then the variable is changed. If not, then the variable is not set because another thread managed to get in the way. The method making the attempt (using the CAS operation) sees that the change was not made and retries.

> 当某个方法试图更新一个共享变量，CAS 操作就会验证要赋值的变量是否保有上一次的已知值。如果是，就修改变量值。如果不是，则不会碰变量，因为另一个线程正在试图更新变量值。要更新数据的方法（通过 CAS 操作）查看是否修改并持续尝试。

### A.3.3 Nonthread-Safe Classes 非线程安全类

There are some classes that are inherently not thread safe. Here are a few examples:

> 有些类天生不是线程安全的。下面是几个例子：

- SimpleDateFormat
- Database Connections
- Containers in java.util
- Servlets

---

> - SimpleDateFormat
> - 数据库连接
> - java.util 中的容器
> - Servlet

Note that some collection classes have individual methods that are thread-safe. However, any operation that involves calling more than one method is not. For example, if you do not want to replace something in a HashTable because it is already there, you might write the following code:

> 注意，有些群集类拥有一些线程安全的方法。不过，涉及调用多个方法的操作都不是线程安全的。例如，如果因为 HashTable 中已经有某物而不打算替换它，可能会写出以下代码：

```java
if(!hashTable.containsKey(someKey)) {
    hashTable.put(someKey, new SomeValue());
}
```

Each individual method is thread-safe. However, another thread might add a value in between the containsKey and put calls. There are several options to fix this problem.

> 单个方法是线程安全的。不过，另一个线程却可能在 containsKey 和 put 调用之间塞进一个值。有几种修正这个问题的手段。

- Lock the HashTable first, and make sure all other users of the HashTable do the same—client-based locking:

> - 先锁定 HashTable，确定其他使用者都做了基于客户端的锁定：

```java
synchronized(map) {
if(!map.conainsKey(key))
    map.put(key, value);
}
```

- Wrap the HashTable in its own object and use a different API—server-based locking using an ADAPTER:

> - 用其对象包装 HashTable，并使用不同的 API——利用 ADAPTER 模式做基于服务端的锁定：

```java
public class WrappedHashtable<K, V> {
    private Map<K, V> map = new Hashtable<K, V>();

    public synchronized void putIfAbsent(K key, V value) {
        if (map.containsKey(key))
            map.put(key, value);
    }
}
```

- Use the thread-safe collections:

> - 采用线程安全的群集：

```java
ConcurrentHashMap<Integer, String> map = new ConcurrentHashMap<Integer, String>();
map.putIfAbsent(key, value);
```

The collections in java.util.concurrent have operations like putIfAbsent() to accommodate such operations.

> 在 java.util.concurrent 中的群集都有 putIfAbsent() 之类提供这种操作的方法。

## A.4 DEPENDENCIES BETWEEN METHODS CAN BREAK CONCURRENT CODE 方法之间的依赖可能破坏并发代码

Here is a trivial example of a way to introduce dependencies between methods:

> 以下是一个有关在方法间引入依赖的小例子：

```java
public class IntegerIterator implements Iterator<Integer> {
    private Integer nextValue = 0;

    public synchronized boolean hasNext() {
        return nextValue < 100000;
    }
    public synchronized Integer next() {
        if (nextValue == 100000)
            throw new IteratorPastEndException();
        return nextValue++;
    }
    public synchronized Integer getNextValue() {
        return nextValue;
    }
}
```

Here is some code to use this IntegerIterator:

> 下面是使用 IntegerIterator 的代码：

```java
IntegerIterator iterator = new IntegerIterator();
while(iterator.hasNext()) {
    int nextValue = iterator.next();
    // do something with nextValue
}
```

If one thread executes this code, there will be no problem. But what happens if two threads attempt to share a single instance of IngeterIterator with the intent that each thread will process the values it gets, but that each element of the list is processed only once? Most of the time, nothing bad happens; the threads happily share the list, processing the elements they are given by the iterator and stopping when the iterator is complete. However, there is a small chance that, at the end of the iteration, the two threads will interfere with each other and cause one thread to go beyond the end of the iterator and throw an exception.

> 如果只有一个线程执行这段代码，不会有什么问题。但如果有两个线程抱着每个线程都处理它获得的值、但列表中的每个元素都只被处理一次的意图，尝试共享 IntegerIterator 的单个实体，会发生什么事？多数时候什么也不会发生；线程开心地共享着列表，处理从迭代器获取的元素，在迭代器完成执行时停下。然而，在迭代的末尾，两个线程也有少量可能互相干涉，导致其中一个超出迭代器末尾，抛出异常。

Here’s the problem: Thread 1 asks the question hasNext(), which returns true. Thread 1 gets preempted and then Thread 2 asks the same question, which is still true. Thread 2 then calls next(), which returns a value as expected but has a side effect of making hasNext() return false. Thread 1 starts up again, thinking hasNext() is still true, and then calls next(). Even though the individual methods are synchronized, the client uses two methods.

> 问题在这里。线程 1 调用 hasNext( )方法，该方法返回 true。线程 1 占先，然后线程 2 也调用这个方法，同样返回 true。线程 2 接着调用 next( )，该方法如期返回一个值，但副作用是之后再调用 hasNext( )就会返回 false。线程 1 继续执行，以为 hasNext( )还是 true，然后调用 next( )。即便单个方法是同步的，客户端还是使用了两个方法。

This is a real problem and an example of the kinds of problems that crop up in concurrent code. In this particular situation this problem is especially subtle because the only time where this causes a fault is when it happens during the final iteration of the iterator. If the threads happen to break just right, then one of the threads could go beyond the end of the iterator. This is the kind of bug that happens long after a system has been in production, and it is hard to track down.

> 这的确是个问题，也是并发代码中此类问题的典型例子。在这个特殊例子中，问题尤其隐蔽，因为只有在迭代器最后一次迭代时发生才会导致错误。如果线程刚好在那个点中断，其中一个线程就可能超出迭代器末尾。这类错误往往在系统部署之后很久才发生，而且很难追踪。

You have three options:

> 出现错误时，你有 3 种做法。

- Tolerate the failure.
- Solve the problem by changing the client: client-based locking
- Solve the problem by changing the server, which additionally changes the client: server-based locking

---

> - 容忍错误；
> - 修改客户代码解决问题：基于客户代码的锁定；
> - 修改服务端代码解决问题，同时也修改了客户代码：基于服务端的锁定。

### A.4.1 Tolerate the Failure 容忍错误

Sometimes you can set things up such that the failure causes no harm. For example, the above client could catch the exception and clean up. Frankly, this is a bit sloppy. It’s rather like cleaning up memory leaks by rebooting at midnight.

> 有时，可以通过一些设置让错误不会导致损害。例如，上述客户代码可以捕捉并清理异常。坦白地说，这有点草草从事，就像是半夜重启解决内存泄露问题一样。

### A.4.2 Client-Based Locking 基于客户代码的锁定

To make IntegerIterator work correctly with multiple threads, change this client (and every other client) as follows:

> 要让 IntegerIterator 在多线程情况下正确运行，对客户代码做如下修改：

```java
IntegerIterator iterator = new IntegerIterator();

while (true) {
    int nextValue;
    synchronized (iterator) {
        if (!iterator.hasNext())
            break;
        nextValue = iterator.next();
    }
    doSometingWith(nextValue);
}
```

Each client introduces a lock via the synchronized keyword. This duplication violates the DRY principle, but it might be necessary if the code uses non-thread-safe third-party tools.

> 每个客户端都通过 synchronized 关键字引入一个锁。这种重复违反了 DRY 原则，但如果代码使用非线程安全的第三方工具，可能必须这样做。

This strategy is risky because all programmers who use the server must remember to lock it before using it and unlock it when done. Many (many!) years ago I worked on a system that employed client-based locking on a shared resource. The resource was used in hundreds of different places throughout the code. One poor programmer forgot to lock the resource in one of those places.

> 这种策略有风险，因为使用服务端的程序员都得记住在使用前上锁、用过后解锁。许多（许多！）年前，我遇到过一个在共享资源上应用基于客户代码锁定的系统。代码中有几百处用到这个资源的地方。有位可怜的程序员忘记在其中一处做资源锁定。

The system was a multi-terminal time-sharing system running accounting software for Local 705 of the trucker’s union. The computer was in a raised-floor, environment-controlled room 50 miles north of the Local 705 headquarters. At the headquarters they had dozens of data entry clerks typing union dues postings into the terminals. The terminals were connected to the computer using dedicated phone lines and 600bps half-duplex modems. (This was a very, very long time ago.)

> 该系统是个多终端分时系统，为 Local 705 卡车司机联盟运行会计软件。计算机放在距 Local 705 总部 50 英里（约 84.65km）以北的一间镶有高于地面的地板、环境可控的机房中。总部有几十位数据录入员，往终端输入记录。终端使用电话专线和 600bit/s 的半双工调制解调器连接到计算机。（这可是很久很久以前的事了。）

About once per day, one of the terminals would “lock up.” There was no rhyme or reason to it. The lock up showed no preference for particular terminals or particular times. It was as though there were someone rolling dice choosing the time and terminal to lock up. Sometimes more than one terminal would lock up. Sometimes days would go by without any lock-ups.

> 每天大概都会有一台终端毫无理由地“死锁”。死锁也不限定在某些终端或特定时间。就像是有人掷骰子选择死锁的时机和终端一般。有时，会有几台终端死锁。有时，好几天都不出现死锁情况。

At first the only solution was a reboot. But reboots were tough to coordinate. We had to call the headquarters and get everyone to finish what they were doing on all the terminals. Then we could shut down and restart. If someone was doing something important that took an hour or two, the locked up terminal simply had to stay locked up.

> 刚开始，唯一的解决手段就是重启。但协同起来很不便。我们得打电话给总部，让大家都完成在终端上的工作。然后我们才能关机、重启。如果有人在做要花上一两个小时才能做完的事，被锁定的终端就只能一直等着。

After a few weeks of debugging we found that the cause was a ring-buffer counter that had gotten out of sync with its pointer. This buffer controlled output to the terminal. The pointer value indicated that the buffer was empty, but the counter said it was full. Because it was empty, there was nothing to display; but because it was also full, nothing could be added to the buffer to be displayed on the screen.

> 经过几个星期的调试，我们发现，原因在于一个指针不同步的环形缓冲区计数器。该缓冲区控制向终端的输出。指针值说明缓冲区是空的，但计数器却指出缓冲区是满的。因为缓冲区是空的，就没什么可显示；但因为缓冲区也是满的，也就无法向其中加入可在屏幕上显示的内容。

So we knew why the terminals were locking, but we didn’t know why the ring buffer was getting out of sync. So we added a hack to work around the problem. It was possible to read the front panel switches on the computer. (This was a very, very, very long time ago.) We wrote a little trap function that detected when one of these switches was thrown and then looked for a ring buffer that was both empty and full. If one was found, it reset that buffer to empty. Voila! The locked-up terminal(s) started displaying again.

> 我们知道了终端为何会死锁，但却不知道为什么环形缓冲区会不同步。我们用了点手段发现问题所在。当时程序能够读取计算机的前面板开关状态（这可是很久很久以前的事了）。我们写了个陷阱程序，侦测这些开关何时被拨动，然后查找既空又满的环形缓冲区。如果找到，就重置该缓冲区为空。乌拉！锁定的终端又重新开始显示了。

So now we didn’t have to reboot the system when a terminal locked up. The Local would simply call us and tell us we had a lock-up, and then we just walked into the computer room and flicked a switch.

> 这样，在终端锁定时就不必重启系统了。客户只需要打电话告诉我们出现死锁，我们就径直走到机房，拨动一下开关即可。

Of course sometimes they worked on the weekends, and we didn’t. So we added a function to the scheduler that checked all the ring buffers once per minute and reset any that were both empty and full. This caused the displays to unclog before the Local could even get on the phone.

> 当然，有时他们会在周末加班，但是我们可不加班。所以我们又在计划列表中添加了一个函数，每分钟检查一次全部环形缓冲区，重置既空又满的缓冲区。在客户打电话之前，显示就已经恢复正常了。

It was several more weeks of poring over page after page of monolithic assembly language code before we found the culprit. We had done the math and calculated that the frequency of the lock-ups was consistent with a single unprotected use of the ring buffer. So all we had to do was find that one faulty usage. Unfortunately, this was so very long ago that we didn’t have search tools or cross references or any other kind of automated help. We simply had to pore over listings.

> 在发现问题原因之前，我们花了好几个星期查看一页又一页的单片机汇编语言代码。我们已经完成计算，算出死锁的频率是周期性的，而且其中有一处未受保护的环形缓冲区使用。所以，剩下的任务就是找出那个错误的用法。不幸这是多年以前的事，那时既没有搜索工具，也没有交叉引用或任何其他自动化帮助手段。我们只能细查代码清单。

I learned an important lesson that cold Chicago winter of 1971. Client-based locking really blows.

> 在芝加哥 1971 年的寒冬，我学到了重要的一课。基于客户代码的锁定实在不可靠。

### A.4.3 Server-Based Locking 基于服务端的锁定

The duplication can be removed by making the following changes to IntegerIterator:

> 按照以下方式修改 IntegerIterator 也能消除重复：

```java
public class IntegerIteratorServerLocked {
    private Integer nextValue = 0;
    public synchronized Integer getNextOrNull() {
        if (nextValue < 100000)
            return nextValue++;
        else
            return null;
    }
}
```

And the client code changes as well:

> 客户代码也要修改：

```java
while (true) {
    Integer nextValue = iterator.getNextOrNull();
    if (next == null)
        break;
    // do something with nextValue
}
```

In this case we actually change the API of our class to be multithread aware.3 The client needs to perform a null check instead of checking hasNext().

> 在这种情形下，我们实际上是修改了类的 API，使其能适应多线程。客户端需要做 null 检查，而不是检查 hasNext()。

In general you should prefer server-based locking for these reasons:

> 通常你应该选用基于服务端的锁定，因为：

- It reduces repeated code—Client-based locking forces each client to lock the server properly. By putting the locking code into the server, clients are free to use the object and not worry about writing additional locking code.
- It allows for better performance—You can swap out a thread-safe server for a non-thread safe one in the case of single-threaded deployment, thereby avoiding all overhead.
- It reduces the possibility of error—All it takes is for one programmer to forget to lock properly.
- It enforces a single policy—The policy is in one place, the server, rather than many places, each client.
- It reduces the scope of the shared variables—The client is not aware of them or how they are locked. All of that is hidden in the server. When things break, the number of places to look is smaller.

---

> - 它减少了重复代码——采用基于客户代码的锁定，每个客户端都要正确锁定服务端。把锁定代码放到服务端，客户端就能自由使用对象，不必费心编写额外的锁定代码；
> - 它提升了性能——在单线程部署中，可以用非多线程安全服务端代码替代线程安全客户端，从而省去花销；
> - 它减少了出错的可能性——只会有一个程序员忘记上锁；
> - 它执行了单一策略——该策略只在服务端这一处地方实施，而不是在许多地方（每个客户端）实施；
> - 它缩减了共享变量的作用范围——客户端不必关心它们或它们是如何锁定的。一切都隐藏在服务端。如果出错，要侦查的范围就小多了。

What if you do not own the server code?

> 如果你无法修改服务端代码又该如何？

- Use an ADAPTER to change the API and add locking

> - 使用 ADAPTER 模式修改 API，添加锁定；

```java
public class ThreadSafeIntegerIterator {
    private IntegerIterator iterator = new IntegerIterator();

    public synchronized Integer getNextOrNull() {
        if(iterator.hasNext())
            return iterator.next();
        return null;
    }
}
```

- OR better yet, use the thread-safe collections with extended interfaces

> - 更好的方法是使用线程安全的群集和扩展接口。

## A.5 INCREASING THROUGHPUT 提升吞吐量

Let’s assume that we want to go out on the net and read the contents of a set of pages from a list of URLs. As each page is read, we will parse it to accumulate some statistics. Once all the pages are read, we will print a summary report.

> 假设我们打算连接上网，从一个 URL 列表中读取一组页面的内容。读到一个页面时，解析该页面并得到一些统计结果。读完所有页面后，打印出一份提要报表。

The following class returns the contents of one page, given a URL.

> 下面的类返回给定 URL 的页面内容：

```java
public class PageReader {
    //…
    public String getPageFor(String url) {
        HttpMethod method = new GetMethod(url);

        try {
            httpClient.executeMethod(method);
            String response = method.getResponseBodyAsString();
            return response;
        } catch (Exception e) {
            handle(e);
        } finally {
            method.releaseConnection();
        }
    }
}
```

The next class is the iterator that provides the contents of the pages based on an iterator of URLs:

> 下一个类是给出 URL 迭代器中每个页面的内容的迭代器：

```java
public class PageIterator {
    private PageReader reader;
    private URLIterator urls;

    public PageIterator(PageReader reader, URLIterator urls) {
    this.urls = urls;
    this.reader = reader;
    }

    public synchronized String getNextPageOrNull() {
    if (urls.hasNext())
        getPageFor(urls.next());
    else
        return null;
    }

    public String getPageFor(String url) {
    return reader.getPageFor(url);
    }
}
```

An instance of the PageIterator can be shared between many different threads, each one using it’s own instance of the PageReader to read and parse the pages it gets from the iterator.

> PageIterator 的一个实体可为多个不同线程共享，每个线程使用自己的 PageReader 实体读取并解析从迭代器中得到的页面。

Notice that we’ve kept the synchronized block very small. It contains just the critical section deep inside the PageIterator. It is always better to synchronize as little as possible as opposed to synchronizing as much as possible.

> 注意，我们把 synchronized 代码块的数量限制在小范围之内。它只包括深处于 PageIterator 内部的临界区。最好是尽可能少地使用同步。

### A.5.1 Single-Thread Calculation of Throughput 单线程条件下的吞吐量

Now lets do some simple calculations. For the purpose of argument, assume the following:

> 来做个简单计算。鉴于讨论的目的，假定：

- I/O time to retrieve a page (average): 1 second
- Processing time to parse page (average): .5 seconds
- I/O requires 0 percent of the CPU while processing requires 100 percent.

---

> - 获取一个页面的 I/O 时间（平均）是 1s；
> - 解析一个页面的处理时间（平均）是 0.5s；
> - I/O 操作不耗费处理器能力，而解析页面耗费 100%处理器能力。

For N pages being processed by a single thread, the total execution time is 1.5 seconds \* N. Figure A-1 shows a snapshot of 13 pages or about 19.5 seconds.

> 对于单个线程要处理的 N 个页面，总的执行时间为 1.5s\*N。图 A-1 显示了 13 个页面或大概 19.5s 的快照。

Figure A-1 Single thread

![](figures/apA/x01-1single_thread.jpg)

### A.5.2 Multithread Calculation of Throughput 多线程条件下的吞吐量

If it is possible to retrieve pages in any order and process the pages independently, then it is possible to use multiple threads to increase throughput. What happens if we use three threads? How many pages can we acquire in the same time?

> 如果能够以任意次序获得页面并独立处理页面，就有可能利用多线程提升吞吐量。如果我们使用三个线程会如何？在同一时间内能获取多少个页面呢？

As you can see in Figure A-2, the multithreaded solution allows the process-bound parsing of the pages to overlap with the I/O-bound reading of the pages. In an idealized world this means that the processor is fully utilized. Each one-second page read is overlapped with two parses. Thus, we can process two pages per second, which is three times the throughput of the single-threaded solution.

> 如你在图 A-2 中所见，多线程方案中与处理器能力有关的页面解析操作可以和与 I/O 有关的页面读取操作叠加进行。在理想状态下，这意味着处理器力尽其用。每个耗时一秒钟的页面读取操作都与两次解析操作叠加进行。这样，我们就能在每秒钟内处理两个页面，即三倍于单线程方案的吞吐量。

Figure A-2 Three concurrent threads

![](figures/apA/x01-2multi_thread.jpg)

## A.6 DEADLOCK 死锁

Imagine a Web application with two shared resource pools of some finite size:

> 想象一个拥有两个有限共享资源池的 Web 应用程序。

- A pool of database connections for local work in process storage
- A pool of MQ connections to a master repository

---

> - 一个用于本地临时工作存储的数据库连接池；
> - 一个用于连接到主存储库的 MQ 池。

Assume there are two operations in this application, create and update:

> 假定该应用中有两个操作：创建和更新。

- Create—Acquire connection to master repository and database. Talk to service master repository and then store work in local work in process database.
- Update—Acquire connection to database and then master repository. Read from work in process database and then send to the master repository

---

> - 创建——获取到主存储库和数据库的连接。与主存储库协调，并把工作保存到本地临时工作数据库；
> - 更新——先获取到数据库的连接，再获取到主存储库的连接。从临时工作数据库中读取数据，再发送给主存储库。

What happens when there are more users than the pool sizes? Consider each pool has a size of ten.

> 如果用户数量多于池的大小会怎样？假设每个池中能容纳 10 个资源。

- Ten users attempt to use create, so all ten database connections are acquired, and each thread is interrupted after acquiring a database connection but before acquiring a connection to the master repository.
- Ten users attempt to use update, so all ten master repository connections are acquired, and each thread is interrupted after acquiring the master repository but before acquiring a database connection.
- Now the ten “create” threads must wait to acquire a master repository connection, but the ten “update” threads must wait to acquire a database connection.
- Deadlock. The system never recovers.

---

> - 有 10 个用户尝试创建，获取了 10 个数据库连接，每个线程在获取到数据库连接之后、获取到主存储库连接之前都被打断；
> - 有 10 个用户尝试更新，获取了 10 个主存储库连接，每个线程在获取到主存储库连接之后、获取到数据库连接之前都会被打断；
> - 现在那 10 个“创建”线程必须等待获取主存储库连接，但那 10 个“更新”线程必须等待获取数据库连接；
> - 死锁。系统永远无法恢复。

This might sound like an unlikely situation, but who wants a system that freezes solid every other week? Who wants to debug a system with symptoms that are so difficult to reproduce? This is the kind of problem that happens in the field, then takes weeks to solve.

> 这听起来不太会出现，但谁会想要一个每隔一周就僵在那里不动的系统呢？谁想要调试出现了难以复现的症状的系统呢？这种问题突然发生，然后得花上好几个星期才能解决。

A typical “solution” is to introduce debugging statements to find out what is happening. Of course, the debug statements change the code enough so that the deadlock happens in a different situation and takes months to again occur.4

> 典型的“解决方案”是加入调试语句，发现问题。当然，调试语句对代码的修改足以令死锁在不同情况下发生，而且要几个月后才会再出现。

To really solve the problem of deadlock, we need to understand what causes it. There are four conditions required for deadlock to occur:

> 要真正地解决死锁问题，我们需要理解死锁的原因。死锁的发生需要 4 个条件：

- Mutual exclusion
- Lock & wait
- No preemption
- Circular wait

---

> - 互斥；
> - 上锁及等待；
> - 无抢先机制；
> - 循环等待。

### A.6.1 Mutual Exclusion 互斥

Mutual exclusion occurs when multiple threads need to use the same resources and those resources

> 当多个线程需要使用同一资源，且这些资源满足下列条件时，互斥就会发生。

- Cannot be used by multiple threads at the same time.
- Are limited in number.

---

> - 无法在同一时间为多个线程所用；
> - 数量上有限制。

A common example of such a resource is a database connection, a file open for write, a record lock, or a semaphore.

> 这种资源的常见例子是数据库连接、打开后用于写入的文件、记录锁或是信号量。

### A.6.2 Lock & Wait 上锁及等待

Once a thread acquires a resource, it will not release the resource until it has acquired all of the other resources it requires and has completed its work.

> 当某个线程获取一个资源，在获取到其他全部所需资源并完成其工作之前，不会释放这个资源。

### A.6.3 No Preemption 无抢先机制

One thread cannot take resources away from another thread. Once a thread holds a resource, the only way for another thread to get it is for the holding thread to release it.

> 线程无法从其他线程处夺取资源。一个线程持有资源时，其他线程获得这个资源的唯一手段就是等待该线程释放资源。

### A.6.4 Circular Wait 循环等待

This is also referred to as the deadly embrace. Imagine two threads, T1 and T2, and two resources, R1 and R2. T1 has R1, T2 has R2. T1 also requires R2, and T2 also requires R1. This gives something like Figure A-3:

> 这也被称为“死命拥抱”。想象两个线程，T1 和 T2，还有两个资源，R1 和 R2。T1 拥有 R1，T2 拥有 R2。T1 需要 R2，T2 需要 R1。如此就出现了如图 A-3 所示的情形。

Figure A-3

![](figures/apA/x01-3breaking_cycle.jpg)

All four of these conditions must hold for deadlock to be possible. Break any one of these conditions and deadlock is not possible.

> 这 4 种条件都是死锁所必需的。只要其中一个不满足，死锁就不会发生。

### A.6.5 Breaking Mutual Exclusion 不互斥

One strategy for avoiding deadlock is to sidestep the mutual exclusion condition. You might be able to do this by

> 避免死锁的一种策略是规避互斥条件。你可以：

- Using resources that allow simultaneous use, for example, AtomicInteger.
- Increasing the number of resources such that it equals or exceeds the number of competing threads.
- Checking that all your resources are free before seizing any.

---

> - 使用允许同时使用的资源，如 AtomicInteger；
> - 增加资源数量，使其等于或大于竞争线程的数量；
> - 在获取资源之前，检查是否可用。

Unfortunately, most resources are limited in number and don’t allow simultaneous use. And it’s not uncommon for the identity of the second resource to be predicated on the results of operating on the first. But don’t be discouraged; there are three conditions left.

> 不幸的是，多数资源都有上限，且不能同时使用。而且第二个资源的标识也常常要依据对第一个资源的操作结果来判断。不过别丧气，还有 3 个其他条件呢。

### A.6.6 Breaking Lock & Wait 不上锁及等待

You can also eliminate deadlock if you refuse to wait. Check each resource before you seize it, and release all resources and start over if you run into one that’s busy.

> 如果拒绝等待，就能消除死锁。在获得资源之前检查资源，如果遇到某个繁忙资源，就释放所有资源，重新来过。

This approach introduces several potential problems:

> 这种手段带来几个潜在问题：

- Starvation—One thread keeps being unable to acquire the resources it needs (maybe it has a unique combination of resources that seldom all become available).
- Livelock—Several threads might get into lockstep and all acquire one resource and then release one resource, over and over again. This is especially likely with simplistic CPU scheduling algorithms (think embedded devices or simplistic hand-written thread balancing algorithms).

---

> - 线程饥饿——某个线程一直无法获得它所需的资源（它可能需要某种很少能同时获得的资源组合）；
> - 活锁——几个线程可能会前后相连地要求获得某个资源，然后再释放一个资源，如此循环。这在单纯的 CPU 任务排列算法中尤其有可能出现（想想嵌入式设备或单纯的手写线程平衡算法）。

Both of these can cause poor throughput. The first results in low CPU utilization, whereas the second results in high and useless CPU utilization.

> 二者都能导致较差的吞吐量。第一个的结果是 CPU 利用率低，第二个的结果是较高但无用的 CPU 利用率。

As inefficient as this strategy sounds, it’s better than nothing. It has the benefit that it can almost always be implemented if all else fails.

> 尽管这种策略听起来没效率，但也好过没有。至少，如果其他方案不奏效，这种手段几乎总可以用上。

### A.6.7 Breaking Preemption 满足抢先机制

Another strategy for avoiding deadlock is to allow threads to take resources away from other threads. This is usually done through a simple request mechanism. When a thread discovers that a resource is busy, it asks the owner to release it. If the owner is also waiting for some other resource, it releases them all and starts over.

> 避免死锁的另一策略是允许线程从其他线程上夺取资源。这通常利用一种简单的请求机制来实现。当线程发现资源繁忙，就要求其拥有者释放之。如果拥有者还在等待其他资源，就释放全部资源并重新来过。

This is similar to the previous approach but has the benefit that a thread is allowed to wait for a resource. This decreases the number of startovers. Be warned, however, that managing all those requests can be tricky.

> 这和上一种手段相似，但好处是允许线程等待资源。这减少了线程重新启动的次数。不过，管理所有请求可要花点心思。

### A.6.8 Breaking Circular Wait 不做循环等待

This is the most common approach to preventing deadlock. For most systems it requires no more than a simple convention agreed to by all parties.

> 这是避免死锁的最常用手段。对于多数系统，它只要求一个为各方认同的约定。

In the example above with Thread 1 wanting both Resource 1 and Resource 2 and Thread 2 wanting both Resource 2 and then Resource 1, simply forcing both Thread 1 and Thread 2 to allocate resources in the same order makes circular wait impossible.

> 在上面的例子中线程 1 同时需要资源 1 和资源 2、线程 2 同时需要资源 2 和资源 1，只要强制线程 1 和线程 2 以同样次序分配资源，循环等待就不会发生。

More generally, if all threads can agree on a global ordering of resources and if they all allocate resources in that order, then deadlock is impossible. Like all the other strategies, this can cause problems:

> 更普遍地，如果所有线程都认同一种资源获取次序，并按照这种次序获取资源，死锁就不会发生。就像其他策略一样，这也会有问题：

- The order of acquisition might not correspond to the order of use; thus a resource acquired at the start might not be used until the end. This can cause resources to be locked longer than strictly necessary.
- Sometimes you cannot impose an order on the acquisition of resources. If the ID of the second resource comes from an operation performed on the first, then ordering is not feasible.

---

> - 获取资源的次序可能与使用资源的次序不匹配；一开始获取的资源可能在最后才会用到。这可能导致资源不必要地被长时间锁定；
> - 有时无法强求资源获取顺序。如果第二个资源的 ID 来自对第一个资源操作的结果，获取次序也无从谈起。

So there are many ways to avoid deadlock. Some lead to starvation, whereas others make heavy use of the CPU and reduce responsiveness. TANSTAAFL!5

> 有许多避免死锁的方法。有些会导致饥饿，另外一些会导致对 CPU 能力的大量耗费和降低响应率。TANSTAAFL！

Isolating the thread-related part of your solution to allow for tuning and experimentation is a powerful way to gain the insights needed to determine the best strategies.

> 将解决方案中与线程相关的部分分隔出来，再加以调整和试验，是获得判断最佳策略所需的洞见的正道。

## A.7 TESTING MULTITHREADED CODE 测试多线程代码

How can we write a test to demonstrate the following code is broken?

> 怎么才能编写显示以下代码有错的测试呢？

```java
01: public class ClassWithThreadingProblem {
02:    int nextId;
03:
04:    public int takeNextId() {
05:        return nextId++;
06:    }
07:}
```

Here’s a description of a test that will prove the code is broken:

> 下面是对能证明上列代码有错的测试的描述：

- Remember the current value of nextId.
- Create two threads, both of which call takeNextId() once.
- Verify that nextId is two more than what we started with.
- Run this until we demonstrate that nextId was only incremented by one instead of two.

> - 记住 nextId 的当前值；
> - 创建两个线程，每个都调用 takeNextId( )一次；
> - 验证 nextId 比开始时大 2；
> - 持续运行，直至发现 nextId 只比开始时大 1 为止。

Listing A-2 shows such a test:

> 代码清单 A-2 展示了这样一个测试：

Listing A-2 ClassWithThreadingProblemTest.java

> 代码清单 A-2 ClassWithThreadingProblemTest.java

```java
01: package example;
02:
03: import static org.junit.Assert.fail;
04:
05: import org.junit.Test;
06:
07: public class ClassWithThreadingProblemTest {
08:     @Test
09:     public void twoThreadsShouldFailEventually() throws Exception {
10:         final ClassWithThreadingProblem classWithThreadingProblem
                = new ClassWithThreadingProblem();
11:
12:         Runnable runnable = new Runnable() {
13:             public void run() {
14:                 classWithThreadingProblem.takeNextId();
15:             }
16:         };
17:
18:         for (int i = 0; i < 50000; ++i) {
19:             int startingId = classWithThreadingProblem.lastId;
20:             int expectedResult = 2 + startingId;
21:
22:             Thread t1 = new Thread(runnable);
23:             Thread t2 = new Thread(runnable);
24:             t1.start();
25:             t2.start();
26:             t1.join();
27:             t2.join();
28:
29:             int endingId = classWithThreadingProblem.lastId;
30:
31:             if (endingId != expectedResult)
32:                 return;
33:         }
34:
35:         fail(“Should have exposed a threading issue but it did not.”);
36:     }
37: }
```

![](figures/apA/0340tab01.jpg)

![](figures/apA/0341tab01.jpg)

This test certainly sets up the conditions for a concurrent update problem. However, the problem occurs so infrequently that the vast majority of times this test won’t detect it.

> 这个测试当然设置了满足并发更新问题发生的条件。不过，问题发生得如此频繁，测试也就极有可能侦测不到。

Indeed, to truly detect the problem we need to set the number of iterations to over one million. Even then, in ten executions with a loop count of 1,000,000, the problem occurred only once. That means we probably ought to set the iteration count to well over one hundred million to get reliable failures. How long are we prepared to wait?

> 实际上，要真正侦测到问题，需要将循环数量设置到 100 万次以上。即便是这样，在 10 个 100 万次循环的执行中，错误也只发生了一次。这意味着我们可能要把循环次数设置为超过亿次才能获得可靠的失败证明。要等多久呢？

Even if we tuned the test to get reliable failures on one machine, we’ll probably have to retune the test with different values to demonstrate the failure on another machine, operating system, or version of the JVM.

> 即便我们调优测试，在单台机器上得到可靠的失败证明，我们可能还需要用不同的值来重新设置测试，得到在其他机器、操作系统或不同版本的 JVM 上的失败证明。

And this is a simple problem. If we cannot demonstrate broken code easily with this problem, how will we ever detect truly complex problems?

> 而且这只是个简单问题。如果连这个简单问题都无法轻易获得出错证明，我们怎么能真正侦测复杂问题呢？

So what approaches can we take to demonstrate this simple failure? And, more importantly, how can we write tests that will demonstrate failures in more complex code? How will we be able to discover if our code has failures when we do not know where to look?

> 我们能用什么手段来证明这个简单错误呢？而且，更重要的是，我们如何能写出证明更复杂代码中的错误的测试呢？我们怎样才能在不知道从何处着手时知道代码是否出错了呢？

Here are a few ideas:

> 下面是一些想法：

- Monte Carlo Testing. Make tests flexible, so they can be tuned. Then run the test over and over—say on a test server—randomly changing the tuning values. If the tests ever fail, the code is broken. Make sure to start writing those tests early so a continuous integration server starts running them soon. By the way, make sure you carefully log the conditions under which the test failed.
- Run the test on every one of the target deployment platforms. Repeatedly. Continuously. The longer the tests run without failure, the more likely that
- The production code is correct or
- The tests aren’t adequate to expose problems.
- Run the tests on a machine with varying loads. If you can simulate loads close to a production environment, do so.

---

> - 蒙特卡洛测试。测试要灵活，便于调整。多次运行测试——在一台测试服务器上——随机改变调整值。如果测试失败，代码就有错。确保及早编写这些测试，好让持续集成服务器尽快开始运行测试。另外，确认小心记录了在何种条件下测试失败。
> - 在每种目标部署平台上运行测试。重复运行。持续运行。测试在不失败的前提下运行得越久，就越能说明：
> - 生产代码正确或；
> - 测试不足以暴露问题。
> - 在另一台有不同负载的机器上运行测试。能模拟生产环境的负载，就模拟之。

Yet, even if you do all of these things, you still don’t stand a very good chance of finding threading problems with your code. The most insidious problems are the ones that have such a small cross section that they only occur once in a billion opportunities. Such problems are the terror of complex systems.

> 即便你做了所有这些，还是不见得有很好的机会发现代码中的线程问题。最阴险的问题拥有很小的截面，在十亿次执行中只会发生一次。这类错误是复杂系统的噩梦。

## A.8 TOOL SUPPORT FOR TESTING THREAD-BASED CODE 测试线程代码的工具支持

IBM has created a tool called ConTest.6 It instruments classes to make it more likely that non-thread-safe code fails.

> IBM 提供了一个名为 ConTest 的工具。它能对类进行装置，令非线程安全代码更有可能失败。

We do not have any direct relationship with IBM or the team that developed ConTest. A colleague of ours pointed us to it. We noticed vast improvement in our ability to find threading issues after a few minutes of using it.

> 我们与 IBM 或开发 ConTest 的团队没有直接关系。有位同事发现了这个工具。在用了几分钟后，我们发现自己发现线程问题的能力得到了很大提升。

Here’s an outline of how to use ConTest:

> 下面是使用 ConTest 的简要步骤：

- Write tests and production code, making sure there are tests specifically designed to simulate multiple users under varying loads, as mentioned above.
- Instrument test and production code with ConTest.
- Run the tests.

---

> - 编写测试和生产代码，确保有专门模拟多用户在多种负载情况下操作的测试，如上文所述；
> - 用 ConTest 装置测试和生产代码；
> - 运行测试。

When we instrumented code with ConTest, our success rate went from roughly one failure in ten million iterations to roughly one failure in thirty iterations. Here are the loop values for several runs of the test after instrumentation: 13, 23, 0, 54, 16, 14, 6, 69, 107, 49, 2. So clearly the instrumented classes failed much earlier and with much greater reliability.

> 用 ConTest 装置代码后，原本千万次循环才能暴露一个错误的比率提升到 30 次循环就能找到错误。以下是装置代码后的几次测试运行结果值：13、23、0、54、16、14、6、69、107、49 和 2。显然装置后的类更加容易和可靠地被证明失败。

## A.9 CONCLUSION 小结

This chapter has been a very brief sojourn through the large and treacherous territory of concurrent programming. We barely scratched the surface. Our emphasis here was on disciplines to help keep concurrent code clean, but there is much more you should learn if you are going to be writing concurrent systems. We recommend you start with Doug Lea’s wonderful book Concurrent Programming in Java: Design Principles and Patterns.7

> 本章只是在并发编程广阔而可怕的领地上的短暂逗留罢了。我们只触及了地表。我们在这里强调的，只是保持并发代码整洁的一些规程，如果要编写并发系统，还有许多东西要学。建议从 Doug Lea 的大作 Concurrent Programming in Java:Design Principles and Patterns 开始。

In this chapter we talked about concurrent update, and the disciplines of clean synchronization and locking that can prevent it. We talked about how threads can enhance the throughput of an I/O-bound system and showed the clean techniques for achieving such improvements. We talked about deadlock and the disciplines for preventing it in a clean way. Finally, we talked about strategies for exposing concurrent problems by instrumenting your code.

> 在本章中，我们谈到并发更新，还有清理及避免同步的规程。我们谈到线程如何提升与 I/O 有关的系统的吞吐量，展示了获得这种提升的整洁技术。我们谈到死锁及干净地避免死锁的规程。最后，我们谈到通过装置代码暴露并发问题的策略。

## A.10 TUTORIAL: FULL CODE EXAMPLES 教程：完整代码范例

### A.10.1 Client/Server Nonthreaded 客户端/服务器非线程代码

Listing A-3 Server.java

> 代码清单 A-3 Server.java

```java
package com.objectmentor.clientserver.nonthreaded;

import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;
import java.net.SocketException;

import common.MessageUtils;

public class Server implements Runnable {
    ServerSocket serverSocket;
    volatile boolean keepProcessing = true;

    public Server(int port, int millisecondsTimeout) throws IOException {
        serverSocket = new ServerSocket(port);
        serverSocket.setSoTimeout(millisecondsTimeout);
    }

    public void run() {
        System.out.printf("Server Starting\n");

        while (keepProcessing) {
            try {
                System.out.printf("accepting client\n");
                Socket socket = serverSocket.accept();
                System.out.printf("got client\n");
                process(socket);
            } catch (Exception e) {
                handle(e);
            }
        }
    }

    private void handle(Exception e) {
        if (!(e instanceof SocketException)) {
            e.printStackTrace();
        }
    }

    public void stopProcessing() {
        keepProcessing = false;
        closeIgnoringException(serverSocket);
    }

    void process(Socket socket) {
        if (socket == null)
            return;

        try {
            System.out.printf("Server: getting message\n");
            String message = MessageUtils.getMessage(socket);
            System.out.printf("Server: got message: %s\n", message);
            Thread.sleep(1000);
            System.out.printf("Server: sending reply: %s\n", message);
            MessageUtils.sendMessage(socket, "Processed: " + message);
            System.out.printf("Server: sent\n");
            closeIgnoringException(socket);
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    private void closeIgnoringException(Socket socket) {
        if (socket != null)
            try {
                socket.close();
            } catch (IOException ignore) {
            }
    }

    private void closeIgnoringException(ServerSocket serverSocket) {
        if (serverSocket != null)
            try {
                serverSocket.close();
            } catch (IOException ignore) {
            }
    }
}
```

Listing A-4 ClientTest.java

> 代码清单 A-4 ClientTest.java

```java
package com.objectmentor.clientserver.nonthreaded;

import java.io.IOException;
import java.net.Socket;

import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import common.MessageUtils;

public class ClientTest {
    private static final int PORT = 8009;
    private static final int TIMEOUT = 2000;

    Server server;
    Thread serverThread;

    @Before
    public void createServer() throws Exception {
        try {
            server = new Server(PORT, TIMEOUT);
            serverThread = new Thread(server);
            serverThread.start();
        } catch (Exception e) {
            e.printStackTrace(System.err);
            throw e;
        }
    }

    @After
    public void shutdownServer() throws InterruptedException {
        if (server != null) {
            server.stopProcessing();
            serverThread.join();
        }
    }

    class TrivialClient implements Runnable {
        int clientNumber;

        TrivialClient(int clientNumber) {
            this.clientNumber = clientNumber;
        }

        public void run() {
            try {
                connectSendReceive(clientNumber);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    @Test(timeout = 10000)
    public void shouldRunInUnder10Seconds() throws Exception {
        Thread[] threads = new Thread[10];

        for (int i = 0; i < threads.length; ++i) {
            threads[i] = new Thread(new TrivialClient(i));
            threads[i].start();
        }

        for (int i = 0; i < threads.length; ++i) {
            threads[i].join();
        }
    }

    private void connectSendReceive(int i) throws IOException {
        System.out.printf("Client %2d: connecting\n", i);
        Socket socket = new Socket("localhost", PORT);
        System.out.printf("Client %2d: sending message\n", i);
        MessageUtils.sendMessage(socket, Integer.toString(i));
        System.out.printf("Client %2d: getting reply\n", i);
        MessageUtils.getMessage(socket);
        System.out.printf("Client %2d: finished\n", i);
        socket.close();
    }
}
```

Listing A-5 MessageUtils.java

> 代码清单 A-5 MessageUtils.java

```java
package common;

import java.io.IOException;
import java.io.InputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.OutputStream;
import java.net.Socket;

public class MessageUtils {
    public static void sendMessage(Socket socket, String message)
            throws IOException {
        OutputStream stream = socket.getOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(stream);
        oos.writeUTF(message);
        oos.flush();
    }

    public static String getMessage(Socket socket) throws IOException {
        InputStream stream = socket.getInputStream();
        ObjectInputStream ois = new ObjectInputStream(stream);
        return ois.readUTF();
    }
}
```

### A.10.2 Client/Server Using Threads 使用线程的客户端/服务器代码

Changing the server to use threads simply requires a change to the process message (new lines are emphasized to stand out):

> 把服务器修改为使用多线程，只需要对处理消息进行修改即可（新的代码行用粗体标出）：

```java
void process(final Socket socket) {
    if (socket == null)
        return;

    Runnable clientHandler = new Runnable() {
        public void run() {

            try {
                System.out.printf("Server: getting message\n");
                String message = MessageUtils.getMessage(socket);
                System.out.printf("Server: got message: %s\n", message);
                Thread.sleep(1000);
                System.out.printf("Server: sending reply: %s\n", message);
                MessageUtils.sendMessage(socket, "Processed: " + message);
                System.out.printf("Server: sent\n");
                closeIgnoringException(socket);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    };

    Thread clientConnection = new Thread(clientHandler);
    clientConnection.start();
}
```
