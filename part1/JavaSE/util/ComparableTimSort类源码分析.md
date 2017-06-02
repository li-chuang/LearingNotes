1.ComparableTimSort与TimSort几乎重复，唯一不同的地方在于去掉了TimSort中的比较器


2.private static final int MIN_MERGE = 32；
  这是能够使序列合并的最小值，比这个更短的需要使用二分插入排序使序列加长，如果整个的数组都比这个短，
  那么将不执行合并操作。
  这个常量的值必须2的幂，可以修改此值，但不建议这么做，主要是修改后，连带还要改其他的一些东西。

3.private final Object[] a;
  这里存放将要被排序的数组

4.private static final int MIN_GALLOP = 7;
  判断数据顺序连续性的阈值。
  简单的理解就是，归并过程中有两个数列，比较的时候，有个数列连续有{MIN_GALLOP}个元素都比另一个数列的第一个元素小，
  那就应该数一下后面到底还有多少个元素比另一个数列的第一个元素小。数完之后一次copy过去，减少copy的次数。
  这就是“飞腾模式”

5.private static final int INITIAL_TMP_STORAGE_LENGTH = 256;
  归并排序中临时数组的最大长度，数组的长度也可以根据需求增长。

6.parivate Object[] tmp;
  合并用的临时数组

7.private int stackSize = 0; // 栈中run的数量
  private final int[] runBase;
  private final int[] runLen;
  stackSize为栈中待归并的run的数量。一个run i的范围从runBase[i]开始，一直延续到runLen[i]。前一个run的结尾总是下一个run的开头。
  所以下面的等式总是成立: runBase[i] + runLen[i] == runBase[i+1];

8.构造方法
  private ComparableTimSort(Object[] a) {
        this.a = a;
        int len = a.length;
        @SuppressWarnings({"unchecked", "UnnecessaryLocalVariable"})
        Object[] newArray = new Object[len < 2 * INITIAL_TMP_STORAGE_LENGTH ? len >>> 1 : INITIAL_TMP_STORAGE_LENGTH];
        tmp = newArray;
	//  以上是分配临时数组tmp的空间，“>>>”无符号右移相当于“除以2”，最终tmp的长度为排序数组长度的一半或者临时数组的最大长度256

        int stackLen = (len <    120  ?  5 :
                        len <   1542  ? 10 :
                        len < 119151  ? 24 : 40);
        runBase = new int[stackLen];
        runLen = new int[stackLen];
        //  以上是分配存储run的栈空间，它不能在运行时扩展。
  }
  注意，这个构造方法是私有的，也就是只能在类的内部创建和使用。创建这个实例是为了保存排序过程中的状态变量。


9.真正的肉戏上场，这两个方法是此类中唯二的API
    static void sort(Object[] a) {
          sort(a, 0, a.length);		// a为需要排序的数组，默认是全部要排序
    }

    static void sort(Object[] a, int lo, int hi) {
        rangeCheck(a.length, lo, hi);	// 验证数组的索引是否都正常
        int nRemaining  = hi - lo;	// 获取需要排序数组的长度
        if (nRemaining < 2)
            return;  			// 数组长度小于2（0或1），不需要排序，直接退出

        if (nRemaining < MIN_MERGE) {	// 如果数组长度小于32，不用归并这么复杂了，直接二分插入排序就可以了
            int initRunLen = countRunAndMakeAscending(a, lo, hi);	// 将数组a进行优化，降序变升序
            binarySort(a, lo, hi, lo + initRunLen);	// 二分插入排序，low,hight都是范围，
            return;					// 最后一个参数表示从此处开始排序，因为前面已经有一段升序了，不必从首位开始
        }

	// 从左到右遍历数组，找到自然排序好的序列，把短的自然升序序列通过二分插入排序扩展到minRun长度的升序序列。
	// 最后合并栈中的所有升序序列，保证规则不变
        ComparableTimSort ts = new ComparableTimSort(a);	// 新建ComparableTimSort对象，保存其中栈信息
        int minRun = minRunLength(nRemaining);		// 获取最小run长度，小于32，返回n，大于32且是2的幂，返回16，其他则返回一个接近16的数
        do {							
          
            int runLen = countRunAndMakeAscending(a, lo, hi);	// 整理整个序列，降序变升序，返回第一个升序后最开始降序的地方

            if (runLen < minRun) {		// 如果自然升序的长度不够minRun，就把 min(minRun,nRemaining)长度的范围内的数列排好序
                int force = nRemaining <= minRun ? nRemaining : minRun;
                binarySort(a, lo, lo + force, lo + runLen);	// 二分插入排序，范围从 lo 到 lo+force ，从lo + runLen处开始有降序
                runLen = force;
            }

            ts.pushRun(lo, runLen);	// 把已经排好序的数列压入栈中，检查是不是需要合并
            ts.mergeCollapse();

            lo += runLen;		// 把指针后移runLen距离，准备开始下一轮片段的排序
            nRemaining -= runLen;	// 剩下待排序的数量相应的减少 runLen
        } while (nRemaining != 0);

        // Merge all remaining runs to complete sort
        assert lo == hi;
        ts.mergeForceCollapse();	// 合并完所有的run完成整个排序
        assert ts.stackSize == 1;	// 最终结果应该是缓存中的有序序列只有一条
    }


