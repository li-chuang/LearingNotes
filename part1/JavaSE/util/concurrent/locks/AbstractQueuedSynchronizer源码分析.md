1.AbstractQueuedSynchronizer提供了一个基于FIFO队列，可以用于构建锁或者其他相关同步装置的基础构架。
  该同步器利用一个int来表示状态，这是大部分同步实现的基础，通过继承可以实现，通过类似acquire和release可以来操作此状态。

  AQS是一个半成品的抽象类，它封装好了线程如何等待锁与如何释放锁的规则。
  比如多个线程竞争锁，没有获得锁的线程将放到FIFO中排队，并且以自旋的方式去尝试获得锁。但是一个线程什么情况下获得锁和释放锁，需要自己去定义。
  简单的说，AQS帮你实现了框架上的规则，但是框架下面的更具体的规则需要自己来实现。
  
  举个不恰当的例子，一个吃饭的游戏： 
  一群人吃饭，但只有一个饭桶，只有获得饭桶的人才能从饭桶里迟到饭，每次只有一个人能获得饭桶，没有抢到饭桶的人就去排队。
  AQS实现了这样一个规则：
  如果没有人获得饭桶，那么大家一起抢饭桶，谁先抢到谁就获得这个饭桶。其他没有获得饭桶的人就去排队，后面参加进来的人就排在队伍的末尾。
  但是大家并不是老老实实待在队伍里不动，也不是等着别人把饭桶让给你，而是队伍里的每个人都在关注着饭桶是否被前面得到的人释放了，并一遍一遍的去尝试着抢饭桶，
  知道抢到饭桶（获得锁）或者自己累了（异常退出）退出抢饭桶的行列，或者他妈喊他回家吃饭（中断或取消）。
  
  这个抢饭桶规则就是AQS给你实现的规则。
  但是什么情况下被认定为抢到饭桶，这个规则需要我们来补充，比如抢饭桶释放忠厚传给队里排第一个的人，或者传给队列里排队最久的那个人，或者传给队列里年龄最小的人等等。

  这也是就AQS获得锁的过程。如果你得到了饭桶，吃饱了饭就要释放饭桶，让给其他人，这就是释放锁的过程。
  


