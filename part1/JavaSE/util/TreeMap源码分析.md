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

  情况四：父节点是红色，叔节点是黑色，父节点是左节点时子节点是左节点/父节点是右节点时子节点是右节点，此时以父节点为中心进行右旋/左旋
		G						P
              /   \					      /   \
           P(red)  U					   X(red)  G(red)
	   /   \  / \			--->>		   /   \    /   \
        X(red)    							 U
	/    \ 								/ \
  注意：和前面一种情况一样，这是一种中间状态，而且情况四是情况三的后一种状态

  前面我们知道红黑树与2-3-4树是对应关系，我们现在用2-3-4树的视角看看这些变化
  情况一：父节点是黑色节点，就像2-3-4树中的二叉节点一样（没有红线），还有两个机会可以插入且不改变层次

  情况二：父节点和叔节点都是红色，这种情况下，红色节点上面的那根线可以放平，连同他们的父节点，共同组成一个四叉结构，形如
		P-G-U
	此时，一个新的节点N加入，会导致此四叉结构分解，G提到上一级，P、U变成二叉结构（没有了水平红线），都有两次机会可以插入且不改变层次

  情况三：父节点是红色，叔节点是黑色，父节点是左节点时子节点是右节点/父节点是右节点时子节点是左节点，在2-3-4树维度下，它们是这样的
		P-G    			G-P				P-X-G			G-X-P			
	       / | \		       / | \		--->>	       /  |  \		       /  |  \
    		    U		      U	 				      U		      U																	
 	现在，要添加一个红色的X在P与G之间，在这种情况下，P、X都有红线且都在G的一边，这时，就可以把它们旋转成一场条了   

  情况四：父节点是红色，叔节点是黑色，父节点是左节点时子节点是左节点/父节点是右节点时子节点是右节点，在2-3-4树维度下，它们是这样的
		X-P-G    		G-P-X				P			P	
	       /  |  \		       /  |  \		--->>	       / \		       / \		
    		      U		      U	 			      X   G		      G   X
									   \		     /
									    U		    U
         此时的四叉结构并不稳定，需要进行处理，于是P上提，从而造成了一次旋转




