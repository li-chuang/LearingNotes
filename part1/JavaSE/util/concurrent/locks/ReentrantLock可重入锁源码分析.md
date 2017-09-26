1.整个Lock锁对象体系与synchronized功能是差不多的，都是用于在多线程情况下保持同步，两者之间互有优劣
  Lock锁对象功能更完善，有许多新加的高级用法，这点是synchronized不如的；
  不过synchronized更接近底层JVM，死锁等线程情况都用java命令获取，而且锁的获取与释放都是在JVM层面自动进行；
  而Lock对象，在JVM层面没有特别之处，都需要通过对象自己完成，如获取锁用lock()方法，释放锁用unlock()方法，且为了保证锁定一定会被释放，必须将 unLock()放到finally{} 中

  另外，一般来说synchronized是够用的，效果相比Lock也不差，所以synchronized还是很有生命力的。


2.在代码中可以看到，ReentrantLock都是把具体实现委托给内部类而是不直接继承自AbstractQueuedSynchronizer，
  这样的好处是用户不会看到不需要的方法，也避免了用户错误地使用AbstractQueuedSynchronizer的公开方法而导致错误。


3.ReentrantLock的重入计数是使用AbstractQueuedSynchronizer的state属性的，state大于0表示锁被占用、等于0表示空闲，小于0则是重入次数太多导致溢出了。


4.可重入锁一个典型的使用如下：
  class X {
      private final ReentrantLock lock = new ReentrantLock();
      // ...
 
      public void m() {
          lock.lock();  // block until condition holds
          try {
              // ... method body
          } finally {
              lock.unlock()
          }
      }
  }


5.ReentrantLock的整体架构
  public class ReentrantLock implements Lock, java.io.Serializable {

  	private final Sync sync;	// ReentrantLock使用了代理模式，sync才是原始对象，

  	abstract static class Sync extends AbstractQueuedSynchronizer {		// AQS是一个很重要的类，Sync抽象静态内部类实现
		...								// 实现AQS抽象类中的很多方法，包括一些核心操作
  	}

  	static final class NonfairSync extends Sync {		// 提供非公平性的锁实现
		...						// 直接尝试获取锁
  	}

  	static final class FairSync extends Sync {		// 提供公平性的锁实现
		...						// 实现公平性的关键在于：如果锁被占用且当前线程不是持有者也不是等待队列的第一个，则进入等待队列。
  	}

  	public ReentrantLock() {			// 这是可重入锁ReentrantLock的默认构造方法
        	sync = new NonfairSync();		// 默认实现的是非公平锁，即每次都是直接获取锁
 	}

  	public ReentrantLock(boolean fair) {		// 这是可重入锁ReentrantLock的构造方法
        	sync = fair ? new FairSync() : new NonfairSync(); 	 
  	}

  	public void lock() {		// 这是ReentrantLock实现的具体方法，其他的方法与此类似
        	sync.lock();		// 注意这里的代理模式，ReentrantLock的lock()方法不是直接代码实现，而是调用sync.lock()来实现此功能，这就是代理模式
  	}

 	...
  }