2.同步器的实现依赖于一个FIFO队列，那么队列中的元素Node就是保存着线程引用和线程状态的容器，每个线程对同步器的访问，都可以看做是队列中的一个节点。
  没有抢到锁的线程将会放到这个队列里面，这个队列是双向队列，所以节点node的属性中包含前驱节点后后继节点，以及排队的线程，同时还有一些判断获得释放锁需要的元素

    static final class Node {
        static final Node SHARED = new Node();		// 标记表示一个节点正在以共享模式等待
      
        static final Node EXCLUSIVE = null;		// 标记表示一个节点正在以排他模式等待

        static final int CANCELLED =  1;		// waitStatus值为1，表示线程已经被取消
        
        static final int SIGNAL    = -1;		// waitStatus值为-1，表示当前节点的后继节点包含的线程需要运行，也就是unpark(park停车，unpark启动)
        
        static final int CONDITION = -2;		// waitStatus值为-2，表示当前节点在等待condition，也就是在condition队列中；
        
        static final int PROPAGATE = -3;		// waitStatus值为-3，表示当前场景下后续的acquireShared能够得以执行；

        volatile int waitStatus;		// 表示节点的状态，上面的几个都是它的状态值，注意，为了保持原子同步性，使用了volatile 关键字
						// 除此之外，还有一个值为0，表示当前节点在sync队列中，等待着获取锁。

        volatile Node prev;			// 前驱节点，比如当前节点被取消，那就需要前驱节点和后继节点来完成连接。		

        volatile Node next;			// 后继节点。

        volatile Thread thread;			// 入队列时的当前线程。使用后清零

        Node nextWaiter;			// 存储condition队列中的后继节点。1.SHARED:表示该节点线程处于共享模式等待。2.EXCLUSIVE：表示该节点线程处于排他模式等待。

        final boolean isShared() {		// 返回节点是否是以共享模式等待
            return nextWaiter == SHARED;
        }

        final Node predecessor() throws NullPointerException {		 // 返回前一个节点
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }

  首先普及几个概念
  自旋锁(Spin Lock)：指当一个线程尝试获取某个锁时，如果该锁已被其他线程占用，就一直循环检测锁是否被释放，而不是进入线程挂起或睡眠状态
	缺点，1，通讯开销很大，一直在用 while() 或 for(;;) 循环进行CAS检测 ；2，不能保证公平性，谁先抢到谁使用，不保证等待进程/线程按照FIFO顺序获得锁。
  标签锁(Ticket Lock)：这个是为了解决自旋锁的公平性问题，类似于现实银行柜台的排队叫号，按排队号获得服务。
        这个锁也需要轮询，不过与自旋锁进行CAS轮询不同，线程只需要轮询此锁的当前服务号是否是自己的排队号，是，则表示自己拥有了锁，可以去接受服务了，不是，则继续轮询。
        当线程释放时，将服务号加1，这样下一个线程看到这个变化，就会推出自旋。
        缺点，在多处理器系统上，排队号缓存的同步是个大问题
  CLH锁：是一种基于链表的自旋锁，申请线程只在本地变量上自旋，它不断轮询前驱的状态，如果发现前驱释放了锁就结束自旋。
  MCS锁：是一种基于链表的自旋锁，申请线程只在本地变量上自旋，直接前驱负责通知其结束自旋，从而极大地减少了不必要的处理器缓存同步的次数，降低了总线和内存的开销
  CLH锁与MCS锁的差异在于：
	a.从代码实现来看，CLH比MCS要简单得多。
	b.从自旋的条件来看，CLH是在前驱节点的属性上自旋，而MCS是在本地属性变量上自旋
	c.从链表队列来看，CLH的队列是隐式的，CLHNode并不实际持有下一个节点；MCS的队列是物理存在的。
	d.CLH锁释放时只需要改变自己的属性，MCS锁释放则需要改变后继节点的属性。

  CLH队列锁原有优点：
	a.进出队快，线程进入同步器仅仅需要在队尾增加一个节点，FIFO严格先进先出，所以总是第一个线程获取锁，所以此框架不支持基于优先级的同步策略
	b.检查是否有线程在等待很容易，只需检查头尾指针是否相同即可
  现用的CLH 和原有的CLH的改进在于：
	a.当timeout或cancel时，可以直接唤起后继节点，使后继使用前驱的状态字段获取锁
	b.在每个节点中使用一个状态字段去控制阻塞，而不是自旋。只有队头线程才允许调用tryAcquire()
	c.head节点使用的是傀儡节点

  节点中有一个状态位，这个状态位与线程状态密切相关，这个状态位(waitStatus)是一个32位的整型常量，它的取值如下：
        static final int CANCELLED =  1;	// 因为超时或者中断，节点会被设置为取消状态，被取消状态的节点不应该去竞争锁，只能保持取消状态不变，不能转换为其他状态。
   						// 处于这种状态的节点会被踢出队列，被 GC 回收。     
        static final int SIGNAL    = -1;	// 表示这个节点的继任节点被阻塞，到时需要通知它
        static final int CONDITION = -2;	// 表示这个节点在条件队列中，因为等待某个条件而被阻塞
        static final int PROPAGATE = -3;	// 在共享模式头节点有可能处于这种状态，表示锁的下一次获取可以无条件传播
				     0 		// 默认状态 0 ，新节点会处于这种状态	

  一个锁可以有多个条件，条件节点内部也有一个状态字段，条件节点是通过nextWaiter 指针串起来的一个独立的队列。
  条件队列中的线程在获取锁之前，必须先被转移到同步队列中去，转移是先断开条件队列的第一个节点，然后插入到同步队列中，这个新插入到同步队列中的节点和同步队列中原有的节点
  一起排队等待获取锁。

  虽然队列内部被设计为FIFO,但并不意味着这个同步器一定是公平的。前面说到，在tryAcquire() 检查之后再排队，因此，新线程完全可以偷偷排在第一个线程前面。
  如果需要绝对公平，那很简单，只需要在tryAcquire()方法，不在队头返回false即可。检查是否在队头可以使用getFirstQueuedThread()方法。
  有一种情况是，队列是空的，同时有多个线程一拥而入，谁先抢到锁就谁运行，这其实与公平并不冲突，是对公平的补充。

  AQS 的内部类Node中有两个常量SHARE 和EXCLUSIVE，这两个常量用于表示这个节点支持共享模式还是独占模式，共享模式指的是允许多个线程获取同一个锁而且可能获取成功，独占模式指的是
  一个锁如果被一个线程持有，其他线程必须等待。
  多个线程读取一个文件可以采用共享模式，而当有一个线程在写文件时不会允许另一个线程写这个文件，这就是独占模式的应用场景。


3.同步器AQS的三个成员变量

    private transient volatile Node head;	// 等待队列的头节点，懒初始化，只能被 setHead 方法修改
						// 注意：头只要是存在的，那么它的Node状态就不会是 CANCELLED 的

    private transient volatile Node tail;	// 等待队列的尾节点，也是懒初始化，只能通过 enq 方法增加新的等待节点


    private volatile int state;			// 同步状态，这个属性有对应的 getState 和 setState 方法

  注意，以上的三个成员变量，都是用volatile 来修饰的，这样可以在底层确保操作的同步


4.真正实现修改 state 状态的方法
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);	// 这个方法为内联函数设置支持
    }

  说明 : 如果当前 state 等于期望值，那么原子地设置同步器 state 为给定更新值；
	 说白了，就是 state 状态，先比较，再更新；比较结果不一致则不更新。


5.在队尾添加节点
    private Node enq(final Node node) {			// enq()方法采用的是变种CLH算法
        for (;;) {
            Node t = tail;
            if (t == null) { 				// 先看头节点是否为空，这一步只会在队列初始化时会执行
                if (compareAndSetHead(new Node()))	// 如果为空就创建一个傀儡及诶单
                    tail = head;			// 头指针与尾指针都指向这个傀儡节点
            } else {					// 如果头节点不为空
                node.prev = t;				
                if (compareAndSetTail(t, node)) {	// 采用CAS操作将当前节点设置为尾节点
                    t.next = node;			// 
                    return t;				// 入队成功，返回原来的尾节点
                }
            }
        }
    }


