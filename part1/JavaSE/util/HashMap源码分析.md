1.HashMap允许key和value都是null
  HashMap大致等于Hashtable，区别在于HashMap方法都不是同步synchronized的，另外HashMap允许null
    注意，Java的设计者是不支持全部同步的，因为这样很影响性能而且 不够灵活。最后的办法是根据实际需要
    在代码块中加synchronized，或者使用线程安全的ConcurrentHashMap，于是慢慢的Hashtable就淘汰了。
  HashMap不保证映射的顺序性，而且，也不保证随着时间的推移原有的顺序依然保持一致，不发生改变。


2.默认的初始容量是16（1<<4），
  默认的加载因子是（0.75），
  默认的最大容量是（1<<30），主要是用来避免初始化时设置过大的容量，当初始化时设置的容量大于（1<<30）时
    容量设置为（1<<30）
