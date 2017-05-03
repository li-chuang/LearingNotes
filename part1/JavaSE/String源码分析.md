1.private final char value[];
  String类的底层实际控制的是一个char类型的数组value[]，这个数组是私有的，而且为final，即这个属性不可以更改
  注：String字符串生成后具有持久性，不可以更改，其实就是这里的final的功效，value[]数组生成后不可以更改

2.private int hash;
  这里获取字符串的缓存码‘hash code’，hash的默认值为0，下面有从Object中继承的类hashCode(),可以获取每个类不同的
  的hash码。


3.String中构造方法很多，无参、String、char[]、byte[]、StringBuffer、StringBuilder都可以


4.public char charAt(int index) // 获取指定位置的char字符
  public int codePointAt(int index) // 获取指定位置的Unicode字符码，以十进制表示
  例如："A".codePointAt(0); //结果为65
       "a".codePointAt(0); //结果为97
       "李".codePointAt(0); //结果为26446
       Unicode是ASCII的扩充，前128位完全一致，所以"a"的Unicode码就是ASCII码，十进制表示为“97”
       中、日、韩的三种文字占用了Unicode中0x3000到0x9FFF的部分，26446为0x674E

5.public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin); // 复制原字符数组到指定的目标数组
  各参赛含义：原数组开始位置，原数组结束位置，目标数组，目标数组的偏移
  底层代码：System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
  还有void getChars(char dst[], int desBegin);

6.public byte[] getBytes(String charsetName); // 将字符串转化为指定编码格式的字节数组
  一个byte有8个bit,一般表示汉字的时候，只用前面的7位，导致各个byte均为负值，且数字小于128
  GBK用2个byte表示一个汉字，utf-8用3个byte表示一个汉字
  编码格式不同，表示方法也不一样

7.public boolean equals(Object anObject);// 此方法用于比对是否相等
  作为一个来自Object类中的基本方法，equals()被大量的其他类根据自己的特定需求进行重写，这里也不例外。
  在代码中，可以看到首先是用‘==’进行比对，这比较的实际是hashcode，也即假如hashcode一致，（存储地址一致，）
  两个字符串自然也是一致的。
  在此之后，判断是否为String类型，然后将字符一位一位的进行比对，所有对应位数上的字符均一致，则认为此字符串
  是相同字符串。


8.public boolean contentEquals(StringBuffer sb);// 参数可以是其他CharSequence
  其实也是用来判断字符串是否一致，字面一致即可

9.public boolean equalsIgnoreCase(String anotherString);// 这个方法其实用得很多，比较字符串是否一致，不区分大小写
  会用到regionMatches方法，

10.public int compareTo(String anotherString);// 比较字符串的大小，相等时为0
  两个字符串（abc,ade）进行比较是从左边开始，比较Unicode，如果不一致，则返回值为两个字符的Unicode差值；
  两个字符串（abc,abcdef）进行比较，返回的是两个字符串长度的差值
  另外compareToIgnoreCase(String str)作用差不多，也是比较大小，不过是不区分大小写，原因就是此方法内单个char字符
  进行比较时，假如不一致，就进行一次Character.toUpperCase(c)操作，再不一致就进行一次Charcater.toLowerCase(c)操作
  返回的还是两个字符的Unicode差值


11.public boolean regionMatchs(int toffset, String other, int ooffset, int len);// 用于比较字符串指定部分是否一致
  参数说明：this字符串偏移，other字符串，other字符串偏移，要比较的长度
  其实内部实现原理还是字符数组挨个比较
  另外还有regionMatches(boolean ignoreCase, int toffset, String other, int ooffset, int len);与上面的区别是多了
  一个参数boolean ignoreCase,设置比较是否忽略大小写
  当然，忽略大小写底层原理就是toUpperCase与toLowerCase;


12.public boolean startsWith(String prefix, int toffset);// 判断从指定toffset偏移处开始的字符串与this字符串是否一致
   public boolean startsWith(String prefix); // toffset默认为0，也就是从开头比较
   public boolean endsWith(String suffix) {  // 判断指定字符串与this字符串的末尾是否一致
   	return startsWith(suffix, value.length - suffix.value.length);
   }
   其实它们更准确的说是在比较子字符串是否一致，只不过默认比较开头或者结尾的多一些，也就用习惯了。