6.使用当前线程构造一个节点
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);		// 使用当前线程和模式构造一个节点

        Node pred = tail;
        if (pred != null) {					// 判断队列中是否有元素
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {		// 如果有元素，就设置当前节点为队尾节点
                pred.next = node;
                return node;					// 如果符合要求，在这里就可以return返回，这里没有成功，则走到下面的enq()方法中
            }
        }
        enq(node);				// 如果没有元素，表示队列为空，做入队操作
        return node;
    }
  addWaiter(Node mode)方法也是通过compareAndSetTail方法来实现队列同步的。addWaiter在性能上做了优化。在大部分情况下队列都不会为空，所以先按照队列不为空的情况进行处理把当前node挂到队列末尾。
  如果当前队列为空或者执行compareAndSetTail同步失败，那么在执行完整的把当前节点挂到队列末尾的的方法enq(node)。



7.设置某节点为头节点
    private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
    }


8.唤醒节点的后继节点
    private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;			// node参数传进来的是头节点，获取此节点的节点状态
        if (ws < 0)					// 如果状态值为负(状态码-1，-2，-3)，表示头节点还需要通知后续节点，后面会通知后续节点，因此将该标志位清零
            compareAndSetWaitStatus(node, ws, 0);	// 将此节点的状态码都改为‘0’，表示当前节点在sync队列中，等待着获取锁

        Node s = node.next;				// 然后查看头结点的下一个结点
        if (s == null || s.waitStatus > 0) {		// 此后继节点节点不存在或者是取消(状态码‘1’)，则寻找下一个可唤醒的节点，然后唤醒它并返回
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)	// 从尾节点向前找符合条件的节点
                if (t.waitStatus <= 0)
                    s = t;
        }						// 如果下一个结点不为空且它的状态码 <=0 ，表示后继节点没有被取消，是一个可以唤醒的节点，于是唤醒后继节点返回。
        if (s != null)
            LockSupport.unpark(s.thread);		// 找到了之后，启动此节点
    }
  如果头结点状态小于0，则先清除头结点状态，置为0，那么唤醒头结点的下一个节点。如果该节点不存在，或者改节点状态是取消状态，那么从队列末尾开始找，找到最后一个状态没有取消的节点，
  然后调用LockSupport.unpark(s.thread)方法释放许可，唤醒该节点的线程。
  为什么只对头结点的后继节点释放许可呢？因为每次只有一个线程获得锁，只有当前节点的前驱节点是头节点才能获得锁，所以只需要唤醒头结点的线程就可以了。


9，以共享模式释放状态
    private void doReleaseShared() {        
        for (;;) {				// 注意，这是一个无限循环，不达成目标不会 break 跳出
            Node h = head;			// 找到头节点
            if (h != null && h != tail) {
                int ws = h.waitStatus;		// 获取节点状态
                if (ws == Node.SIGNAL) {	// -1 表示此节点需要运行
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))		// 0 表示等待获取锁，在这里改变状态
                        continue;            // loop to recheck cases		// 假如改失败了，则继续检查，再次尝试
                    unparkSuccessor(h);						// 直到改成功，启动后继节点
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))	// 或者状态是‘0’的，改为‘-3’，后续的共享模式可以获取执行，将状态向后一个节点传播。
                    continue;                // loop on failed CAS		// 修改不成功的继续检查
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
  doReleaseShared方法实现的功能如下：遍历队列中的所有节点，如果节点状态为signal，把改siganl状态置为0，并调用unparkSuccessor(h);方法把该节点的后继节点线程唤醒；
  如果该节点状态为0，则把状态设置为PROPAGATE。


