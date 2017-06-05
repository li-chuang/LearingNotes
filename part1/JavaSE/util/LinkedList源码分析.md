1.LinkedList底层是双向链表，里面可以放入任何元素，包括null
  LinkedList中的各种操作实际上是对双向链表的操作
  LinkedList中的各种操作不是线程安全的“not synchronized”，多线程的时候需要用synchronized修饰各种操作


2.静态内部类Node
  private static class Node<E> {
	E item;
	Node<E> next;
	Node<E> prev;
	Node(Node<E> prev, E element, Node<E> next){
		this.item = element;
		this.next = next;
		this.prev = prev;
	}
  }
  私有，是让外部方法无法访问它；
  静态，说明它的实例化不依赖于外部类的实例化
  
  LinkedList的底层就是双向链表，也就是一个个的Node节点，除了存储数据的item外，
  还有指向前一个节点的prev与指向后一个节点的next，这些构成了双向链表的基本单元。


3.指向前后的指针
  transient Node<E> first;
  transient Node<E> last;
  注意，按照C中的说法它们是指针指向前后两个节点的指针，不过在Java中，它们被认为是两个对象，对象的类型就是Node<E>
  头部节点与尾部节点在链表中是特殊节点，不仅仅用来存储数据，而且用来作为查询、插入、删除逻辑的起始点进行操作，
  所以，first与last是一种标签，它们才是操作点，插入的其它节点都需要迎合first与last的要求：
  当链表为空的时候，first与last节点自然也是为空的，即first == null && last == null
  当链表不为空的时候，first作为头部节点，前向指针为空，即first.prev == null && first.item != null
      同理，last作为尾部节点，后向指针自然也是空的，即last.next == null && last.item != null


4.构造方法
  public LinkedList() {		// 构建一个空的List
  }
  public LinkedList(Collection<? extends E> c) {	// 构建一个有初始值得List
        this();
        addAll(c);
  }


5.向前插入双向链表
  private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
  }
  说明：数据需要插入到链表的最前端，首先，原来在头部的数据要转移到临时节点f中，节点f中保存的是上一个头部节点的数据，
    然后创建一个新的节点，将数据“e”放入其中，并且将尾部指针指向原来的数据节点f，
    再用新的节点对first节点进行赋值，
    此后进行判断，如果原来的节点f == null，则说明之前节点为空，没有装数据，这是插入的第一个值，于是把头尾设置为一样的
      如果f节点有值，first节点的数据被替换后，原来的临时节点的数据转正，将f的前部指针指向新的数据。
    最后，更新长度。

  注意，这个方法是私有的，其他的方法调用此方法完成节点的插入，例如：
  public void addFirst(E e) {
        linkFirst(e);
  }


6.向后插入双向链表
  void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
  }
  注意，last是尾部节点，向尾部插入的时候，需要把last对像置换出来，不然此对象就会埋没在链表中了。
  与向前插入差不多，向后插入的时候也需要专门将last节点先保存在一个临时节点中，当将新节点赋值到last节点后，再将临时节点的指针
    指向尾部节点last。
  
  向后插入是双向链表的默认插入方式，很多方法都是基于此方法实现的。
  

7.插入数据到链表指定的节点之前
  void linkBefore(E e, Node<E> succ) {		// 需要确保succ != null
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
  }
  这就是一个标准的双向链表节点插入；


8.删除非空的首部节点
  private E unlinkFirst(Node<E> f) { 	// 确保f节点为首部节点，并且f节点不为空节点，即f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; 
        first = next;		// 原来首部节点的下一个节点现在赋值为首部节点
        if (next == null)
            last = null;
        else
            next.prev = null;	// 首部节点的前向指针为空，前面已经没有节点了。
        size--;
        modCount++;
        return element;		// 返回删除值
  }
  删除非空的尾部节点，与此非常的类似
  private E unlinkLast(Node<E> l) {	// 确保l节点为尾部节点，且不为空，即l == last && l != null;
        final E element = l.item;
        final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; 
        last = prev;		// 原来尾部节点的上一个节点现在接替为尾部节点，
        if (prev == null)
            first = null;
        else
            prev.next = null;	// 尾部节点没有下一个节点
        size--;
        modCount++;
        return element;		// 返回删除值
  }
 

9.删除指定的节点
  E unlink(Node<E> x) {		// 确保 x != null;
        final E element = x.item;
        final Node<E> next = x.next;	// 获得指定节点的前一个节点与后一个节点
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;
        } else {
            prev.next = next;		// 将前一个节点的next指针指向下一个节点next
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;		// 将后一个节点的prev指针指向前一个节点prev
            x.next = null;
        }

        x.item = null;			// 先切断x节点的所有指针，然后将节点的数据值设置为null
        size--;
        modCount++;
        return element;
  }

10.获取first与last的值
  public E getFirst() {			// 获取第一个节点的数据值
        final Node<E> f = first;	// 注意，first是一个特殊节点，此节点一直都存在，并且一直都是双向链表的第一个节点
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
  }
  public E getLast() {			// 获取最后一个节点的数据值
        final Node<E> l = last;		// 注意，同first一样，last也是一个特殊节点，此节点一直都存在，且一直是双向链表的最后一个节点
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
  }

11.添加首部/尾部值与移除首部/尾部值
  removeFirst、removeLast、addFirst、addLast、add、remove都是在上面方法的基础上的封装，它们都是public访问权限的
  public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
  }
  public void addFirst(E e) {
        linkFirst(e);
  }
  这些都是开放的接口，可以供外界使用


12.将列表转换为数组
  public Object[] toArray() {
        Object[] result = new Object[size];
        int i = 0;
        for (Node<E> x = first; x != null; x = x.next)
            result[i++] = x.item;
        return result;
  }
  可以看到，转换的核心是一个for循环，通过这个for循环给数组赋值

