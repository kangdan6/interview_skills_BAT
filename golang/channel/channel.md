# channel和select

### 介绍一下channel
go语言中，不要通过共享内存来通信，而要通过通信来实现内存共享。Go的CSP(Communicating Sequential Process)并发模型，中文可以叫做通信顺序进程，是通过goroutine和channel来实现的。
所以channel收发遵循先进先出FIFO，分为有缓存和无缓存，channel中大致有buffer(当缓冲区大小部位0时，是个ring buffer)、sendx和recvx收发的位置(ring buffer记录实现)、sendq、recvq当前channel因为缓冲区不足而阻塞的队列、使用双向链表存储、还有一个mutex锁控制并发、其他原属等

**接着回答关于发送数据**：

0. 如果channal是nil,那么将会阻塞当前g,并且永远无法被唤醒。
1. 当channel中recvq存在阻塞的goroutine，那么数据直接发送给当前的g，返回 （易考察）
2. 当channel中存在缓冲，则发送到ring buf中，并修改sendx等
3. 如果都不满足上述情况，(无缓存的channel或者buffer已满)，则将要发送的数据和当前 goroutine 打包成 sudog 对象放入到 sendq 队列中，执行 `gopark()` 将 g 阻塞，当前 Goroutine 置为 waiting 状态，也会陷入阻塞等待其他的协程从 Channel 接收数据，让出 cpu 的使用权；此时因为数据还没发送到channel，为了以防数据被gc，会使用KeepAlive确保数据不会被垃圾回收。
4. 被唤醒，做些清理工作

**接着回答关于接收数据**：
1. 看是不是 nil channel，是的话直接阻塞
2. 看有没有被阻塞的发送者，有多话，
  2.1. 如果channel没有缓冲，那么直接从发送者手里拿，返回
  2.2. 否则从缓冲队首拿数据，并且把阻塞的发送者的数据放队尾，返回
3. 看有没有被阻塞的发送者，有多话，如果channel缓冲为0，那么直接从发送者手里拿，否则拿走队首的数据，并且把阻塞的发送者的数据放队尾，返回
4. 看看缓冲有没有数据，有就读缓冲，返回
5. 阻塞，等待发送者来唤醒自己
6. 被唤醒，做些清理工作

**比较深入的 发送g调度的时机(对goroutine调度不是很清楚，建议不要提，因为后面肯定还会问gmp)**

1. 发送数据时发现 Channel 上存在等待接收数据的 Goroutine，立刻设置处理器的 runnext 属性，但是并不会立刻触发调度；
2. 发送数据时并没有找到接收方并且缓冲区已经满了，这时会将自己加入 Channel 的 sendq 队列并调用 runtime.goparkunlock 触发 Goroutine 的调度让出处理器的使用权；

### 关于关闭 channel 有几点需要注意的是：

- 重复关闭 channel 会导致 panic。

- 向关闭的 channel 发送数据会 panic。

- 从关闭的 channel 读数据不会 panic，读出 channel 中已有的数据之后再读就是 channel 类似的默认值，比如 chan int 类型的 channel 关闭之后读取到的值为 0。

  

### 说说go语言的channel特性？

1. 给一个 nil channel 发送数据，造成永远阻塞
2.  从一个 nil channel 接收数据，造成永远阻塞
3. 给一个已经关闭的 channel 发送数据，引起 panic
4. 从一个已经关闭的 channel 接收数据，如果缓冲区中为空，则返回一个零值
5. 无缓冲的channel是同步的，而有缓冲的channel是非同步的
6. 关闭一个nil channel将会发生panic

###  常见问题

1. Channel分配在堆上还是在栈上？哪些对象分配在堆上？哪些对象分配在栈上
Go编译器通过逃逸分析确定哪些对象分配在堆上，哪些分配在栈上。
逃逸分析：通过指针的动态范围决定一个变量究竟是分配在栈上还是应该分配在堆上。
有两个结论：
● 如果一个函数结束之后外部没有引用，那么优先分配到栈中(如果申请的内存过大，栈区存不下，会分配到堆)。
● 如果一个函数结束之后外部还有引用，那么必定分配到堆中
堆上动态内存分配的开销比栈要大很多，所以有时我们传递值比传递指针更有效率。因为复制是栈上完成的操作，开销要比变量逃逸到堆上再分配内存要少的多


2. chan比mutex更轻么？还有更轻量的方法么？


3. 什么时候用chan不如mutex效率高？
一般来说，chan比mutex效率要高，但是在goroutine之间通讯时就不一定了

mutex是典型的保护资源的思路，channel是通过通信来共享内存，有限考虑channel

### 通常考察select随机执行变成有序执行


## reference
[Golang channel 源码深度剖析](https://www.cyhone.com/articles/analysis-of-golang-channel/)