10.设置头节点并传播
  private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        
        if (propagate > 0 || h == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
  先把当前节点设为为头节点，然后转向下一个节点，如果同样是“shared”类型的，调用doReleaseShared方法，再做一个"releaseShared"操作。


11.取消锁请求
    private void cancelAcquire(Node node) {
        if (node == null)		// 忽略节点不存在的情况
            return;

        node.thread = null;		// node不再关联到任何线程，取消的节点不会再去竞争锁，然后踢出队列，最终被 GC 

        Node pred = node.prev;		// 跳过被cancel的前驱node，找到一个有效的前驱节点pred。由于设置后继比设置前驱更有效，很多处理的时候只设置了后继
					// 然后在其他的请求中设置前驱，在这里也是如此
        while (pred.waitStatus > 0)	// CANCELLED == 1，这一句是用作情况整理用的，一些被取消掉的节点的引用在此处进行整理
            node.prev = pred = pred.prev;

        Node predNext = pred.next;

        node.waitStatus = Node.CANCELLED;	// 将node的waitStatus置为CANCELLED

        if (node == tail && compareAndSetTail(node, pred)) {	// 如果node是tail，更新tail为pred，并使pred.next指向null
            compareAndSetNext(pred, predNext, null);
        } else {
            int ws;
            if (pred != head &&								// 如果node既不是tail，又不是head的后继节点
                ((ws = pred.waitStatus) == Node.SIGNAL ||				// 则将node的前继节点的waitStatus置为SIGNAL
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&	// 并使node的前继节点指向node的后继节点（相当于将node从队列中删掉了）
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                unparkSuccessor(node);				// 如果node是head的后继节点，则直接唤醒node的后继节点
            }

            node.next = node; // help GC
        }
    }
  取消锁请求cancelAcquire()的主要操作有两类：
  清理状态
	a.node不再关联到任何线程
	b.node的waitStatus置为CANCELLED
  node出队（包括三个场景的出队）
	a.node是tail
	b.node既不是tail，也不是head的后继节点
	c.node是head的后继节点


12.判断当前线程是否应该阻塞
  一个节点尝试获取锁失败，通过此方法检查和更新节点的状态status
  如果当前线程应该阻塞，则返回true。使用时需要确保pred == node.prev，即pred 节点是 node节点的前驱节点。
  将新加入的节点放入队列之后，这个节点有两种状态，要么获取锁，要么就挂起。如果这个节点不是头节点，就看看这个节点是否应该挂起，如果应该挂起，就挂起当前节点。
  是否应该挂起是通过 shouldParkAfterFailedAcquire() 方法来判断的。
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;			// 获取前驱节点状态
        if (ws == Node.SIGNAL)				// SIGNAL == -1，表示这个节点的前驱节点会通知它，那么它可以放心大胆的阻塞了。
            return true;				// 如果前驱节点是SIGNAL状态，则意味这当前线程需要被unpark唤醒。此时，返回true。
        if (ws > 0) {					// 如果前驱节点是“取消”状态，一个被取消的节点，则跳过这个“前驱节点”继续向前找，直到找到一个有效的节点作为“前驱节点”为止
            do {
                node.prev = pred = pred.prev;		// 移除 node 前面等待状态为 CANCELLED 的节点
            } while (pred.waitStatus > 0);		// 向前遍历跳过被取消的节点，直到找到一个没有被取消的节点为止
            pred.next = node;				// 将找到的这个节点作为它的前驱节点
        } else {
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);	//  如果前继节点为“0”或者“共享锁”状态，则设置前继节点为SIGNAL状态。
        }
        return false;					// 返回false表示线程不应该被挂起
    }
  本方法通过以下规则，判断“当前线程”是否需要被阻塞
  a.如果前驱节点状态为SINGAL,表明当前节点需要被其他线程unpack(唤醒)，此时则返回true。
  b.如果前驱节点状态为CANCELED(ws > 0) ,说明前驱节点已经被取消，则通过回溯找到一个有效（非CANCELED状态）的节点，并返回false
  c.如果前驱节点状态既不是SINGAL，也不是CANCELED，则设置前驱节点的状态为SINGAL，并返回false

  为什么“前继节点是SIGNAL”状态，则“当前线程”需要被阻塞呢。这是因为在FIFO框架中，只有最前面一个节点获取锁，后一个节点可以对锁进行争用，其他的节点都应该是阻塞的。
  在这里也是如此，前驱节点阻塞，当前节点也应该阻塞。


13.产生一个中断
    private static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
  一个方便的方法用来中断当前线程


14.阻塞当前线程并检查是否已经被阻塞
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);			// 通过LockSupport的park()阻塞“当前线程”。
        return Thread.interrupted();		// 返回线程的中断状态。
    }


15.通过自旋方式去获得锁，并返回是否中断过，使用独占不可中断模式
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();		// 获取前驱节点
                if (p == head && tryAcquire(arg)) {		// 首先判断当前节点的前驱节点是不是头节点，如果是，则用自行自定义的tryAcquire(arg)获取锁		
                    setHead(node);				// 设置当前节点为头节点
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&	// 到了此处说明前驱节点不是头节点。此处执行中断检查，检查并更新没有获得锁的node的状态并判断该节点的线程是否应该阻塞
                    parkAndCheckInterrupt())			// 阻塞当前线程并检查是否已经被阻塞
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);				// 取消锁请求
        }
    }
  “当前线程”会根据公平性原则进行阻塞等待，直到获取锁为止；并且返回当前线程在等待过程中有没有并中断过。
  在for(;;)中，为了保证FIFO 的公平性，一般只有头节点可以获取锁去运行线程，后一个节点等待获取线程，其他的节点阻塞。
  如果当前节点的前驱节点是队列的头节点，在头节点释放锁之后，就轮到当前节点获取锁了。所以，在for(;;)中会判断当前节点前驱是否为头节点，如果是，则不断自旋，
  通过tryAcquire()获取锁；如果获取成功，说明前驱的头节点中线程已经释放了锁，当前节点可以设置为头节点。
     