5.获取锁lock()方法
    public void lock() {
        sync.lock();
    }
  通过前面的构造方法可知，此处提供的是非公平性的锁实现：
  如果锁没被占据，直接获取此锁，并且将锁计数设置为‘1’；
  如果锁本来就是被自己占据，自加一，并且获取此锁；
  如果此锁被其他线程占据，则禁用当前线程，并且在获得锁之前，该线程将一直处于休眠状态，此时锁保持计数被设置为‘1’。
  
  在Sync中lock()为抽象方法，具体实现在NonfairSync与FairSync中
  NonfairSync中具体实现如下所示：
  	final void lock() {
            if (compareAndSetState(0, 1))	 // 0未获取，1已经获取，比较state是否为‘0’，获得后将state设置为‘1’
                setExclusiveOwnerThread(Thread.currentThread());	// 设置独占模式，则一个锁只能被一个线程持有，其他线程必须要等待。
            else
                acquire(1);			// 加锁失败,再次尝试加锁，失败则加入等待队列，禁用当前线程，直到被中断或有线程释放锁时被唤醒
        }
  FairSync中具体实现如下所示：
	final void lock() {
            acquire(1);				// 尝试加锁，失败则加入等待队列，禁用当前线程，直到被中断或有线程释放锁时被唤醒
        }

  可以发现，它们的还是调用的AQS中的代码，
  其中compareAndSetState方法的实现如下所示：
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);	// 这个方法就比较底层了，首先比较state值，然后再设置state值
    }
  该方法以原子操作的方式更新state变量，如果当前状态值等于预期值，则以原子方式将同步状态设置为给定的更新值，
  如果state值为expect,则更新为update值且返回true,否则不更改state且返回false.
  
  其中setExclusiveOwnerThread方法的实现如下所示：
    protected final void setExclusiveOwnerThread(Thread t) {
        exclusiveOwnerThread = t;		// 设置独占模式同步的当前所有者，
    }
  在不公平锁的情况下，将当前线程设置为独占模式。

  其中acquire()方法的实现如下所示：
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&					// 首先尝试获取锁，成功则直接返回
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))	// 否则将当前线程加入锁的等待队列并禁用当前线程
            selfInterrupt();					// 直到线程被中断或者在锁为其它线程释放时唤醒
    }

  总的来说，它们都是为了获取锁，有公平性与非公平性两种方式可以获取得到。


6.获取可中断锁lockInterruptibly()
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
  获取锁，除非当前线程被中断 Thread.interrupt()；
  如果这个锁没被占据，获取此锁并立即返回，并且将计数器设置为‘1’；
  如果当前线程已经保持此锁，则将保持计数加 1，并且该方法立即返回；
  如果此锁被其他线程占据，则按照调度计划禁用当前线程，并且在发生以下两种情况之一以前，该线程将一直处于休眠状态： 
      1）锁由当前线程获得；2）其他某个线程中断当前线程；
  如果当前线程获得该锁，则将锁保持计数设置为 1；如果当前线程： 
      1）在进入此方法时已经设置了该线程的中断状态；或者 
      2）在等待获取锁的同时被中断。 
      则抛出 InterruptedException，并且清除当前线程的已中断状态。 
  在此实现中，因为此方法是一个显式中断点，所以要优先考虑响应中断，而不是响应锁的普通获取或重入获取

  sync对象调用的 acquireInterruptibly()来自AQS中，Sync继承过来，此方法的具体实现如下：
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())		// 首先判断是否中断，中断则终止
            throw new InterruptedException();
        if (!tryAcquire(arg))			// 首先尝试获取锁，成功则直接返回
            doAcquireInterruptibly(arg);	// 尝试获取锁，检测到中断则直接退出循环，抛出InterruptedException异常
    }
  acquireInterruptibly()以独占模式获取锁，如果被中断则中止；
  acquire()也是以独占模式获取锁，忽略中断。

  说明一：以上两个方法lock()与lockInterruptibly()都是获取锁，它们的区别在于，
	a) lock优先考虑获取锁，待获取锁成功后，才响应中断
	b) lockInterruptibly()优先考虑响应中断，而不是响应锁的普通获取或重入获取

  说明二：doAcquireInterruptibly大体上相当于前面的acquireQueued,关键的区别在于检测到interrupted后的处理，acquireQueued简单的记录下中断曾经发生，
        然后就象没事人似的去尝试获取锁，失败则休眠。而doAcquireInterruptibly检测到中断则直接退出循环，抛出InterruptedException异常


