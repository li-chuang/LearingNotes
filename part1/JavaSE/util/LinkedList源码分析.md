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


4.构造方法
  public LinkedList() {		// 构建一个空的List
  }
  public LinkedList(Collection<? extends E> c) {	// 构建一个有初始值得List
        this();
        addAll(c);
  }
  
  
  
  