16.获得锁，以独占可中断模式获取
    private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);	// 使用当前线程构造一个节点，并且设置节点类型为独占模式
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();	
                if (p == head && tryAcquire(arg)) {	// 和前面的一样，判断前驱节点是否为头节点，并且不断尝试获取锁
                    setHead(node);			// 获得锁之后，将当前节点设置为头节点，只有头节点才能够获取锁运行，线程运行起来的那一定是头节点
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();	// 和前面返回是否中断过不一样，这里直接抛出异常
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
  外界可以对当前线程进行中断，提前结束获取状态的操作，线程也提前返回。
  中断的时候，acquireQueued对中断没响应，正常执行代码，只是在最后返回一个Boolean变量，显示是否中断过。
  doAcquireInterruptibly则是一有中断操作，直接抛出异常


17.获得锁，以独占模式获取
   其中参数 nanosTimeout 表示最长超时等待时间
    private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
        long lastTime = System.nanoTime();
        final Node node = addWaiter(Node.EXCLUSIVE);	// 使用当前线程构造一个节点，并且设置节点类型为独占模式
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();	// 获取前驱节点，并判断前驱节点是否为头节点
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                if (nanosTimeout <= 0)			
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&	// 检查并更新没有获得锁的node的状态并判断该节点的线程是否应该阻塞
                    nanosTimeout > spinForTimeoutThreshold)	// spinForTimeoutThreshold 默认为1000，用spinForTimeoutThreshold防止时间小于1ms的park
                    LockSupport.parkNanos(this, nanosTimeout);	// ????在规定时间内获得锁都行，而不是像前面两个方法那样，有没有获得锁立即得出结果。如果超时时间已过，还没有获得锁，返回false。
                long now = System.nanoTime();
                nanosTimeout -= now - lastTime;
                lastTime = now;
                if (Thread.interrupted())
                    throw new InterruptedException();		// 这个也是可中断的
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
  以上三种方法获取锁，返回值的区别：
  acquireQueued返回值代表轮训获取锁的过程中有没有中断。
  doAcquireInterruptibly无返回值，如果有中断，抛出异常
  doAcquireNanos返回值代表是否在超时时间之内成功获得锁
 
  以上三种方法获取锁，使用上的区别：
  acquireQueued方法会一直阻塞，直到获得锁，如果被打断，返回false表示被打断，不会抛出中断异常。
  doAcquireInterruptibly方法一直阻塞，直到获取锁，或者被打断抛出中断异常，不需要返回值。
  doAcquireNanos方法会阻塞到获取锁，返回true表示获得了锁，或者被打断抛出异常，或者到超时，返回false表示没有获得锁。


18.获取锁，以共享不可中断模式获取
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);	// 使用当前线程构造一个节点，并且设置节点类型为共享模式，加入到队尾
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();	// 获取前驱节点
                if (p == head) {			// 判断新节点的前驱节点是否为头节点，如果是
                    int r = tryAcquireShared(arg);	// tryAcquireShared()方法的具体实现是在子类中完成的，作用是让前驱在共享模式下获取锁
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);	// 如果获取成功，把当前节点设置为头节点，又因为是共享模式，获取锁的操作可向后传播
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();		// 中断当前线程
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&	// 如果不是头节点，获取失败后判断是否应该阻塞
                    parkAndCheckInterrupt())			// 阻塞当前线程并检查是否已经被中断
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);			// 当前线程取消获取锁
        }
    }
  与之对应的，还有一个‘以独占不可中断模式获取’
  ‘共享模式’可以用于读写锁中的‘读锁’，可以多个线程获取使用
  ‘独占模式’可以用于读写锁中的‘写锁’，表明在同一时间点，只能有唯一的一个线程可以获取锁

  线程在争用‘锁’的时候，它们会形成一个个Node节点放在FIFO列表中，除了第一个节点（head节点是一个空节点不算）已经获取到‘锁’正在运行线程外，第二个节点不断尝试获取锁，
  其他的节点则一直在阻塞。
  于是有的时候，需要调整线程的运行，原来正在阻塞的节点因为某些原因不再需要它们的运行了，这个时候，就可以发出信号让线程中断。
  对中断信号处理的差异这个时候就显现出来了：
    有的中断信号一来，立即进行处理，抛出异常，进入 finally 块中，此节点获取锁操作被取消；
    有些则仅仅记录一下，照常运行，仅仅返回一个信号值，表明曾经被中断过。


19.获取锁，以共享可中断模式获取
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);		// 生成一个节点，设置为共享模式，并且加入到FIFO队尾
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();		// 查看自己的前驱节点，并检查是否为头节点
                if (p == head) {
                    int r = tryAcquireShared(arg);		// 获取锁的实现在子类中完成
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);		// 进入到这里，说明获取锁成功，可以将此节点设置为头节点了，而且由于是共享模式，获取锁的操作可以向后传播
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&	// 进入到这里，说明前面获取锁的操作没有成功，判断是否应该阻塞一下，有没有中断信号什么的，以后再通知唤醒
                    parkAndCheckInterrupt())			// 到了这一步，说明此线程应该阻塞，并且执行中断
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);				// 可中断模式的核心在这里，这个finally 块一般只有报错的时候可进，也就是中断的时候，进入这里，取消获取锁操作
        }
    }