10.被优化的二分插入排序算法
  使用二分插入排序算法给指定一部分数组排序。这是给小数组排序的最佳方案
  如果开始的部分数据是有序的那么我们可以利用它们。这个方法默认数组中的位置lo(包括在内)到start(不包括在内)的范围内是已经排好序的。
  参数说明：a      被排序的数组；        
	    lo     待排序范围内的首个元素的位置；
	    hi     待排序范围内最后一个元素的后一个位置
 	    start  待排序范围内的第一个没有排好序的位置，确保 (lo <= start <= hi)
  private static void binarySort(Object[] a, int lo, int hi, int start) {
        assert lo <= start && start <= hi;
        if (start == lo)		// 如果start 从起点开始，做下预处理；也就是原本就是无序的。
            start++;
        for ( ; start < hi; start++) {		// 从start位置开始，对后面的所有元素排序
            @SuppressWarnings("unchecked")
            Comparable<Object> pivot = (Comparable) a[start];	// pivot 代表正在参与排序的值，

            int left = lo;			// 设置pivot所属范围的left和right索引值
            int right = start;
            assert left <= right;

            //保证的逻辑: pivot >= all in [lo, left) && pivot <  all in [right, start).
            while (left < right) {
                int mid = (left + right) >>> 1;		// 无符号右移，等同于“除以2”，获取中间值
                if (pivot.compareTo(a[mid]) < 0)	// 将pivot与中间值比较，判断是在前半部分还是后半部分
                    right = mid;
                else
                    left = mid + 1;
            }
            assert left == right;			// 退出后，保证left与right集中在一处

            // 此时，仍然能保证: pivot >= [lo, left) && pivot < [left,start)
	    // 所以，pivot的值应当在left所在的位置，然后需要把[left,start)范围内的内容整体右移一位
            // 腾出空间。如果pivot与区间中的某个值相等，left指正会指向重复的值的后一位，所以这里的排序是稳定的。
            int n = start - left;  	// 需要移动的范围的长度
           
            switch (n) {		// switch语句是一条小优化，1-2个元素的移动就不需要System.arraycopy了。
                case 2:  a[left + 2] = a[left + 1];
                case 1:  a[left + 1] = a[left];
                         break;
                default: System.arraycopy(a, left, a, left + 1, n);
            }
            a[left] = pivot; 		// 移动过之后，把pivot的值放到应该插入的位置，就是left的位置了
        }
  }
  举例说明：{3,6,7,8,2,4,9,1,5,10}
  可知 lo = 0, start = 4, hight = 9 , pivot = a[start] = a[4] = 2
  left与right是临时参数，负责二分查找，当left == right 时，此处就是pivot的位置


11.将run中降序部分优化为升序
  返回值为 从首个元素开始的最长升序子序列的结尾位置+1
    private static int countRunAndMakeAscending(Object[] a, int lo, int hi) {
        assert lo < hi;		// 这里是一个断言，lo是低位索引，hi是高位索引的后一位
        int runHi = lo + 1;	
        if (runHi == hi)	// 注意，hi为序列索引高位的后一位
            return 1;

        // 从头到尾检查，如果有降序排序，则将之翻转为升序排序
        if (((Comparable) a[runHi++]).compareTo(a[lo]) < 0) { // 降序
            while (runHi < hi && ((Comparable) a[runHi]).compareTo(a[runHi - 1]) < 0)
                runHi++;
            reverseRange(a, lo, runHi);		// 确定降序的范围后翻转
        } else {                                // 升序
            while (runHi < hi && ((Comparable) a[runHi]).compareTo(a[runHi - 1]) >= 0)
                runHi++;
        }

        return runHi - lo;	// runHi是一个相对值，相减后获得run中第一个升序排序的长度
    }


