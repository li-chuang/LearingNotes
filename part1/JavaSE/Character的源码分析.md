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

6.