4.红黑树的删除操作
    public V remove(Object key) {
        Entry<K,V> p = getEntry(key);
        if (p == null)			// 如果存在，则删除，如果不存在，直接返回null
            return null;

        V oldValue = p.value;
        deleteEntry(p);
        return oldValue;
    }
  在红黑树操作任何元素之前，需要找到这个元素，这里也不例外，删除之前，需要先找到这个元素
    final Entry<K,V> getEntry(Object key) {
        if (comparator != null)
            return getEntryUsingComparator(key);
        if (key == null)
            throw new NullPointerException();
        Comparable<? super K> k = (Comparable<? super K>) key;
        Entry<K,V> p = root;
        while (p != null) {			// 不断进行迭代比较
            int cmp = k.compareTo(p.key);
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
                return p;
        }
        return null;				// 确实没找到，则返回null
    }
  找到了之后，则正式开始删除指定的元素，总的纲领还是先删除节点，然后再平衡红黑树。不过与插入节点相比，删除节点的情况则要复杂好多
  删除的情况分为以下几类：
  a)被删除的节点是叶子节点，则只需要将它从其父节点中删除即可
  b)被删除节点p只有左子树，将p的左子树pL添加成p的父节点的左子树即可；
    被删除节点p只有右子树，将p的右子树pR添加成p的父节点的右子树即可。
  c)若被删除节点p的左、右子树均非空，有两种做法：
    i.将 pL 设为 p 的父节点 q 的左或右子节点（取决于 p 是其父节点 q 的左、右子节点），将 pR 设为 p 节点的中序前趋节点 s 的
      右子节点（s 是 pL 最右下的节点，也就是 pL 子树中最大的节点）。
    ii.以 p 节点的中序前趋或后继替代 p 所指节点，然后再从原排序二叉树中删去中序前趋或后继节点即可。（也就是用大于 p 的最小节点
      或小于 p 的最大节点代替 p 节点即可）。
    private void deleteEntry(Entry<K,V> p) {
        modCount++;
        size--;

        if (p.left != null && p.right != null) {	// 如果被删除节点的左子树、右子树都不为空，符合（情况c）
            Entry<K,V> s = successor(p);		// 用 p 节点的中序后继节点代替 p 节点
            p.key = s.key;				
            p.value = s.value;
            p = s;
        } 

        // 如果 p 节点的左节点存在，replacement 代表左节点；否则代表右节点。
        Entry<K,V> replacement = (p.left != null ? p.left : p.right);

        if (replacement != null) {		// 左、右节点至少存在一个
            replacement.parent = p.parent;
            if (p.parent == null)		// 如果 p 没有父节点，则 replacemment 变成父节点
                root = replacement;
            else if (p == p.parent.left)	// 如果 p 节点是其父节点的左子节点
                p.parent.left  = replacement;
            else				// 如果 p 节点是其父节点的右子节点
                p.parent.right = replacement;

            p.left = p.right = p.parent = null;	// p 节点删除

            if (p.color == BLACK)
                fixAfterDeletion(replacement);
        } else if (p.parent == null) { 		// 如果 p 节点没有父节点，也没有子节点，说明这就是一个单独的根节点，直接删除
            root = null;
        } else { 				// 没有孩子，也就是说这是一个叶子节点，只需要将它从其父节点上删去即可
            if (p.color == BLACK)		// 如果此节点是黑色，删去一个黑色节点，将导致此分支上末端到根节点途径的黑色节点减少
                fixAfterDeletion(p);		// 所以需要修复红黑树

            if (p.parent != null) {		// 如果 p 是其父节点的左子节点
                if (p == p.parent.left)
                    p.parent.left = null;
                else if (p == p.parent.right)	// 如果 p 是其父节点的右子节点
                    p.parent.right = null;
                p.parent = null;
            }
        }
    }
  如果实在难以理解，可以看下面的图：
  被删除的节点(30)只有左子树：
		18				18
	       /  \			       /  \
	     12    30			     12    23
   	      \    / 		--->>	      \     \
	      14  23			      14     26
	      /    \  			      /
             13     26			     13
  被删除的节点(12)只有右子树：
		18				18
	       /  \			       /  \
	     12    30			     14    30
   	      \    / 		--->>	     /     /
	      14  23			    13    23
	      /    \  			   	   \   
             13     26				    26	
  以上只有一侧子树的情况比较好处理，直接越过被删除节点，将父节点与其左、右节点续接在一起即可	
  被删除节点既有左子树，又有右子树：  
  i.将p(20)左子节点设为q(5)的子节点，将p(20)的右子节点设为pL子树中最大节点的的右子节点
		5  				5
	      /   \			       / \
	     3     20			      3   10
  		  /  \		--->>		 /  \
		 10   30 			8    15
		/  \				      \
	       8   15				       30	
  ii.用被删除节点后继(30)代替被删除节点(15)，这种方法也是代码中使用的方法：找到中序后继节点，替换即可，再删除此后继节点
		5				5
	       / \			       / \
	      3   15			      3   30
	         /  \		--->>	     	 /
     	        10   30   		        10
	       / 			       /
	      8				      8
  获取“P”节点的中序后继节点
    static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {	// 返回指定节点的继承人节点，如果没有则返回null
        if (t == null)
            return null;
        else if (t.right != null) {	// 指定节点的右支不为空
            Entry<K,V> p = t.right;
            while (p.left != null)	// 获取右支最小值，也是刚刚比指定节点大的值
                p = p.left;
            return p;
        } else {			
            Entry<K,V> p = t.parent;
            Entry<K,V> ch = t;
            while (p != null && ch == p.right) {
                ch = p;
                p = p.parent;
            }
            return p;
        }
    }
  中序后继节点的情况大致有以下几种：
		18				18			18
	       /  \			       /  \		       /  \
	     12    30			     12    23		      10   23	
   	    /  \   / 			    /       \   	       \     \
	   10  14  23			   10        26			11    26
	      /  \  \  							 \    
             13  15  26						          12		    	  	 
  节点P(12)的中序后继节点分别为：13	18	18
  对于一颗二叉查找树，给定节点t，其后继可以通过以下方法找到：
  a) t的右子树不空，则t的后继是其右子树中最小的那个元素
  b) t的右子树为空，则t的后继是其第一个向左走的祖先

  搞定了如何删除指定的节点后，原来是红节点还好，没有什么影响，删除了黑色节点，则需要调整红黑树的结构，使红黑树再次保持平衡
  下面的代码是将指定的节点删除后，替换的节点也转换到位，由于之前的删除，导致红黑树不再平衡，于是需要经过一些修正，再次回到平衡状态
  x节点是替换过来的值，当然，替换过来的值有些改变了位置，有些改变了key-value，但都没有改变颜色
    private void fixAfterDeletion(Entry<K,V> x) {		// 删除节点后修复红黑树
        while (x != root && colorOf(x) == BLACK) {		// 直到 x 不是根节点，且 x 的颜色是黑色，才进行调整
            if (x == leftOf(parentOf(x))) {			// 如果 x 是其父节点的左子节点
                Entry<K,V> sib = rightOf(parentOf(x));		// 获取 x 节点的兄弟节点

                if (colorOf(sib) == RED) {			// 如果 sib 节点是红色
                    setColor(sib, BLACK);
                    setColor(parentOf(x), RED);
                    rotateLeft(parentOf(x));
                    sib = rightOf(parentOf(x));			// 再次将 sib 设为 x 的父节点的右子节点
                }

                if (colorOf(leftOf(sib))  == BLACK &&		// 如果 sib 的两个子节点都是黑色
                    colorOf(rightOf(sib)) == BLACK) {
                    setColor(sib, RED);
                    x = parentOf(x);				// 让 x 等于 x 的父节点
                } else {
                    if (colorOf(rightOf(sib)) == BLACK) {	// 如果 sib 的只有右子节点是黑色
                        setColor(leftOf(sib), BLACK);
                        setColor(sib, RED);
                        rotateRight(sib);
                        sib = rightOf(parentOf(x));
                    }
                    setColor(sib, colorOf(parentOf(x))); 	// 设置 sib 的颜色与 x 的父节点的颜色相同
                    setColor(parentOf(x), BLACK);
                    setColor(rightOf(sib), BLACK);
                    rotateLeft(parentOf(x));
                    x = root;
                }
            } else { 						// 对称的，如果 x 是其父节点的右子节点
                Entry<K,V> sib = leftOf(parentOf(x));

                if (colorOf(sib) == RED) {			// 如果 sib 的颜色是红色
                    setColor(sib, BLACK);
                    setColor(parentOf(x), RED);
                    rotateRight(parentOf(x));
                    sib = leftOf(parentOf(x));
                }

                if (colorOf(rightOf(sib)) == BLACK &&		// 如果 sib 的两个子节点都是黑色
                    colorOf(leftOf(sib)) == BLACK) {
                    setColor(sib, RED);
                    x = parentOf(x);
                } else {
                    if (colorOf(leftOf(sib)) == BLACK) {	// 如果 sib 只有左子节点是黑色
                        setColor(rightOf(sib), BLACK);
                        setColor(sib, RED);
                        rotateLeft(sib);
                        sib = leftOf(parentOf(x));
                    }
                    setColor(sib, colorOf(parentOf(x)));
                    setColor(parentOf(x), BLACK);
                    setColor(leftOf(sib), BLACK);
                    rotateRight(parentOf(x));
                    x = root;
                }
            }
        }
        setColor(x, BLACK);
    }
  好吧，节点删除就此打住，我以后再专门开一个点来说明，这里就不再展开了。
  