12.将降序序列翻转为升序
    private static void reverseRange(Object[] a, int lo, int hi) {
        hi--;			// 这里再次强调，hi为序列最后一位的下一位，所以在使用的时候需要先前移一位
        while (lo < hi) {
            Object t = a[lo];	// 翻转的原理就是不断的换位
            a[lo++] = a[hi];
            a[hi--] = t;
        }
    }

13.获取参与归并run的最短长度
  返回指定长度数组的最小可接受run长度。如果自然排序的长度小于此长度，那么就通过二分查找排序扩展到此长度。
  粗略的讲，计算结果是这样的：
  a.如果 n < MIN_MERGE, 直接返回 n。（太小了，不值得做复杂的操作）；
  b.如果 n 正好是2的幂，不断进行 n / 2 ，直到 n 小于 MIN_MERGE 为止；
  c.其它情况下，不断进行 n/2 , 最后返回一个数 k，满足 MIN_MERGE/2 <= k <= MIN_MERGE,
  这样结果就能保证 n/k 非常接近但小于一个2的幂。这个数字实际上是一种空间与时间的优化。
    private static int minRunLength(int n) {		// 参数n 为参与排序的数组的长度
        assert n >= 0;
        int r = 0;      // 只要不是2的幂就会置1，是2的幂则 r = 0；其他时候 r = 1
        while (n >= MIN_MERGE) {	// MIN_MERGE = 32 
            r |= (n & 1);		// r = r | (n & 1) 注意这里的位运算，这导致 r 只能是0 或者 1。
            n >>= 1;			// n = n >> 1 右移1位，这里相当于除以2
        }
        return n + r;
    }


14.将指定的升序序列压入等待合并的栈中
    runBase 为升序序列的首个元素的位置
    runLen  为升序序列的长度
    private void pushRun(int runBase, int runLen) {
        this.runBase[stackSize] = runBase;
        this.runLen[stackSize] = runLen;
        stackSize++;
    }
  

15.检查栈中待归并的升序序列，如果他们不满足下列条件就把相邻的两个序列合并，直到他们满足下面的条件
      a. runLen[i - 3] > runLen[i - 2] + runLen[i - 1]
      b. runLen[i - 2] > runLen[i - 1]
  每次添加新序列到栈中的时候都会执行一次这个操作。所以栈中的需要满足的条件需要靠调用这个方法来维护
    private void mergeCollapse() {
        while (stackSize > 1) {			// 栈中待归并的升序序列至少有2条
            int n = stackSize - 2;
            if (n > 0 && runLen[n-1] <= runLen[n] + runLen[n+1]) {	// 第二条，第三条之和大于第一条，
                if (runLen[n - 1] < runLen[n + 1])			// 且第一条小于第三条，合并（标记为A）
                    n--;
                mergeAt(n);
            } else if (runLen[n] <= runLen[n + 1]) {			// 第一条小于第二条，合并（标记为B）
                mergeAt(n);
            } else {							// 其他情况则不做处理（标记为C）
                break; // Invariant is established
            }
        }
    }
  举例说明：
  i.归并序列长度分别为13,8,6，则n = 1，n > 0 成立，13 <= 8 + 6 成立，但13 < 6 不成立，所以A处不符合，
    8 < 6 不成立，所以B处不符合
    最终此归并序列不做处理
  ii.归并序列长度分别为13,6,8，则n = 1，n > 0 成立，13 <= 6 + 8 成立，但13 < 8 不成立，所以A处不符合，
    6 < 8 成立，所以在B处会进行一次序列归并；
    归并后，序列长度分别为13,14，A处自然不符合条件，
    13 < 14 成立，所以在B处会进行一次序列归并；
    归并后，序列长度为27

  所以，总结下来说就是：
  i.假如序列长度逐渐减小，如13,8,6,4...都不会做处理
  ii.假如序列长度最后两位是升序的，形如...6,8，则B处会执行，
  iii.假如序列最后一位的值特别大，大过了倒数第三位，形如...13,2,14，则A处会执行


