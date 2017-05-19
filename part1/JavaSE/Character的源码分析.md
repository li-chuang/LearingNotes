1.Character类作为基本类型char的包装类，和其他包装类一样，都是用final修饰的，都不准再有继承。
  Character中的字符信息基于Unicode标准，从U+0000到U+FFFF的叫做基本字符集，用16位两个字节表示，
  大于U+FFFF的称之为增补字符集，用四个字节表示，默认编码方式是UTF-16,注意“高代理项”与“低代理项”

2.由于Character中的字符是基于Unicode标准的，而Unicode字符都是有一个对应的数字编码，所以接受一个数字值是没问题的
  char c = '马'  // 将这个字符形式对应的Unicode编号值赋给变量
  char c = 39532
  char c = 0x9a6c
  char c = '\u9a6c'
  它们都是一样的，本质都是将Unicode编号39532赋给了字符
  再次强调，不是将‘马’这个符号赋给了变量，而是将‘马’对应的Unicode编号赋值给了变量,剩下的三种才是正规的赋值方式，
    第一种只是比较方便直接一点而已。
  

3.设置最小进制数为2，最大进制数为36
  设置最小值为\u0000，最大值为\uFFFF


4.内部类UnicodeBlock的方法
  public static UnicodeBlock of(char c){
	return of((int)c);  // 注意这里，所有的字符，都是转为Unicode的十进制编号进行操作的
  }

  public static UnicodeBlock of(int codePoint){
	 int top, bottom, current;
         bottom = 0;
         top = blockStarts.length;
         current = top/2;

         // invariant: top > current >= bottom && codePoint >= unicodeBlockStarts[bottom]
         while (top - bottom > 1) {
             if (codePoint >= blockStarts[current]) {
                 bottom = current;
             } else {
                 top = current;
             }
             current = (top + bottom) / 2;
         }
         return blocks[current];
  }
  此方法的用途是返回指定字符所属类型的名称，blockStarts数组中保存的是各个类型的起始值，
  blocks数组中保存的是各个类型的名称；
  此处用到了二分法，最开始current = top/2，然后随着判断的情况不断调整current的值，最终返回blocks[curent]

  使用举例：
  System.out.println(new String("\u0031"));				// 1
  System.out.println(Character.UnicodeBlock.of('1'));			// BASIC_LATIN
  System.out.println(Character.UnicodeBlock.forName("BASIC_LATIN"));	// BASIC_LATIN

  System.out.println(new String("\u06DE"));				//  ۞  
  System.out.println(Character.UnicodeBlock.of('۞'));			// ARABIC
  System.out.println(Character.UnicodeBlock.forName("ARABIC"));		// ARABIC

5.内部枚举UnicodeScript，情况与上面UnicodeBlock一样，就不做深入分析了


6.字符类中的缓存
  private static class CharacterCache {
        private CharacterCache(){}
        static final Character cache[] = new Character[127 + 1];
        static {
            for (int i = 0; i < cache.length; i++)
                cache[i] = new Character((char)i);
        }
  }
  说明：
  i.这是单例模式极好的案例，私有的构造方法，final修饰的变量，static静态块包围的代码，确保只在类加载的时候执行一次
  ii.缓存起来的是ASCII码，它们是用得最广泛的，缓存起来很实用
  iii.最底层也是new Character(char ch)得来的，然后放入缓存中保存，new即是分配了内存了


7.获取一个字符的实例
  public static Character valueOf(char c) {
        if (c <= 127) { // must cache
            return CharacterCache.cache[(int)c];
        }
        return new Character(c);
  }
  代码挺简单，但有一点需要注意：
  i.ASCII码的可以直接在缓存中取，没在缓存区中的，再new出来
  ii.以后获取一个字符的实例时，注意要使用valueOf(char c)方法，而不要直接new Character(char c)，直接new出一个实例
    绕过了缓存，和缓存中的字符没有任何关系了，只要new出来的，自然重新分配内存，用‘==’进行比较时自然返回‘false’了
  iii.强调一次，用valueOf(char c)，不要使用new Character(char c)


8.获取此包装类中的基础字符
  public char charValue(){
	return value; // value是这个包装类的核心，包装类就是用来包装这些基础数据类型的
  }


9.hashCode()与equals()
  public int hashCode(){
	return (int)value; 
  }

  public boolean equals(Object obj){
	if(obj instanceof Character){
  		return value == ((Character)obj).charValue();
  	}
	return false;
  }
  Character中的比较挺简单，和Integer中差不多，因为Character底层是由Unicode实现的，字符都可以用数字表示，
  所以区别它们就非常的简单了。


10.isValidCodePoint方法
      	public static boolean isValidCodePoint(int codePoint) {
        // Optimized form of: codePoint >= MIN_CODE_POINT && codePoint <= MAX_CODE_POINT
        int plane = codePoint >>> 16;
        return plane < ((MAX_CODE_POINT + 1) >>> 16);
  }
  用于判断给定的数字在Unicode中是否有效，Unicode的范围最大到1114111 [0x10ffff]，最小值0 [0x0]
  这里比大小的方式很有特色，无符号右移16位后再进行比较
  
