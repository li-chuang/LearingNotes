1.读写锁ReadWriteLock维护了一对相关的锁，一个用于只读操作，一个用于写入操作。写入锁是线程独占的，而读取锁可以由多个线程保持


2.原来的‘锁’ReentrantLock可以认为是‘互斥锁’，一次只允许一个线程操作共享数据，不管是什么操作，都是如此；
  读写锁ReadWriteLock在此基础上做了改进，进行了更细粒度的控制：对于写入操作，一次只能有一个线程可以进行处理；对于读取操作，允许任意数量的线程同时进行读取。

  读写锁适用于读多写少的情况。


3.在‘读写锁’中，‘读锁’与‘写锁’的获取不做区分，代码中没有任何意图或设定能使某种‘锁’先获取，某种‘锁’后获取，‘读写锁’的获取都是一样的。
  当然，这个类中支持‘锁’获取的公平策略：
  a）非公平模式（默认）：连续竞争的非公平锁可能无限期推迟一个或多个reader或writer线程，但吞吐量通常要高于公平锁。
  b）公平模式：线程利用一个近似到达顺序的策略来争夺进入。当释放当前保持的锁时，可以为等待时间最长的单个writer线程分配写入锁，如果有一组等待时间大于所有正在等待的writer线程的reader，
     将为该组分配读者锁


4.写入锁可以降级为读取锁，实现方法是：先获取写入锁，然后获取读取锁，最后释放写入锁。
  当然，从读取锁升级到写入锁是不可能的。


5.读取锁与写入锁都支持锁获取期间的中断


6.下面是一个可重入读写锁的实例，用完完成缓存的更新：
  class CachedData {
    Object data;
    volatile boolean cacheValid;
    final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
 
    void processCachedData() {
      rwl.readLock().lock();
      if (!cacheValid) {
         // Must release read lock before acquiring write lock
         rwl.readLock().unlock();
         rwl.writeLock().lock();
         try {
           // Recheck state because another thread might have
           // acquired write lock and changed state before we did.
           if (!cacheValid) {
             data = ...
             cacheValid = true;
           }
           // Downgrade by acquiring read lock before releasing write lock
           rwl.readLock().lock();
         } finally {
           rwl.writeLock().unlock(); // Unlock write, still hold read
         }
      }
 
      try {
        use(data);
      } finally {
        rwl.readLock().unlock();
      }
    }
  }


7.ReentrantReadWriteLocks常用来提高集合的并发性能，尤其是当这些集合比较庞大而且读取多于写入。
  下面就是一个扩展并且的可并发访问的TreeMap，
  class RWDictionary {
     private final Map<String, Data> m = new TreeMap<String, Data>();
     private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
     private final Lock r = rwl.readLock();
     private final Lock w = rwl.writeLock();
 
     public Data get(String key) {
         r.lock();
         try { return m.get(key); }
         finally { r.unlock(); }
     }
     public String[] allKeys() {
         r.lock();
         try { return m.keySet().toArray(); }
         finally { r.unlock(); }
     }
     public Data put(String key, Data value) {
         w.lock();
         try { return m.put(key, value); }
         finally { w.unlock(); }
     }
     public void clear() {
         w.lock();
         try { m.clear(); }
         finally { w.unlock(); }
     }
  }


8.和ReentrantLock中一样，ReentrantReadWriteLock也是把具体实现委托给内部类处理，这是代理模式；
  下面就是ReentrantReadWriteLock类的整体架构：

  public class ReentrantReadWriteLock implements ReadWriteLock, java.io.Serializable { 		// ReadWriteLock是一个借口，提供了writeLock()与readLock()两个借口方法

	private final ReentrantReadWriteLock.ReadLock readerLock;				// 内部类提供readerLock读取锁对象

	private final ReentrantReadWriteLock.WriteLock writerLock;				// 内部类提供writerLock写入锁对象

	final Sync sync;			// 对象sync，执行所有同步机制，Sync继承自AQS，提供同步机制的基层实现

  	public ReentrantReadWriteLock() {			// 构造函数，默认是非公平获取锁
        	this(false);
    	}

	public ReentrantReadWriteLock(boolean fair) {		// 构造函数，可以设置公平策略	
         	sync = fair ? new FairSync() : new NonfairSync();
        	readerLock = new ReadLock(this);		// ReadLock类是内部类
        	writerLock = new WriteLock(this);		// WriteLock类是内部类
    	}

	public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }	// 获取写入锁
   	public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }	// 获取读取锁

	abstract static class Sync extends AbstractQueuedSynchronizer {		// 抽象静态内部类Sync提供对AQS的继承，是读写锁处理的核心
		...
	}

	static final class NonfairSync extends Sync {			// NonfairSync 为Sync的非公平版本，直接尝试获取锁
		...
	}

	static final class FairSync extends Sync {			// FairSync 为Sync的公平版本，多个争用时，先到先得
		...
	}

	public static class ReadLock implements Lock, java.io.Serializable {		// 继承自Lock接口，锁的获取与释放都在这里
		...
	}

	public static class WriteLock implements Lock, java.io.Serializable {		// 继承自Lock接口，锁的获取与释放都在这里
		...
	}

	protected Thread getOwner() {					// 剩下的就是一堆可用于系统状态监控的方法
        	return sync.getOwner();					// 基本上设计出来就是为了方便系统设备监控的，有些也可以用，到时候再查
   	}

	...
  }


9.其实，看了这个框架后发现，这确实只是一个代理模式，主要的代码都在AQS中，这里无非就是一些外围的代码，或者一些抽象方法实现下。
  具体的实现还是要去AQS 中看。