16.合并栈中所有待合并的序列，最后剩下一个序列。这个方法在整次排序中只执行一次
    private void mergeForceCollapse() {
        while (stackSize > 1) {
            int n = stackSize - 2;
            if (n > 0 && runLen[n - 1] < runLen[n + 1])	// 注意，这里是倒数第三位 小于倒数第一位
                n--;
            mergeAt(n);
        }
    }

17.合并在栈中位于i和i+1的两个相邻的升序序列，序列i必须是栈中倒数第二或倒数第三的序列
  换句话说，i必须符合 i == stackSize - 2 || i == stackSize - 3
    private void mergeAt(int i) {	// 参数 i 为待合并的第一个序列所在的位置
        assert stackSize >= 2;
        assert i >= 0;
        assert i == stackSize - 2 || i == stackSize - 3;

        int base1 = runBase[i];		// 获取第 i 个序列的首个元素位置
        int len1 = runLen[i];		// 获取第 i 个序列的序列长度
        int base2 = runBase[i + 1];
        int len2 = runLen[i + 1];
        assert len1 > 0 && len2 > 0;
        assert base1 + len1 == base2;	// 注意这个判断，由于各个序列是挨个排序的，所以这一条要符合

        // 记录合并后序列的长度，如果i == stackSize - 3 就把最后一个序列的信息往前移一位，因为本次合并不关它的事。
	// i+1对应的序列被合并到i序列中了，所以i+1 数列可以消失了
        runLen[i] = len1 + len2; 	 
        if (i == stackSize - 3) {	// i 为倒数第三个，那么合并的就是倒数第三个序列与倒数第二个序列
            runBase[i + 1] = runBase[i + 2];	// 将倒数第一个序列（也就是最后一个序列）前移，此次合并与它无关
            runLen[i + 1] = runLen[i + 2];
        }
        stackSize--;	// //i+1消失了，所以长度也减下来了

 	// 找出第二个序列的首个元素可以插入到第一个序列的什么位置，因为在此位置之前的序列已经就位了。它们可以被忽略，不参加归并。
        int k = gallopRight((Comparable<Object>) a[base2], a, base1, len1, 0);
        assert k >= 0;
        base1 += k;	// 因为要忽略前半部分元素，所以起点和长度相应的变化
        len1 -= k;
        if (len1 == 0)	// 如果序列2 的首个元素要插入到序列1的后面，那就直接结束了,
            return;	// 因为序列2在数组中的位置本来就在序列1后面,也就是整个范围本来就是有序的

 	// 跟上面相似，看序列1的最后一个元素(a[base1+len1-1])可以插入到序列2的什么位置（相对第二个序列起点的位置，非在数组中的位置），
        // 这个位置后面的元素也是不需要参与归并的。所以len2直接设置到这里，后面的元素直接忽略。
        len2 = gallopLeft((Comparable<Object>) a[base1 + len1 - 1], a,
                base2, len2, len2 - 1);
        assert len2 >= 0;
        if (len2 == 0)
            return;

        // 合并剩下的两个有序序列
        if (len1 <= len2)
            mergeLo(base1, len1, base2, len2);
        else
            mergeHi(base1, len1, base2, len2);
    }
  这里举例进行说明：
  数据为 3,4，5,67,85,12,19,96,97，2,5,33,34,35 ,很明显，序列分为三段[3,4，5,67,85],[12,19,96,97],[2,5,33,34,35],
  在这种情况下，倒数第三条与倒数第二条首先进行处理，即将[3,4，5,67,85],[12,19,96,97]进行合并
  获取第二个序列的首个元素可以插入到第一个序列的位置，换句话说，就是12可以插入到3，4, 5的后面
  获取第一个序列的最后一个元素可以插入到第二个序列的位置，也就是说85可以插入到96,97的前面
  综上所述，[3,4,5]在最前面，[96,97]在最后面，位置时确定的。剩下的是将[67,85]与[12,19]进行排序
  中间部分合并成[12,19,67,85]后，将前后的序列加上，即完成了两个升序序列的合并。


