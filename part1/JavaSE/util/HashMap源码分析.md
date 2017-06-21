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

        h ^= k.hashCode();		// 调用k的hashCode方法，与hashSeed做异或操作

        h ^= (h >>> 20) ^ (h >>> 12);	
        return h ^ (h >>> 7) ^ (h >>> 4);	// 这样移位，所有位在最低位上都进行了运算，于是其他位的变化都会反映到最低位上
    }
    static int indexFor(int h, int length) {
        return h & (length-1);		// indexFor的代码很简单，就是把散列值和数组长度做一个"与"操作，
    }
  说明：
    a.首先，假如你确信代码不需要修改，或者觉得别人修改后效果更差，可以使用final修饰符，这样此代码就不可以被继承重写
    b.'^'为异或运算，对同一个字符进行两次异或运算就会回到原来的值。
    c.以上这段代码称之为“扰动函数”，做了4次移位或混合，其实也有其他的方法可以实现同一目标，Java8中就对此做了优化，不过原理
      还是一样的。
      理论上散列值是一个int型，如果直接拿散列值作为下标访问HashMap主数组的话，考虑到int的范围从-2147483648到2147483648，
      大概有40亿的映射空间。只要哈希函数映射得比较均匀松散，一般应用是很难出现碰撞的。
      但问题是一个40亿长度的数组，内存是放不下的（HashMap的默认容量才16），所以这个散列值是不能直接拿来用的，用之前还要先做
      对数组的长度取模运算，得到的余数才能用来访问数组下标。于是就有了以下函数
 	static int indexFor(int h, int length) {
        	return h & (length-1);		// indexFor的代码很简单，就是把散列值和数组长度做一个"与"操作，
    	}
      这也正好解释了为什么HashMap的数组长度要取2的整次幂（初始长度16）。因为这样（数组长度 - 1）正好相当于一个“低位掩码”。
      “与”操作的结果就是散列值的高位全部归零，只保留低位值，用来做数组下标访问。
      但这个时候问题来了，这样就算我的散列值再松散，要是只取最后几位的话，碰撞也会很严重。
      这个时候，“扰动函数”的价值就提现出来了，经过多次移动与异或操作，使最后几位包含了所有位的信息。举例如下，
	创建一个HashMap，其中entry数组为默认大小16，现在有一个key,value对需要存储到HashMap中，该key的hashcode是0x0ABC000(32位)，
	如果不经过hash函数处理这个hashcode，这个键值对过会儿将被存放到entry数组中小标为0处。下标 = 0x0ABC0000 & (16-1) = 0；
	然后我们又要存储另外一个键值对，其key的hashcode是0x0DEF0000，得到数组下标依然是0 。
	这是一个特别差劲的hash算法，明明key相差很大，但有效的只有最后几位，导致它们都存放在了同一个链表里，从而引发冲突。
	这个时候，就需要“扰动函数”出面，通过若干次的移位，异或操作，将原本集中在前端的二进制“1”均匀分布，变得松散，例如，
	经过hash函数处理后，0x0ABC0000变为0x0A02188B，0x0DEF0000变为0x0D2AFC70,它们的下标不再是清一色的“0”了。
    d.至于这个“扰动函数”为什么要这样写，可以参见：
	http://www.cnblogs.com/killbug/p/4560000.html