13.public int hashCode() { // 获取hashcode值
   	int h = hash;
   	if( h == 0 && value.length > 0){
		char val[] = value;
		for( int i = 0; i < value.length; i++){
			h = 31 * h + val[i];
		}
		hash = h;
	}
  	return h;
   }
  其实上面的实现的数学表达式就是: s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
  其中s[i]是String的第i个字符，n是String的长度
  具体的计算步骤为：
  String str = 'abcd';

  h = 0
  value.length = 4

  val[0] = a
  val[1] = b
  val[2] = c
  val[3] = d
 
  h = 31 * 0 + a = a

  h = 31 * (31 * 0 + a) + b = 31 * a + b;

  h = 31 * (31 * (31 * 0 + a) + b) + c = 31* (31 * a + b) + c = 31 * 31 * a + 31 * b + c
  
  h = 31 * (31 * 31 * a + 31 * b + c) + d = 31 * 31 * 31 * a + 31 * 31 * b + 31 * c + d
  
  ...
  h = 31 ^ (n-1) * val[0] + 31 ^ (n-2) * val[1] + 31 ^ (n-3) * val[2] + ...+ val[n-1]
  
  当然，还有一个点需要解释清楚，为什么要用31这个数，解释如下
  a.31是素数，最终计算出来的结果可以减少冲突
  b.31可以由i*31 == (i<<5) - 1 来表示，也就是左移5位再减1，现在很多虚拟机都有相关优化，
    可以提高算法效率
  c.选择系数的时候要尽量大一些，这样计算出来的hashcode地址也大，可以减少冲突，查找起来
    效率也高，也不能太大，这样数据容易溢出
  d.31只占用5bit,相乘造成数据溢出的概率较小
  最后，散列码生成的原则是：不同的字符串应该有不同的散列码


14.public int indexOf(int ch, int fromIndex);// 获取指定字符在字符串中第一个出现的位置
  第二个参数为开始搜索位置，默认为0，即从最前面开始
  内部原理也很简单，从fromIndex处开始，挨个比对，成功后传回位置索引
  而lastIndexOf()则是从后往前对比，原理其实是一样的
  注意：第一个参数可以是字符，也可以是此字符对应的Unicode码，如indexOf(65)与indexOf('A')
        是一回事，一样的。这是否意味着诸如‘A’这样的字符在JVM层面是自动转化为Unicode的呢？

15.public int indexOf(String str , int fromIndex);//这个是查找指定字符串在this字符串中的位置
  上面的方法是找‘字符’的位置，这里是找‘字符串’的位置
  lastIndexOf(String str, int fromIndex)方法也差不多

16.substring(int beginIndex),substring(int beginIndex, int endIndex)
  以及subSequence(int beginIndex , int endIndex)都是获取子字符串

17.public String concat(String str);// 连接两个字符串
  原理：
  char buf[] = Arrays.copyOf(value, len + otherLen);// 将原有数组扩容，形成新数组
  str.getChars(buf, len);// 将str字符串复制到目标字符数组buf的指定开始位置len
  // 底层代码 System.arraycopy(value, 0, dst, dstBegin, value.length);
  return new String(buf, true);

18.replace/replaceAll/replaceFirst 底层都是用Pattern正则表达式匹配的
   public String[] split(String regex) 是进行字符分割的，注意返回的是数组
   toLowerCase与toUpperCase是进行大小写转换的 ，注意，此方法还可以带Locale参数 
   trim,去掉前后端的空格，原理是判断前后是否有空格，记录偏移位置，然后一个substring(start, end)
	搞定

19.public char[] toCharArray(){
	char result[] = new char[value.length];
	System.arraycopy(value, 0, result, 0, value.length);
	return result;
   }
  注意，虽然String底层也就是char[],但这里的话，还是复制了一个返回的，毕竟String中的value
        是用final修饰的

20.public static String format(String format, Object ... args);//用于进行格式化


21.public static String valueOf();// 各种参数的valueOf
   几乎可以将其他任何类型转换为String类型
   Object obj => return (obj == null) ? "null" : obj.toString();
   char[] data[] => return new String(date);
   boolean b => return b ? "true" : "false";
   int i => return Integer.toString(i);// Long, Float, Double类似

22.public native String intern();// 这是有关常量池的问题，
  str.intern()即先到常量池去取这个字符串


