20.获取锁，以共享限时模式
    private boolean doAcquireSharedNanos(int arg, long nanosTimeout) throws InterruptedException {
        long lastTime = System.nanoTime();
        final Node node = addWaiter(Node.SHARED);		// 生成一个节点，设置为共享模式，并且加入到FIFO队尾
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();		// 这里都一致，检查是否为头节点
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {				// 判断是否获取了‘锁’
                        setHeadAndPropagate(node, r);		// 设置头节点并且将操作向后传播
                        p.next = null; // help GC		
                        failed = false;
                        return true;
                    }
                }
                if (nanosTimeout <= 0)				// 检查最大等待时间的有效性
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&	// 检查是否应该将节点阻塞
                    nanosTimeout > spinForTimeoutThreshold)	// 检查时间限制是否大于 1ms(1000ns)，防止时间小于1ms的休眠阻塞
                    LockSupport.parkNanos(this, nanosTimeout);	// 最多等待指定的时间后，阻塞当前线程
                long now = System.nanoTime();
                nanosTimeout -= now - lastTime;			// 这里是计算出一个剩余时间
                lastTime = now;
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
  有些节点它可能一直在尝试获得‘锁’，但‘锁’却一直不释放，这个时候此节点就在不断进行自旋。
  于是有些时候需要这种方法，设置一个最大等待时间，在规定的时间内，如果没有获得‘锁’，还可以继续尝试，而不是像其他的方法那样阻塞，只有在超过了最大等待时间而且没有获得‘锁’，
  才会真正阻塞。

  以上三个方法，分别为：
  doAcquireShared(): 带参数请求共享锁。 （忽略中断）
  doAcquireSharedInterruptibly(): 带参数请求共享锁，且响应中断。（每次循环时，会检查当前线程的中断状态，以实现对线程中断的响应）
  doAcquireSharedNanos(): 带参数请求共享锁但是限制等待时间。（第二个参数设置超时时间，超出时间后，方法返回。）


21.获取锁对外接口，独占，忽略中断
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&			// tryAcquire()方法获取锁成功则返回true，if判断就是false，不会再往下走；只有获取锁失败才会继续向下运行
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))	// 如果获取锁失败，那么就创建一个代表当前线程的节点加入到等待队列的尾部
            selfInterrupt();					// 如果以上两种方式都没有能够获取到锁，则自我中断当前线程
    }
  在AQS类方法中方法名不含shared的默认都是独占模式，在独占模式下，子类需要重写tryAcquire()方法。
  线程首先通过tryAcquire()方法在独占模式下获取锁，如果获取成功就直接返回，否则通过acquireQueued()获取锁，如果仍然失败则selfInterrupt当前线程


22.获取锁对外接口，独占，可中断  
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();	// 如果中断，抛出异常
        if (!tryAcquire(arg))			// 首先通过tryAcquire()尝试获取锁，tryAcquire()方法的具体实现是在子类中完成的
            doAcquireInterruptibly(arg);	// 如果上一步获取锁失败，则使用doAcquireInterruptibly()方法继续获取锁
    }


23.获取锁对外接口，独占，限制等待时间，可中断
    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();	// 如果中断，抛出异常
        return tryAcquire(arg) ||		// 首先通过tryAcquire()尝试获取锁
            doAcquireNanos(arg, nanosTimeout);	// 如果上一步获取锁失败，则使用doAcquireNanos()方法继续获取锁，规定的时间内还没有获取到锁，则认为超时，返回false
    }
  
24.以上是独占模式的三个对外接口，共享模式也有三个对外接口
    共享，忽略中断 
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
    共享，可中断 
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
    共享，限制等待时间，可中断
    public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquireShared(arg) >= 0 ||
            doAcquireSharedNanos(arg, nanosTimeout);
    }
  以上的6个public方法，都是对外接口，它们是前面6个private具体实现方法的对外封装。
  6个方法分为两组，前面3个方法是独占的，后面3个方法是共享的。
  方法取名的时候默认是独占的，而共享的方法名中有 'Shared'，其他的部分几乎都是一样的。


25.独占模式下的释放
    public final boolean release(int arg) {
        if (tryRelease(arg)) {			// 释放锁的规则需要自己实现
            Node h = head;
            if (h != null && h.waitStatus != 0)		// 头节点存在且状态不为‘0’
                unparkSuccessor(h);			// 那么需要对头节点的后继节点进行唤醒操作，调用方法unparkSuccessor()
            return true;
        }
        return false;
    }
  锁的释放分为两步，第一步用 tryRelease() 释放锁，并且修改相应节点的状态，由于FIFO中获得到锁的都是头节点，修改的自然也是头节点的状态
  第二步就是唤醒后继节点，原生的CLH是后继节点自旋，头节点一释放锁，后面的节点就去争夺锁，改良后的CLH可以在锁释放后通知后续节点，然后获得锁。


