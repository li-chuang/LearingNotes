1.HashMap允许key和value都是null
  HashMap大致等于Hashtable，区别在于HashMap方法都不是同步synchronized的，另外HashMap允许null
    注意，Java的设计者是不支持全部同步的，因为这样很影响性能而且 不够灵活。最后的办法是根据实际需要
    在代码块中加synchronized，或者使用线程安全的ConcurrentHashMap，于是慢慢的Hashtable就淘汰了。
  HashMap不保证映射的顺序性，而且，也不保证随着时间的推移原有的顺序依然保持一致，不发生改变。


2.HashMap中存放的键值对的数量 size
  默认的初始容量是capacity = 16（1<<4），
  默认的加载因子是loadFactor = 0.75，装载因子用来衡量HashMap满的程度。计算HashMap的实时装载因子的方法为：size/capacity
  默认的最大容量是（1<<30），主要是用来避免初始化时设置过大的容量，当初始化时设置的容量大于（1<<30）时
    容量设置为（1<<30）
  扩容的阈值threshold = capacity * load factor，HashMap的实际容量大于此阈值时会执行resize操作


3.构造方法
    public HashMap(Map<? extends K, ? extends V> m) {	// m也是一个Map容器对象，假设 m.size() = 25
	// 等价于 HashMap(Math.max((int) (m.size() / 0.75) + 1, 16), 0.75); 等价于HashMap(34 , 0.75);
        this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1, DEFAULT_INITIAL_CAPACITY), DEFAULT_LOAD_FACTOR);
	// 扩张映射表
        inflateTable(threshold);
	// 将m放入新的容器中
        putAllForCreate(m);
    }
  由上面可知，对于一个实际容量25的Map对象，至少要设置34的初始容量，不然就会太拥挤，超过加载因子
    public HashMap(int initialCapacity, float loadFactor) {
        ...
        this.loadFactor = loadFactor;
        threshold = initialCapacity; 	// 将初始容量的值赋值给threshold阈值
        init();
    }
  而事实上，Map容器的最初threshold等同于初始容量


4.扩张映射表，假设此处toSize = 25
    private void inflateTable(int toSize) {
        int capacity = roundUpToPowerOf2(toSize);			// 此处结果capacity = 32，是最小大于toSize的2^n数
        threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);	// 此处threshold = 24
        table = new Entry[capacity];
        initHashSeedAsNeeded(capacity);
    }
  注意，映射表的底层是用Entry记录实现的，所以要实现一个长度为capacity的Entry数组


5.hash()方法，检索对象或者向结果hash中追加hashCode，避免遇到冲突
    final int hash(Object k) {
        int h = hashSeed;
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();		// 一次散列，调用k的hashCode方法，与hashSeed做异或操作

        h ^= (h >>> 20) ^ (h >>> 12);	// 二次散列
        return h ^ (h >>> 7) ^ (h >>> 4);
    }
  说明：
    1.首先，假如你确信代码不需要修改，或者觉得别人修改后效果更差，可以使用final修饰符，这样此代码就不可以被继承重写
    2.'^'为异或运算，对同一个字符进行两次异或运算就会回到原来的值。
	假设key.hashCode()的值为：0x7FFFFFFF，table.length为默认值16，算法执行如下
