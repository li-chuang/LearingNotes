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
  这里比大小的方式很有特色，无符号右移16位后再进行比较，简化了比较操作。
  MAX_CODE_POINT = 0X10FFFF，加上1后为0x110000,无符号右移后为0x11;
  位数比较大的比较，并且其中一个余下的位数都是‘0’，可以这样移位后再比较，确实比较巧妙。


11.isBmpCodePoint方法
  public static boolean isBmpCodePoint(int codePoint) {
        return codePoint >>> 16 == 0;
  }
  判断给定的字符数字是否在“基本多文本平面”内，即是否在0x0000到0xFFFF范围内，也就是是否可以用一个
  单个的char（16位）表示这个字符


12.isSupplementaryCodePoint方法
  public static boolean isSupplementaryCodePoint(int codePoint) {
        return codePoint >= MIN_SUPPLEMENTARY_CODE_POINT
            && codePoint <  MAX_CODE_POINT + 1;
  }
  判断指定字符是否是“增补字符”，范围在0x10000到0x10FFFF之间。这个还算正常，直接进行比较

13.isHighSurrogate方法
  public static boolean isHighSurrogate(char ch) {
        return ch >= MIN_HIGH_SURROGATE && ch < (MAX_HIGH_SURROGATE + 1);
  }
  判断字符是否处于“高位代理项”区间内，在这个区间内的值不代表一个字符，而是用来在UTF-16编码格式中表示“高位代理”
  范围在0xD800[1101 1000 0000 0000]到0xDBFF[1101 1011 1111 1111]，注意最前边的110110区段。
  在UTF-16格式中，大部分的字符都用两个字节表示，但是也有一些极其少有的字符，在增补字符集中，必须要四个字节才可以表示，
  分成前后两部分（两个WORD），前部WORD格式是以“110110”开头，后部WORD格式以“110111”开头，
  为了避免Unicode解析的时候混淆，所以Unicode码表中的110110/1部分是空置的，不代表任何字符，只作为指引前后WORD的符号存在着。
  在实际使用中，如果一个两字节字符形式如[1101 10xx xxxx xxxx]，范围在0xD800到0xDBFF，即代表此WORD是作为一个四字节字符的前段存在的，
  相应的，如果一个两字节字符形式如[1101 11xx xxxx xxxx]，范围在0xDC00到0xDFFF，即代表此WORD是作为一个四字节字符的后段存在的，
  除此之外的两字节字符，才可以直接解析为Unicode。

14.isLowSurrogate方法
  public static boolean isLowSurrogate(char ch) {
        return ch >= MIN_LOW_SURROGATE && ch < (MAX_LOW_SURROGATE + 1);
  }
  判断字符是否处于“低位代理项”区间内，范围是0xDC00到0xDFFF
  这个范围内的值，和上面所说的一样，不指代一个特定的Unicode字符，而只是作为UTF-16编码格式的“低位代理”存在着
  原理分析参考上一段。

15.isSurrogate方法
  public static boolean isSurrogate(char ch) {
        return ch >= MIN_SURROGATE && ch < (MAX_SURROGATE + 1);
  }
  判断是否处于“代理区”，范围在0xD800到0xDFFF，这个区间是前两段的结合，包括“高位代理”与“低位代理”两部分，
  在这个区间的值，不代表Unicode中特定的字符，而只是用来代表UTF-16编码中的“增补字符”代理区。

16.isSurrogatePair方法
  public static boolean isSurrogatePair(char high, char low) {
        return isHighSurrogate(high) && isLowSurrogate(low);
  }
  判断一个代理对的值是否都是有效的，由此判断一个四字节的字符是否是合法的


17.charCount方法
  public static int charCount(int codePoint) {
        return codePoint >= MIN_SUPPLEMENTARY_CODE_POINT ? 2 : 1;
  }
  判断给定的值是用几个char表示，值大于0x10000的，用两个char,否则就是用一个char表示
  
18.toCodePoint方法
  public static int toCodePoint(char high, char low) {
        // Optimized form of:
        // return ((high - MIN_HIGH_SURROGATE) << 10) + (low - MIN_LOW_SURROGATE) + MIN_SUPPLEMENTARY_CODE_POINT;
        return ((high << 10) + low) + (MIN_SUPPLEMENTARY_CODE_POINT - (MIN_HIGH_SURROGATE << 10) - MIN_LOW_SURROGATE);
  }
  将增补区间的“高低代理对”转换为Unicode字符值，
  第一个参数为“高位代理”区间的值，第二个参数为“低位代理”区间的值，去除前段的110110/1后，合并为Unicode编码值
  举例说明：
  “𝒜”的UTF-16/UTF-16BE (hex) 编码为0xD835 0xDC9C (d835dc9c) ，用四个字节表示，第一个WORD为0xD835，第二个WORD为0xDC9C
  二进制分别为[1101 1000 0011 0101],[1101 1100 1001 1100]，分别去掉前六位[0000 1101 01],[00 1001 1100]，
  合并后[0000 1101 0100 1001 1100]，转换为16进制就是0x0D49C，再加上0x10000，最后的结果就是这四个字节所表示的Unicode码了
  最后的Unicode码位0x1D49C。
  以上是Unicode增补字符用UTF-16编译成四字节的逆过程，原理就是这样的，不过在代码中有一些优化。
  注释掉的部分理解为：((0xD835 - 0xD800) << 10) + (0xDC9C - 0xDC00) + 0x10000
  新的方案为：((0xD835 << 10) + 0xDC9C ) + (0x10000 - (0xD800 << 10) - 0xDC00)
  注意，以上两个写法实际上是一样的，只是将运算换了一个位置而已。


19.codePointAt方法
  public static int codePointAt(CharSequence seq, int index) {
        char c1 = seq.charAt(index++);
        if (isHighSurrogate(c1)) {
            if (index < seq.length()) {
                char c2 = seq.charAt(index);
                if (isLowSurrogate(c2)) {
                    return toCodePoint(c1, c2);
                }
            }
        }
        return c1;
  }
  下面还有与之类似的方法，都是返回指定字符串/字符数组中index处字符的Unicode值
  如codePointAt(char[] a, int index) 也是这样，
  而codePointBefore(char[] a, int index)则是获取字符串/字符数组指定索引index前面的字符的Unicode值

20.highSurrogate方法与lowSurrogate方法
  public static char highSurrogate(int codePoint) {
        return (char) ((codePoint >>> 10) + (MIN_HIGH_SURROGATE - (MIN_SUPPLEMENTARY_CODE_POINT >>> 10)));
  }
  public static char lowSurrogate(int codePoint) {
        return (char) ((codePoint & 0x3ff) + MIN_LOW_SURROGATE);
  }
  分别获取（四字节表示）Unicode码的高位代理和低位代理
  
  在这里继续解释下“高位代理”与“低位代理”的概念：
  一个char是16位，理论上只有65536种取值，这个数量对于表示Unicode码值是远远不够的。那对于大于65536的Unicode码值，
  用16位的char如何表示呢？Unicode标准制定组想到了一个好办法，从这65536种取值中，拿出2048个值，规定它们是代理，
  让它们两个为一组，来代表编号大于65536的字符。
  更具体一点说，编号为0xD800到0xDBFF的称之为High Surrogate，共1024个，编号为0xDC00到0xDFFF的称之为Low Surrogate，
  也是1024个，它们两两组合出现，于是又多出了1048576种字符
  
