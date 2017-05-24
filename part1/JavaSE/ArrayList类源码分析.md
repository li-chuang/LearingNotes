1.ArrayList实现自List接口，底层用数组实现，任何对象，包括null都可以存放到List中
  此类大致近似于Vector，不过最大的区别是Vector是线程安全的，对外公布的方法都有synchronized修饰，
  而ArrayList的方法不是线程安全的，好处就是操作速度快，毕竟synchronized修饰后，操作几乎都是线性的，
  会严重影响速度，而且可以根据需要自愿加上synchronized，而不必像Vector那样所有操作强制线程安全，
  所以，ArrayList的使用逐渐成为主流，Vector逐渐被淘汰


2.private static final int DEFAULT_CAPACITY = 10; // 初始化容量为10
  像这种容器类，扩容都是比较麻烦的操作，所以需要尽量减少扩容操作，
  这就需要我们在初始化一个容器类的时候，尽量预估容量大小，初始化的时候设置一个够用的容量，避免频繁的
  做扩充容量操作

3.private static final Object[] EMPTY_ELEMENTDATA = {};
  这个是空元素数组，用于ArrayList对象为空的时候，这个时候底层数组的内容也为空

4.private transient Object[] elementData;
  ArrayList的底层核心就是这个数组，当第一个元素加入的时候，空的ArrayList形如elementData == EMPTY_ELEMENTDATA
  的都想扩展容量到默认容量DEFAULT_CAPACITY = 10

5.几个构造方法
  public ArrayList() {
        super();
        this.elementData = EMPTY_ELEMENTDATA;
  }
  构造一个空的list,分配的初始化容量为10，有点像懒加载，当第一个元素加入的时候，才真正拥有
  public ArrayList(int initialCapacity) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
        this.elementData = new Object[initialCapacity];
  }
  这个构造方法，可以通过参数，设置底层的数组的容量
  public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray(); // c.toArray(),调用的是具体实现类的toArray(),
        size = elementData.length;
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
  }
  这里面以前有个比较重要的bug，就是在被注释的地方说明的，
  public Object[] toArray() 返回的类型不一定就是 Object[]，其类型取决于其返回的实际类型，
  参数中的“c”是某容器的实例，泛型不可知，可能是String，可能是Student，也可能是Object，但用Object[]整体表示，
  还是可以的，毕竟这是向上转型。但它的实际类型，却不是Object，而是“c”的实际类型，如String，Student等，
  所以，在此时，虽然这个数组名义上是Object[]，elementData理论上是Object数组对象，但向其中放入object对象则会出错
  	Object[] array = Arrays.asList("A").toArray();
  	System.out.println(array.getClass()); // class [Ljava.lang.String;
  	array[0] = new Object(); // cause ArrayStoreException
  
  在代码中，对elementData做了一个类型判断，然后将elementData转为真正的Object[]数组，这样数组中就可以存放任意对象了

6.trimToSize方法
  public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = Arrays.copyOf(elementData, size);
        }
  }
  这个方法用来去掉预留元素位置，将elementData的数组设置为ArrayList实际的容量，动态增长的多余容量被删除了，
  在代码中，用Arrays.copyOf(T [],int newLength)这个方法来截取elementData数组。
  一般在容器中，为了避免频繁申请存储空间，每次扩容的时候都会多申请一些，比如默认10的空间用完后，再加入一个元素，
  空间大小将扩大到15，但实际的元素个数为11；
  当元素比较多的时候，可以通过这种方法，去掉预留的元素位置，释放内存


7.容器的扩容机制
  public void ensureCapacity(int minCapacity) {
        int minExpand = (elementData != EMPTY_ELEMENTDATA) ? 0 : DEFAULT_CAPACITY;
        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
  }
  如果elementData中有值，则预定的minExpand为0，容量扩展为多少由minCapacity决定；
  如果elementData中没有值，最小扩展值minExpand为默认的容量10，传入的参数比10大才扩容为希望值，否则容量就是默认值。
  这个方法是用来设置ArrayList的容量的，除了用这个方法设置，构造方法中也可以将容量作为参数来进行容量设置，
  尤其是要进行大量元素插入的时候，如一万条数据，可以提前设置好容量，不然ArrayList将不断进行扩容，每次扩容又不断
  进行数据拷贝，严重影响效率。
  private void ensureExplicitCapacity(int minCapacity) { // 确保明确的容量
        modCount++;
        if (minCapacity - elementData.length > 0)	// 只要设置的值比已存在的数据多，就可以扩容
            grow(minCapacity);
  }
  private void grow(int minCapacity) {			//实际扩容逻辑
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1); // 注意这里，右移是除以2，所以新的容量是原来容量的1.5倍，
        if (newCapacity - minCapacity < 0)		    // 真正增加的容量只有原来的一半	
            newCapacity = minCapacity;			    // 容量扩展，谁大选谁
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);	// 最终进行数据拷贝
  }
  注意，容量扩展是用旧容量值右移一位实现的，然后加上原有的值，就成了新的容量。


8.contains方法
  public boolean contains(Object o) {
        return indexOf(o) >= 0;
  }
  indexOf(o)与lastIndexOf(o)作用类似，都是获取o对象在elementData数组中的位置，只不过一个从前面开始，一个从后面开始


9.clone()方法
  public Object clone() {
        try {
            @SuppressWarnings("unchecked")
            ArrayList<E> v = (ArrayList<E>) super.clone();	// 一个新的ArrayList对象，从这个clone()方法开始
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            throw new InternalError();
        }
  }
  这是一个浅克隆，元素本身没有被克隆，类似于只是获得了一个新的引用。

10.toArray()方法
  public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
  }
  这个方法可以将容器对象转为Object[]数组形式并返回，
  注意，返回的数组是通过数据拷贝得到的，所以和原来的数据elementData没有关系，可以自由操作改动
  并且：
    这个方法是容器类型与数组类型之间的桥梁
  
