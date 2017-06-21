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


5.hash()方法，获取散列码
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


6.HashMap的底层是用Entry实体类来实现的
    static class Entry<K,V> implements Map.Entry<K,V> {		//注意，这是一个静态内部类，所以是用的Map.Entry这样的写法
        final K key;		// 保存“键”
        V value;		// 保存“值”
        Entry<K,V> next;	// 所有哈希地址相同的元素构成一个称为同义词链的单链表，这里是实现同义词单链表的核心
        int hash;

        Entry(int h, K k, V v, Entry<K,V> n) {
            value = v;
            next = n;
            key = k;
            hash = h;
        }

	...

        public final boolean equals(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry e = (Map.Entry)o;
            Object k1 = getKey();
            Object k2 = e.getKey();
            if (k1 == k2 || (k1 != null && k1.equals(k2))) {	// 先比较key是否一致，key代表着存储地址
                Object v1 = getValue();
                Object v2 = e.getValue();
                if (v1 == v2 || (v1 != null && v1.equals(v2)))	// 然后再比较value是否一致，都在同义词链表上，则比较具体值
                    return true;
            }
            return false;
        }

        public final int hashCode() {
            return Objects.hashCode(getKey()) ^ Objects.hashCode(getValue());	// key与value的hashcode然后再异或生成新hashcode
        }

        public final String toString() {
            return getKey() + "=" + getValue();
        }

        ...
    }
  Entry是HashMap中的基本单元，可以看做是“链地址法”中的同义词链。
  当有一个key-value对需要插入的时候，首先计算key的hashcode值，然后通过indexFor()方法映射到不同的存储地址，并组建一个Entry链节点，
  假如未来有key-value对有相同的hashcode，则它们的存储地址将与前面的一样，生成一个Entry链节点后组成一个单链表。
  最后，所有hashcode/存储地址一致的元素构成一个称为同义词链的单链表。
  而在查询的时候，首先计算key的hashcode，然后根据这个hashcode，找到与此对应的单链表，如果单链表中有多个值，然后再进行value的比较，
  返回value一致的元素。


7.获取key映射的值value
    public V get(Object key) {
        if (key == null)
            return getForNullKey();
        Entry<K,V> entry = getEntry(key);

        return null == entry ? null : entry.getValue();
    }
  这是十分平常的get(key)方法，但因为一些特殊情况需要处理，所以需要特别注意下
  首先就是 key == null 的情况，在HashMap中，key == null 是允许的，null key 默认映射到索引为“0”处，所以当key == null的时候还是需要
  仔细的区分，可能这个 key 就是没有值的，也可能这个key == null，同时映射有值value，它们这样放入： put(null, "something!!");
    private V getForNullKey() {
        if (size == 0) {		// key == null ,且 size == 0，确实没有值，返回 null
            return null;
        }
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {	// key == null 的数据默认放入哈希地址为“0”的索引处
            if (e.key == null)					// 然后在此索引对应的同义词链表上挨个进行寻找，寻找e.key == null的
                return e.value;				// 假如确实有key == null ,返回value == “something!!”
        }
        return null;					// 如果没有key == null ，返回 null
    }
  key == null 这个特殊情况处理之后，进入正式的通过key 取 value流程，
    final Entry<K,V> getEntry(Object key) {
        if (size == 0) {
            return null;
        }
        int hash = (key == null) ? 0 : hash(key);	// 注意这里，key == null，hash值返回“0”，其他情况则按照流程获取散列码
        for (Entry<K,V> e = table[indexFor(hash, table.length)];	// 通过此散列码，找出符合条件的元素在Entry数组的哪个位置
             e != null;
             e = e.next) {		// 找到地址，就是找到了同义词链表的入口，然后一路向下比对即可
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))		// 找到 key 一致的元素
                return e;
        }
        return null;
    }	
  最后，返回此key映射的value。	



8.存放键值对key-value
    public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null)
            return putForNullKey(value);	// key == null的情况
        int hash = hash(key);
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }
  假如键值对中有key == null的情况，则使用putForNullKey()方法进行处理，专门用来存入key == null的键值对
    private V putForNullKey(V value) {
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {	// 与前面的一样，key == null的值照例是存放在table[0]处的
            if (e.key == null) {		
                V oldValue = e.value;		// 如果有一个单链表节点key == null，则用后面的value取代前面的value
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
        addEntry(0, null, value, 0);		// 此处的hash是散列码，经过“掩码”处理后得到在存储表中位置，由于key == null
        return null;				// 散列码也默认是“0”,在“桶”中的索引也默认是“0”
    }
  前面还只是铺垫，下面才是真正的将键值对key-value存入同义词单链表节点中
    void addEntry(int hash, K key, V value, int bucketIndex) {
        if ((size >= threshold) && (null != table[bucketIndex])) {	// 此处在扩容
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);		// 散列码通过“掩码”处理后，剩下的就是存储地址
        }
        createEntry(hash, key, value, bucketIndex);		// 创建一个单链表节点，将数据存入
    }
  数据存入单链表节点的具体实现如下所示
    void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];			// 将原来“桶”的位置腾出来，新节点的 next 将指向这个节点
        table[bucketIndex] = new Entry<>(hash, key, value, e);	// 然后将一个新的单链表节点赋值给这个“桶”，有点类似于头部插入
        size++;
    }
