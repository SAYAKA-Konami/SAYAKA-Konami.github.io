# Concurrent_Queue


# 同步队列（线程池的参数之一）

## `ArrayBlockingQueue`

数组实现的阻塞队列。在初始化时就要指定其大小。

出队入队共用一把锁：由于该队列的实现是基于“循环数组”。而循环数组的插入和删除都需要借助头尾指针。

例如在插入时需要两个指针协作判断队列是否已满。

## `LinkedBlockingQueue`

基于链表实现的阻塞队列。初始化时可以指定其容量大小。默认时int的最大值。

出入队分别使用一把锁。如果队满，那么会阻塞在由插入锁绑定的 `notFullCondition`上。

如果队空，那么会阻塞在出队锁绑定的`notEmptyCondition`上。

相较于Array。其出队入队的吞吐量较高，毕竟其出队入队并不共享同一把锁。

## `SynchronousQueue`

`SynchronousQueue` 是一个不存储元素的阻塞队列，也即是单个元素的队列。

每一个 put 操作必须等待一个 take 操作，否则不能继续添加元素。`SynchronousQueue`可以看成是一个传球手，负责把生产者线程处理的数据直接传递给消费者线程。队列本身并不存储任何元素，非常适合于传递性场景, 比如在一个线程中使用的数据，传递给另外一个线程使用，`SynchronousQueue` 的吞吐量高于 `LinkedBlockingQueue` 和 `ArrayBlockingQueue`。

## `LinkedTransferQueue`

`LinkedTransferQueue` 是一个由链表结构组成的无界阻塞 `TransferQueue` 队列。

`LinkedTransferQueue`采用一种预占模式。意思就是消费者线程取元素时，如果队列不为空，则直接取走数据，若队列为空，那就生成一个节点（节点元素为null）入队，然后消费者线程被等待在这个节点上，后面生产者线程入队时发现有一个元素为null的节点，生产者线程就不入队了，直接就将元素填充到该节点，并唤醒该节点等待的线程，被唤醒的消费者线程取走元素，从调用的方法返回。我们称这种节点操作为“匹配”方式。

队列实现了 `TransferQueue`接口重写了`tryTransfer`和`transfer`方法，这组方法和 `SynchronousQueue`公平模式的队列类似，具有匹配的功能

## `LinkedBlockingDeque`

`LinkedBlockingDeque` 是一个由链表结构组成的双向阻塞队列。

所谓双向队列指的你可以从队列的两端插入和移出元素。双端队列因为多了一个操作队列的入口，在多线程同时入队时，也就减少了一半的竞争。相比其他的阻塞队列，`LinkedBlockingDeque` 多了 addFirst，addLast，offerFirst，offerLast，peekFirst，peekLast 等方法，以 First 单词结尾的方法，表示插入，获取（peek）或移除双端队列的第一个元素。以 Last 单词结尾的方法，表示插入，获取或移除双端队列的最后一个元素。另外插入方法 add 等同于 addLast，移除方法 remove 等效于 removeFirst。

在初始化 LinkedBlockingDeque 时可以设置容量防止其过渡膨胀，默认容量也是 Integer.MAX_VALUE。另外双向阻塞队列可以运用在“**工作窃取**”模式中。

