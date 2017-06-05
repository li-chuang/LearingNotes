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

  



