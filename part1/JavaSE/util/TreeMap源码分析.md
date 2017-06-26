1.TreeMap底层的数据存储是通过红黑树（Red-Black Tree）实现的，红黑树确保数据的存储是有序的，
  并且确保时间复杂度为log(n)。
  红黑树是一种十分优秀的数据存储结构，在Java8中，即使HashMap中都改用红黑树进行存储，比用同义词
  链表效率更高。


2.TreeMap中最重要的是红黑树的实现，下面就是红黑树实现的基础——节点
    static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left = null;		// 节点的左右子节点默认为 null
        Entry<K,V> right = null;	
        Entry<K,V> parent;
        boolean color = BLACK;		// 节点颜色默认为黑色

        Entry(K key, V value, Entry<K,V> parent) {	// 注意，Entry静态内部类中有6个属性，此处只实现了三个，剩下的取默认值
            this.key = key;
            this.value = value;
            this.parent = parent;
        }

        public K getKey() {		// 返回节点中的 key 值
            return key;
        }

        public V getValue() {		// 返回节点中的 value 值
            return value;
        }

        public V setValue(V value) {	// 重新设置 value 值，并返回原有值
            V oldValue = this.value;
            this.value = value;
            return oldValue;
        }

        public boolean equals(Object o) {	// 这里equals的比较，最终比的是key和value的值是否一致，而不是整体对象进行比较
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;

            return valEquals(key,e.getKey()) && valEquals(value,e.getValue());	// 这里，key & value一致就返回true
        }

        public int hashCode() {
            int keyHash = (key==null ? 0 : key.hashCode());
            int valueHash = (value==null ? 0 : value.hashCode());
            return keyHash ^ valueHash;				// hashCode 由key和value共同决定
        }

        public String toString() {
            return key + "=" + value;
        }
    }
  剩下的一点补充的代码，进行两个值的比对
    static final boolean valEquals(Object o1, Object o2) {
        return (o1==null ? o2==null : o1.equals(o2));
    }
  可以看到，这里多加了对两个值都是 null 的比较，而不仅仅只有o1.equals(o2)；
  换句话说，两个key都为null，或者两个value都为null，都是可以的，它们都是一致的。


3.下面的都是对红黑树的各种处理，现在，就从红黑树最基本的插入开始，看看红黑树是如何工作的。
    public V put(K key, V value) {
        Entry<K,V> t = root;	// 获取根节点
        if (t == null) {
            compare(key, key); 	// 这里是对key == null的情况进行判断与处理，检查key是否为null

            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);	// 注意，红黑树纯天然就是“二分查找”的，先与父节点进行比较
                if (cmp < 0)			// 值比父节点的值小，转入左节点继续比较
                    t = t.left;
                else if (cmp > 0)		// 值比父节点的值大，转入右节点继续比较
                    t = t.right;
                else
                    return t.setValue(value);	// 相等，则无需进行插入等操作，直接进行value值的替换即可，并返回被替换的值
            } while (t != null);		// 只要不是到底部，一直进行比较
        }
        else {	
            if (key == null)			
                throw new NullPointerException();
            Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                parent = t;
                cmp = k.compareTo(t.key);	// 此处用的是自带的比较器，但比较的过程还是一样的，不过自带比较器对null的情况
                if (cmp < 0)			// 处理与前面的不一样，此处key == null 会报错
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        Entry<K,V> e = new Entry<>(key, value, parent);	// 注意，这个Entry的构造方法只传入了三个值，左右子节点默认为null，颜色默认为黑色
        if (cmp < 0)		// 如果到了尾部叶子节点处，插入的数据比此节点小，将新节点接入此尾部节点的左子树
            parent.left = e;
        else
            parent.right = e;	// 插入的数据比此节点大，将新节点接入此尾部节点的右子树
        fixAfterInsertion(e);	// 插入后调整，这个很重要
        size++;
        modCount++;
        return null;
    }
  数据加入到红黑树中，其实整个过程很好理解，先进行“二分查找”，如果能找到相同的key，则只是将value进行替换即可，并返回被替换的值
  如果没有找到相同的值，则找准插入的位置，插入后进行节点调整，最终完成整次 put 操作。
  下面就是插入后调整的具体实现：
    