18.在一个序列中，将一个指定的key，从左往右查找它应当插入的位置。如果序列中存在与key相同的值(一个或者多个)，那返回这些值中最左边的位置。
  参数说明：key   准备插入的key
   	    a     参与排序的数组
	    base  序列中第一个元素的位置
	    len   序列的长度
 	    hint  开始查找的位置，有0 <= hint <= len;越接近结果查找越快
  返回一个整数，表示它应该插入的位置
    private static int gallopLeft(Comparable<Object> key, Object[] a, int base, int len, int hint) {
        assert len > 0 && hint >= 0 && hint < len;	// 断言各项参数符合条件

        int lastOfs = 0;				// 上一个偏移位置
        int ofs = 1;					// 当前偏移位置
        if (key.compareTo(a[base + hint]) > 0) {	// key > a[base+hint], hint使用的时候设置为len-1,于是在这里a[base+hint]正好是最后一个元素
	    // 遍历右边，直到 a[base+hint+lastOfs] < key <= a[base+hint+ofs]
            int maxOfs = len - hint;			// 设置maxOfs最大偏移值
            while (ofs < maxOfs && key.compareTo(a[base + hint + ofs]) > 0) {
                lastOfs = ofs;
                ofs = (ofs << 1) + 1;
                if (ofs <= 0)   // int overflow
                    ofs = maxOfs;
            }
            if (ofs > maxOfs)
                ofs = maxOfs;

            lastOfs += hint;			// 因为目前的offset是相对hint的，所以做相对变换
            ofs += hint;
        } else { 	// key <= a[base + hint]
            // 遍历左边，直到[base+hint-ofs] < key <= a[base+hint-lastOfs]
            final int maxOfs = hint + 1;					// maxOfs = len = hint + 1 = 5
            while (ofs < maxOfs && key.compareTo(a[base + hint - ofs]) <= 0) {	// a[base + hint]是最后一位，ofs是偏移位数，偏移从1开始
                lastOfs = ofs;							// 即key与倒数第二位比较大小，然后依次比较
                ofs = (ofs << 1) + 1;
                if (ofs <= 0)   // int overflow
                    ofs = maxOfs;
            }
            if (ofs > maxOfs)
                ofs = maxOfs;

            int tmp = lastOfs;			// 因为目前的offset是相对hint的，所以做相对变换成相对于base的值
            lastOfs = hint - ofs;
            ofs = hint - tmp;
        }
        assert -1 <= lastOfs && lastOfs < ofs && ofs <= len;

        // 现在的情况是 a[base+lastOfs] < key <= a[base+ofs], 所以，key应当在lastOfs的右边，又不超过ofs。在base+lastOfs-1到 base+ofs范围内做一次二叉查找。
        lastOfs++;
        while (lastOfs < ofs) {
            int m = lastOfs + ((ofs - lastOfs) >>> 1);

            if (key.compareTo(a[base + m]) > 0)
                lastOfs = m + 1;  // a[base + m] < key
            else
                ofs = m;          // key <= a[base + m]
        }
        assert lastOfs == ofs;    // so a[base + ofs - 1] < key <= a[base + ofs]
        return ofs;
    }
  举例如下：
  两个升序序列[1,3,5,9] [4,6,10,11,12]
  则 key = 9 ，第一个序列的最后一个值，看它可以插入到第二个序列的哪个位置
     a = [1,3,5,9,4,6,10,11,12],也就是整个序列，所以在这里面需要用到相对位置，处理起来方便一些
     base = 4，即第二个序列中的第一个元素
     len = 5，即第二个序列的长度
     hint = 4，即开始查找的位置，也就是12所处的位置

  首先，key = 9 与 a[base + hint] = 12 比较,结果分成了两支，由于key < a[base + hint] ,我们先看后一支
  然后，分支中 maxOfs = 5，再次判断key = 9 与 a[base + hint - ofs] 的大小，其中ofs是从“1”开始的偏移值
     即最开始 key 与 a[8]比，然后进入这一支后开始与 a[7]比。符合条件，进入while中循环
  再，while循环中，lastOfs = 1，ofs = 3，注意那个左移，相当于乘以2，此时还是满足while循环条件
  再，while循环，key 与 a[5] = 6 比较，此时跳出while循环
  再，对各个偏移值做修正，原来的值是倒数的，现在把这些值转化为正序
     比如lastOfs = 1 表示上一个偏移值为倒数第二个数，ofs = 3 表示当前的偏移值为倒数第4个数
     现在转为lastOfs = 1 ，ofs = 3，此时这些值都是基于base了。
  再，之前划定返回，当前偏移值是通过上一个偏移值乘以2再加1获得的，之后需要在这两个偏移值中间找到具体的偏移位置
  最后，在base+lastOfs-1到 base+ofs范围内做一次二叉查找，直到找到具体位置后返回。
 
