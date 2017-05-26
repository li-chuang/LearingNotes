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