7.尝试获取锁tryLock()
    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }
  只有在调用时锁未被其他线程锁定的情况下，才获取该锁.
  如果这个锁没被占据，获取此锁并立即返回，并且将计数器设置为‘1’；
      即使已将此锁设置为使用公平排序策略，但是调用 tryLock() 仍将 立即获取锁（如果有可用的），而不管其他线程当前是否正在等待该锁.
      在某些情况下，此“闯入”行为可能很有用，即使它会打破公平性也如此。如果希望遵守此锁的公平设置，则使用 tryLock(0, TimeUnit.SECONDS) ，它几乎是等效的（也检测中断）
  如果当前线程已经保持此锁，则将保持计数加 1，该方法将返回 true。
  如果锁被另一个线程保持，则此方法将立即返回 false 值

  Sync中的nonfairTryAcquire()方法，在Sync类中具体实现，且不可以改写，其具体实现如下所示：
	final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();				// 获取锁的状态state
            if (c == 0) {				// 锁是空闲的，进行加锁必须用CAS来确保即使有多个线程竞争锁也是安全的  
                if (compareAndSetState(0, acquires)) {	// 比较并设置锁的状态值state
                    setExclusiveOwnerThread(current);	// 把当前线程设为锁的持有者
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {	// 锁被占用且当前线程是锁的持有者，说明是重入。
                int nextc = c + acquires;
                if (nextc < 0) // overflow			// 溢出。加锁次数从0开始，加锁与释放操作是对称的，所以绝不会是小于0值，小于0只能是溢出
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);				// 锁被持有的情况下，只有持有者才能更新锁保护的资源
                return true;
            }
            return false;
        }  

  另外还有带参数的tryLock()
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout)); 	// 返回值代表是否在超时时间之内成功获得锁，此方法具体实现在AQS中
    }
  作用与不带参数的tryLock()类似，区别就是加了一个时间限制，拿不到锁，就等一段时间，超时就返回false
  
  以上几种方法，都是用来获取锁，不过获取方式有些差别：
  1）lock()，拿不到锁就不罢休，不然线程就一直block，比较无赖的做法；
  2）tryLock()，马上返回，拿到锁就返回true，不然返回false，比较潇洒的做法；
  3）tryLock(long,TimeUnit)，拿到锁就返回true，拿不到锁，就等一段时间，超时才返回false，比较聪明的做法；
  4）lockInterruptibly()，拿到锁就返回true，拿不到锁阻塞时，如果有中断，立即处理中断，抛出中断异常



8.释放锁unlock()
    public void unlock() {
        sync.release(1);
    }
  尝试释放锁
  如果当前线程是锁的拥有者，将锁计数器减去‘1’；如果锁计数器现在是‘0’，则表示这个锁已经被释放

  sync对象调用的方法release()是在AQS中实现，具体实现方式如下：
    public final boolean release(int arg) {
        if (tryRelease(arg)) {				// 尝试释放状态
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);			// 唤醒当前节点的后继节点所包含的线程，也即将休眠中的线程唤醒，让其继续acquire状态
            return true;
        }
        return false;
    }  


9.返回绑定到此 Lock 实例的新 Condition 实例
    public Condition newCondition() {
        return sync.newCondition();
    }
  在这里，可以获得Condition对象，用这个方法，可以在Condition对象对象与ReentrantLock对象之间取得联系。
  它的具体处理，在写Condition类源码分析的时候再详细阐述。


10.获取当前线程锁计数器
    public int getHoldCount() {
        return sync.getHoldCount();
    }
  如果获取的锁计数器数据为‘0’，说明当前线程已经没有保留任何锁了。
  获取锁计数器数据一般用于测试或者调试中，例如，如果确定不能进入某段已经持有‘锁’的代码，那么可以这样断言：
      class X {
        ReentrantLock lock = new ReentrantLock();
        // ...
        public void m() {
          assert lock.getHoldCount() == 0;	// 就是这句断言，下面的代码此时只有当前线程可以进入
          lock.lock();
          try {
            // ... method body
          } finally {
            lock.unlock();
          }
        }
      }