26.共享模式下的释放
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {		// 释放锁的规则同样需要自己实现
            doReleaseShared();
            return true;
        }
        return false;
    }
  共享锁的释放同样分为两步，第一步是调用同步器的 tryReleaseShared() 方法来释放状态，
  第二步就是唤醒后继节点，在 doReleaseShared() 方法中唤醒其后继节点。


27.在AQS中还有一个非常重要的内部类ConditionObject条件队列，条件队列的元素是一个个正在等待状态的线程。

  条件队列提供了一种挂起方式，当某个线程等待的条件非真时，挂起自己并释放锁，一旦等待条件为真，则立即醒来。这也是条件队列提供的主要功能。

  ConditionObject实现的是 Condition接口，Condition的目的是替代Object的 wait,notify,notifyAll 方法，它是基于Lock实现的，
  而Lock设计出来就是来替代 synchronized 的。

  对象的内置锁（synchronized语义对应的同步机制）关联着一个内置的条件队列，Object的wait/notify/notifyAll等方法构成了内部条件队列的API（操作内置条件队列）。
  内置锁的局限表现在每个内置锁只能关联一个条件队列，当线程需要等待多个条件时，则需要同时获取多个内置锁。

  与内置锁对应的是显式锁，显式锁关联的条件队列是显式条件队列。
  显式锁可以与多个条件队列关联，Condition是显式锁的条件队列，它是Object的wait/notify/notifyAll等方法的扩展。提供了在一个对象上设置多个等待集合的功能，即一个对象上设置多个等待条件。

  Condition也称为条件队列，与内置锁关联的条件队列类似，它是一种广义的内置条件队列。它提供给线程一种方式使得该线程在调用wait方法后执行挂起操作，直到线程等待的某个条件为真时被唤醒。 
  条件队列必须跟锁一起使用，因为对共享状态变量的访问发生在多线程环境下，原理与内部条件队列一样。一个Condition的实例必须跟一个Lock绑定。因此， Condition一般作为Lock的内部类现。

  以下为 Condition接口的说明：
  public interface Condition {

      void await() throws InterruptedException;					// 暂停此线程

      void awaitUninterruptibly();						// 跟上面类似,不过不响应中断

      long awaitNanos(long nanosTimeout) throws InterruptedException;		// 带超时时间的await()

      boolean await(long time, TimeUnit unit) throws InterruptedException;	// 带超时时间的await()
    
      boolean awaitUntil(Date deadline) throws InterruptedException;		// 带deadline的await()

      void signal();								// 唤醒某个等待在此condition的线程
  
      void signalAll();								// 唤醒所有等待在此condition的所有线程
  }

  在原有的synchronized模式下，只能达到一个条件的等待与唤醒，在Object下，当前线程可以释放Object上的监视器并且挂起，直到有另外的线程调用Object.notify()方法
  或者Object.notifyAll()方法唤醒当前线程，成功获取监视器后继续往下执行。
  而在Lock模式下，由ConditionObject代替了Condition中监视器方法操作的位置，而且与前面synchronized模式只能完成一个条件的等待与唤醒不一样，Condition可以实现多个，
  可以达到多个条件的等待与唤醒

  这是一个有关有限缓冲区的例子，有put()与take()两种操作，如果尝试去take一个空的缓冲区，则这个线程被阻塞直到缓冲区不再为空，同理，如果尝试给一个满的缓冲区put值，
  这个线程也会被阻塞直到有空间被释放出来，这样我们就可以对put 与take 分开进行单独处理，相互之间不干涉。这种情形与synchronized中的wait/notify不一样，当有多个线程wait
  的时候，notify是任选一个线程唤醒，这比Condition定向唤醒差多了，而notifyAll又是全部唤醒，唤醒范围太广，没有Condition定向唤醒精细。举例如下：
  class BoundedBuffer {
      final Lock lock = new ReentrantLock();			// 获得一个可重入锁
      final Condition notFull  = lock.newCondition();		 
      final Condition notEmpty = lock.newCondition(); 

      final Object[] items = new Object[100];
      int putptr, takeptr, count;
 
      public void put(Object x) throws InterruptedException {
        lock.lock();						// 1.获得锁才进入操作；	7.线程再次获得锁进入操作	
        try {
          while (count == items.length)
            notFull.await();					// 2.当缓冲区满时，notFull条件等待，此线程暂停，释放锁；	8.此时缓冲区不满，notFull条件不会等待
          items[putptr] = x;
          if (++putptr == items.length) putptr = 0;
          ++count;
          notEmpty.signal();					// 9.线程通过运行，加入了数据到缓冲空间，将notEmpty条件唤醒，就是notEmpty条件没有沉睡也没有关系
        } finally {
          lock.unlock();					// 10.操作完成，释放锁
        }
      }
 
      public Object take() throws InterruptedException {
        lock.lock();						// 3.另外一个线程获得锁进入操作
        try {
          while (count == 0)					// 4.此时缓冲区已满，此条件不符，notEmpty条件自然不会等待，线程继续运行
            notEmpty.await();
          Object x = items[takeptr];
          if (++takeptr == items.length) takeptr = 0;
          --count;
          notFull.signal();					// 5.线程通过运行，释放了缓冲区空间，于是将notFull条件唤醒
          return x;
        } finally {
          lock.unlock();					// 6.线程运行完成，释放锁
        }
      }
    }


