1.TreeMap底层的数据存储是通过红黑树（Red-Black Tree）实现的，红黑树确保数据的存储是有序的，并且确保时间复杂度为log(n)。
  红黑树是一种十分优秀的数据存储结构，在Java8中，即使HashMap中都改用红黑树进行存储，比用同义词链表效率更高。
  在Java7中，HashMap用的是同义词链表，使用hash值确定存储的同义词链表所在位置，各种操作效率都很高，但却不保证顺序，而且扩容后
    重新装载，原有的顺序还有可能会改变；
    而TreeMap用的是红黑树进行存储，这样在对其进行操作之前，需要通过循环找到被操作Entry的位置，这点比较耗时，但好处是所有的数据
    都是排序好的。


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
    private void fixAfterInsertion(Entry<K,V> x) {
        x.color = RED;				// 新加入红黑树的默认节点就是红色
        while (x != null && x != root && x.parent.color == RED) {	// 注意，假如父节点是黑色，添加一个红色节点后结构还是正常的，或是根节点，不用进行调节

            if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {		// 如果X的父节点（P）是其父节点的父节点（G）的左节点，获取其叔(U)节点(图1)
                Entry<K,V> y = rightOf(parentOf(parentOf(x)));
                if (colorOf(y) == RED) {			// 如果叔节点是红色的（父节点有判断是红色）. 即是双红色，比较好办，通过改变颜色就行(图2). 
                    setColor(parentOf(x), BLACK);		// 把P和U都设置成黑色然后,X加到P节点。 G节点当作新加入节点继续迭代
                    setColor(y, BLACK);
                    setColor(parentOf(parentOf(x)), RED);
                    x = parentOf(parentOf(x));
                } else {					// 处理红父，黑叔的情况
                    if (x == rightOf(parentOf(x))) {		// 如果X是右边节点(图3)
                        x = parentOf(x);
                        rotateLeft(x);				// 进行左旋(图4)
                    }
                    setColor(parentOf(x), BLACK);		// 到这，X只能是左节点了，而且P是红色，U是黑色的情况
                    setColor(parentOf(parentOf(x)), RED);
                    rotateRight(parentOf(parentOf(x)));		// 把P改成黑色，G改成红色，以G为节点进行右旋(图5)
                }
            } else {							// 父节点在右边的情况(图6)
                Entry<K,V> y = leftOf(parentOf(parentOf(x)));	// 获取U
                if (colorOf(y) == RED) {			// 红父红叔的情况,直接将颜色从红色改为黑色，然后将新节点设置为红色插入(图7)
                    setColor(parentOf(x), BLACK);
                    setColor(y, BLACK);
                    setColor(parentOf(parentOf(x)), RED);	// 把G当作新插入的节点继续进行迭代
                    x = parentOf(parentOf(x));
                } else {					// 红父黑叔，并且是右父的情况
                    if (x == leftOf(parentOf(x))) {		// 如果插入的X是左节点(图8)
                        x = parentOf(x);
                        rotateRight(x);				// 以P为节点进行右旋(图9)
                    }
                    setColor(parentOf(x), BLACK);
                    setColor(parentOf(parentOf(x)), RED);
                    rotateLeft(parentOf(parentOf(x)));		// 以G为节点进行左旋(图10)
                }
            }
        }
        root.color = BLACK;			// 红黑树的根节点始终是黑色
    }    
  其实，有了注解后，还是比较难理解，可以参照下面的图：
  图1.			图2.			图3.			图4.			图5.
	  G			G			G			G			P
	/   \		      /   \		      /   \		      /	  \		      /   \
     P(red)  U		   P(red)  U(red)	   P(red)  U(black)	   P(red)  U(black)        X(red)  G(red)
			 /			     \			   /				     \
		        X      			      X			 X(red)				      U

  图6.			图7.			图8.			图9.			图10.
	  G			G			G			G			P
        /   \		      /   \		      /   \		      /	  \		      /   \
       U   P(red)         U(red)  P(red)  	 U(black) P(red)	 U(black)  P(red)	   G(red)  X(red)
				 /			  /			     \		   /
				X			 X			     X(red)       U	
  其实，总结起来，只有3中情况，包括一种变色和两种旋转，父节点在左边还是在右边没有太大的影响，相对应调整一下即可
  情况一：父节点是黑色，随便怎么插都可以，路径中黑色数量没有变化，颜色也不冲突

  情况二：父节点和叔节点都是红色，插入的节点初始也是红色，此时改变父节点与叔节点的颜色
		G						G(red)
	      /   \						/    \
  	  P(red)   U(red)				       P      U
	/     \	   /	\		--->>		     /   \  /   \	
       X(red)						  X(red)
       /   \						   /  \
  注意：红黑树中有一条重要的总结特征，即所有叶子节点到根节点经过的黑色节点数量是一致的。由于我们已经将P和U改成了黑色，导致G这支路径中黑色节点数量
        增加，所以我们需要将G节点改成红色进行平衡（假如G是根节点则除外），这样叶子节点到G所通过的黑色节点数量都是“1”，插入前后没有变化

  情况三：父节点是红色，叔节点是黑色，父节点是左节点时子节点是右节点/父节点是右节点时子节点是左节点，此时以父节点为中心进行左旋/右旋
		G						G
	      /   \					      /   \
          P(red)   U					   X(red)  U
         /    \	  / \			--->>		   /   \  / \
 	    X(red) 					P(red) 
	     /	\					/   \
  注意：父节点与叔节点颜色不一致，而红黑树所有叶子节点到根节点途经的黑色节点数量一致，说明这是一个中间状态，父节点是红色叔节点是黑色不是一种可保持
        的长期状态

  情况四：父节点是红色，叔节点是黑色，父节点是左节点时子节点是左节点/父节点是右节点时子节点是右节点
		G						P
              /   \					      /   \
           P(red)  U					   X(red)  G(red)
	   /   \  / \			--->>		   /   \    /   \
        X(red)    							 U
	/    \ 								/ \
  注意：和前面一种情况一样，这是一种中间状态，原理和前面差不多