11.判断‘锁’是否为当前线程所持有
    public boolean isHeldByCurrentThread() {
        return sync.isHeldExclusively();
    }
  和上面的方法一样，此方法也多运用在测试与调试中，举例，获取‘锁’的那个线程才可以进行访问
      class X {
        ReentrantLock lock = new ReentrantLock();
        // ...
     
        public void m() {
            assert lock.isHeldByCurrentThread();	// 注意，这里是断言，此方法只能是获取‘锁’的那个线程可以访问，
            // ... method body
        }
      }
  同时，它还可以确保在不可重用方法中使用可重用锁，举例，
      class X {
        ReentrantLock lock = new ReentrantLock();
        // ...
     
        public void m() {
            assert !lock.isHeldByCurrentThread();	// 这里断言，获取‘锁’的线程不能进入这个方法，或者说，此方法不能是获取‘锁’的那个线程可以访问，
            lock.lock();
            try {
                // ... method body
            } finally {
                lock.unlock();
            }
        }
      }


12.判断‘锁’是否被任何线程所持有
    public boolean isLocked() {
        return sync.isLocked();
    }
  这个方法被设计出来是用于监控系统的 state ，而不是用来进行同步控制


13.判断‘锁’分配是否公平
    public final boolean isFair() {
        return sync instanceof FairSync;
    }
  公平返回true，不公平返回false


14.返回拥有这个‘锁’的线程
    protected Thread getOwner() {
        return sync.getOwner();
    }
  返回此‘锁’的拥有者，如果没有线程拥有这个‘锁’，则返回null。
  这个方法常用于方便锁监控，一般不会是拥有此锁的线程调用此方法，而是另外的线程，通过此方法来获取其他活动的线程，从而进行线程监控；
  监控了此‘锁’，也就监控了获得此‘锁’进行工作的线程。


15.判断是否有其他线程正在等待获取这个‘锁’
    public final boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }
  这个方法设计出来主要是用于监控系统中的state。
  如果有其他的线程正在等待获取‘锁’，则返回true，否则返回false


16.判断指定的线程是否正在等待获取这个‘锁’
    public final boolean hasQueuedThread(Thread thread) {
        return sync.isQueued(thread);
    }
  这个方法设计出来主要是用于监控系统中的state。
  如果指定的线程正在排队等待获取这个‘锁’，则返回true，否则返回false


17.返回正在等待获取此‘锁’的线程的数量
    public final int getQueueLength() {
        return sync.getQueueLength();
    }
  这个方法设计出来是用于监控系统中的state，不是用来进行同步控制的。
  返回值为等待获取此‘锁’的线程的数量


18.返回正在等待获取此‘锁’线程的集合
    protected Collection<Thread> getQueuedThreads() {
        return sync.getQueuedThreads();
    }
  由于线程的各种状态随着时间的推移都是在变化的，获取到的线程的集合只是估计值，
  这个方法也是用来进行系统监控的。


19.判断是否有其他的线程用给定的Condition等待获取此‘锁’
    public boolean hasWaiters(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.hasWaiters((AbstractQueuedSynchronizer.ConditionObject)condition);
    }
  这个方法一般也是用来进行系统监控的。


20.返回用给定的Condition等待获取此‘锁’的线程数量
    public int getWaitQueueLength(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitQueueLength((AbstractQueuedSynchronizer.ConditionObject)condition);
    }
  首先，这也是一个预估值，另外，这个方法主要用于系统监控，不是用来进行同步控制。


21.返回用给定的Condition等待获取此‘锁’的线程的集合
    protected Collection<Thread> getWaitingThreads(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitingThreads((AbstractQueuedSynchronizer.ConditionObject)condition);
    }
  和前面的一样，主要用于系统监控，而且线程状态都在不断变化，获取的仅仅能代表获取那一瞬间的情况


22.获取‘锁’本身的情况
    public String toString() {
        Thread o = sync.getOwner();			// 返回此‘锁’的持有线程
        return super.toString() + ((o == null) ?
                                   "[Unlocked]" :	// 没有线程持有此‘锁’，此锁此时未锁定
                                   "[Locked by thread " + o.getName() + "]");	// 写出持有此‘锁’的线程名
    }