28.AQS维护的队列是当前等待资源的队列，AQS会在资源被释放后，依次唤醒队列中从前到后的所有节点，使它们对应的线程恢复执行，直到队列为空。
  而Condition自己也维护了一个队列，该队列的作用是维护一个等待signal信号的队列，两个队列的作用是不同的，事实上，每个线程也仅仅会同时存在这两个队列中的一个，
  流程是这样的：
  a.线程1调用reentrantLock.lock()时，线程被加入到AQS的等待队列中
  b.线程1中当await()方法被调用时，该线程从AQS中移除，对应操作是锁的释放
  c.接着马上被加入到Condition的等待队列中，并等待着signal信号来唤醒
  d.由于线程1释放了锁，于是线程2获取锁，并加入到AQS的等待队列中
  e.线程2调用signal()方法，这个时候Condition的等待队列中只有线程1一个节点，于是它被取出来，加入到AQS的等待队列中，注意，这个时候，线程1还没有被唤醒
  f.signal()方法执行完毕，线程2工作完成，调用reentrantLock.unLock()方法释放锁。这个时候AQS中只有线程1，于是，AQS释放锁后按从头到尾的顺序唤醒线程时，线程1被唤醒，
    于是线程1恢复执行。
  g.直到释放锁，整个过程执行完毕。

  可以看到，整个协作过程是靠节点在AQS的等待队列和Condition的等待队列中来回移动实现的，Condition作为一个条件类，自己很好的维护了一个等待信号的队列，并在适合的时候将
  节点加入到AQS的等待队列中来实现线程的唤醒操作。


29.理明白了AQS与Condition的关系，以及明白了Condition里面维护的就是一个队列，剩下具体实现的一些方法就清楚了。
  首先从一个线程加入到Condition队列中开始，
        private Node addConditionWaiter() {
            Node t = lastWaiter;				// lastWaiter是队列中最后一个节点
            if (t != null && t.waitStatus != Node.CONDITION) {	// 如果尾节点被取消，清理掉
                unlinkCancelledWaiters();			// 遍历整个列表，清除已经取消的节点
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);	// 创建一个节点，设置此节点线程正在等待条件唤醒，CONDITION == -2
            if (t == null)
                firstWaiter = node;		// 当Condition队列为空的时候，此节点自然是firstWaiter
            else
                t.nextWaiter = node;		// 否则，就将节点挂在等待队列的末尾
            lastWaiter = node;
            return node;
        }


30.其次是调用await()等待，加入到Condition队列中
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();				// 此方法加入Condition队列可以中断
            Node node = addConditionWaiter();					// 将线程包装为Node节点并加入到Condition队列中，并返回这个加入队列中的节点
            int savedState = fullyRelease(node);				// 释放当前的锁，注意，这里是线程因为业务需求要进行等待，需要确保完全释放锁，并唤醒其他线程执行
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {					// 判断此节点是否在同步队列中
                LockSupport.park(this);						// 将同步队列中的这个线程挂起
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)	// 检查挂起中是否可以中断
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)	// 线程唤醒后继续争抢‘锁’
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
  剩下的几个await方法都类似，大同小异，都是把当前线程创建一个Node加入Condition队列，接着就一直循环检查其在不在AQS同步队列，如果当前节点在AQS同步队列里，就可以竞争锁，恢复运行。
  这个等待操作，完成了一下的工作：
  a.通过addConditionWaiter()方法将当前线程作为一个Node加入到Condition队列中
  b.通过fullyRelease()方法将当前线程的锁释放，反应到AQS队列中，则是头节点的状态变化，头节点结束运行，马上被移除，从而完成从AQS队列移除的操作
  c.然后就是一个while循环一直进行检查，判断node节点是否在AQS同步队列中，这里完成的就是‘等待signal唤醒’的操作了。因为signal操作时，与await相反，会将Node节点从Condition队列取出，
    加入到AQS同步队列中。所以，只要node在AQS队列中，就说明已经进行过signal唤醒操作，Node节点转移了队列。之后就退出了while循环
  d.线程唤醒后尝试获取‘锁’，之后线程恢复执行，一直到释放锁，执行完毕



31.最后是调用signal()等待，加入到AQS同步队列中
        public final void signal() {
            if (!isHeldExclusively())				// 如果不是排他模式，抛出异常
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;				// 将Condition等待队列的第一个节点出队，并将其加入到AQS同步队列中
            if (first != null)
                doSignal(first);				// 具体实现在doSignal()方法中完成
        }
  综上所述，条件队列上的等待和唤醒操作，本质上是节点在AQS线程等待队列和Condition条件队列之间相互转移的过程，当需要等待某个条件时，线程会将当前节点添加到条件队列中，
  并释放持有的锁；当某个线程执行条件队列的唤醒操作，则会将条件队列的节点转移到AQS等待队列。
  每个Condition都是一个条件队列，可以通过Lock的newCondition创建多个等待条件。
  
