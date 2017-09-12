1.DualPivotQuicksort是自JDK1.7开始采用的快速排序算法，用于Arrays工具类的sort()方法中，基本类型的排序都是使用此排序算法实现的.
  当然，提一句，TimSort也用于Arrays工具类的sort()方法中，主要用于对象类型的排序。

2.JDK源码流程图



3.基本类型的sort()排序都差不多，所以只使用int类型来举例
    public static void sort(int[] a) {		// a 为待排序的int数组
        sort(a, 0, a.length - 1);
    }

4.步骤3中调用的就是如下方法，此方法也可以直接使用
    public static void sort(int[] a, int left, int right) {	// a 为待排序的int数组，left 为指定数组排序区间最左边的元素
								// right 为指定数组排序区间最右边的元素
        // Use Quicksort on small arrays
        if (right - left < QUICKSORT_THRESHOLD) {		// 判断要排序的数组长度是否小于286，如果是，则使用Quicksort ，处理完后return
            sort(a, left, right, true);				// 这个方法才是DualPivotQuicksort排序的重点，其他的都是外围处理
            return;
        }

        /*
         * Index run[i] is the start of i-th run		// run[i] 意味着第i个有序数列开始的位置，（升序或者降序）
         * (ascending or descending sequence).
         */							// 如果数组长度大于等于286，处理方法类似TimSort，使用归并排序
        int[] run = new int[MAX_RUN_COUNT + 1];			// 寻找run（升序或降序），MAX_RUN_COUNT == 67，即run的个数最多68个		
        int count = 0; run[0] = left;

        // Check if the array is nearly sorted
        for (int k = left; k < right; run[count] = k) {		// 检查数组是不是已经接近有序状态
            if (a[k] < a[k + 1]) { // ascending
                while (++k <= right && a[k - 1] <= a[k]);
            } else if (a[k] > a[k + 1]) { // descending
                while (++k <= right && a[k - 1] >= a[k]);
                for (int lo = run[count] - 1, hi = k; ++lo < --hi; ) {		//如果是降序的，找出k之后，把数列倒置
                    int t = a[lo]; a[lo] = a[hi]; a[hi] = t;
                }
            } else { // equal
                for (int m = MAX_RUN_LENGTH; ++k <= right && a[k - 1] == a[k]; ) {	// 数列中有至少MAX_RUN_LENGTH==33的数据相等的时候，直接使用快排
                    if (--m == 0) {
                        sort(a, left, right, true);					// 也就是DualPivotQuicksort
                        return;
                    }
                }
            }

            /*
             * The array is not highly structured,
             * use Quicksort instead of merge sort.
             */
            if (++count == MAX_RUN_COUNT) {		// 数组中有序数列的个数超过了MAX_RUN_COUNT==67，数组并非高度有序，使用快速排序代替归并排序
                sort(a, left, right, true);		// 也就是DualPivotQuicksort
                return;
            }
        }

        // Check special cases				// 检查特殊情况
        if (run[count] == right++) { // The last run contains one element 最后一个有序数列只有最后一个元素
            run[++count] = right;				// 那给最后一个元素的后面加一个哨兵
        } else if (count == 1) { // The array is already sorted 整个数组中只有一个有序数列，说明数组已经有序啦，不需要排序了
            return;
        }

        /*
         * Create temporary array, which is used for merging.			// 创建合并用的临时数组。
         * Implementation note: variable "right" is increased by 1.		// 注意： 这里变量right被加了1，它在数列最后一个元素位置+1的位置
         */
        int[] b; byte odd = 0;
        for (int n = 1; (n <<= 1) < count; odd ^= 1);

        if (odd == 0) {				// 奇数
            b = a; a = new int[b.length];
            for (int i = left - 1; ++i < right; a[i] = b[i]);	// 注意：i 是先自加1，最后一个没有赋值(哨兵没有赋值)，导致最终获得的数组长度是偶数
        } else {				// 偶数
            b = new int[a.length];		// 全部赋值
        }

        // Merging								// 合并
        for (int last; count > 1; count = last) {				// 最外层循环，直到count为1，也就是栈中待合并的序列只有一个的时候，标志合并成功
            for (int k = (last = 0) + 2; k <= count; k += 2) {			// 遍历数组，合并相邻的两个升序序列
                int hi = run[k], mi = run[k - 1];				// 合并run[k-2] 与 run[k-1]两个序列
                for (int i = run[k - 2], p = i, q = mi; i < hi; ++i) {		
                    if (q >= hi || p < mi && a[p] <= a[q]) {
                        b[i] = a[p++];
                    } else {
                        b[i] = a[q++];
                    }
                }
                run[++last] = hi;						// 这里把合并之后的数列往前移动
            }
            if ((count & 1) != 0) {						// 如果栈的长度为奇数，那么把最后落单的有序数列copy过对面
                for (int i = right, lo = run[count - 1]; --i >= lo;
                    b[i] = a[i]
                );
                run[++last] = right;
            }
            int[] t = a; a = b; b = t;						// 临时数组，与原始数组对调，保持a做原始数组，b 做目标数组
        }
    }

  说明：
  a.上述代码时双轴快排的外围处理，一进来就判断数组长度是否小于286，符合条件才进入双轴快排核心处理方法sort(a, left, right, true);中
    数组长度大于等于286的，数据量比较大，快排不合适，代码中使用TimSort某些思想，归并排序进行处理
  b.数据量较多，首先检查数据的有序状态，降序转为升序，获得run[]数组，run[i] 表示第i个有序数列开始的位置，
    这样，整个待排序序列，分割成为一个个升序序列，升序序列的首位序列数保存在run[]数组中。
  c.在获取升序序列的过程中，如果发现有大量的数据(MAX_RUN_LENGTH==33)是连续相等的，则直接使用双轴快排，效率比归并排序高
  d.另外，数组中有序数列的个数超过了MAX_RUN_COUNT==67，表示数据升降很频繁，数组高度无序，此时直接使用双轴快排，效率比归并排序高
    例如MAX_RUN_COUNT==2，则表示整个数据可以分为两个升序序列，使用TimSort的归并/插入思想处理起来很容易。
  e.除了 c 中‘数据连续相等’与 d 中‘数据高度无序’两种特殊状况可以转为双轴快排外，其他的都是进行归并排序，其核心就是“合并run栈”
  f.最后进行run合并


5.双轴快排的实现重点（源代码太坑爹，不贴了）
  a.当数组长度小于47时，使用插入排序
    参数a为需要排序的数组，left代表需要排序的数组区间中最左边元素的索引，right代表区间中最右边元素的索引，leftmost代表该区间是否是数组中最左边的区间。举个例子
    数组：[2, 4, 8, 5, 6, 3, 0, -3, 9]可以分成三个区间（2, 4, 8）{5, 6}<3, 0, -3, 9>
    对于（）区间，left=0, right=2, leftmost=true
    对于 {}区间， left=3, right=4, leftmost=false,同理可得<>区间的相应参数
    当区间长度小于47时，该方法会采用插入排序；否则采用快速排序。
  b.当leftmost为true时，它会采用传统的插入排序（traditional insertion sort），代码也较简单，其过程类似打牌时抓牌插牌。
  c.当leftmost为false时，它采用一种新型的插入排序（pair insertion sort），改进之处在于每次遍历前面已排好序的数组需要 插入两个元素，而传统插入排序在遍历过程中
    只需要为一个元素找到合适的位置插入。
    对于插入排序来讲，其关键在于为待插入元素找到合适的插入位置，为了找到这个位置，需要遍历之前已经排好序的子数组，所以对于插入排序来讲，整个排序过程中其遍历的
    元素个数决定了它的性能。很显然，每次遍历插入两个元素可以减少排序过程中遍历的元素个数 。
  d.对于快速排序来讲，其每一次递归所做的是使需要排序的子区间变得更加有序 ，而不是绝对有序；所以对于快速排序来说，其性能决定于每次递归操作使待排序子区间变得有序的程度，
    另一个决定因素当然就是递归次数。
  e.pivot的选取方式是将数组分成近视等长的七段，而这七段其实是被5个元素分开的，将这5个元素从小到大排序，取出第2个和第4个，分别作为pivot1和pivot2
    注意：当这个5元素都互不相等时，才采用双枢轴快速排序！
  f.Pivot选取完之后，分别从左右两端向中间遍历，左边遍历停止的条件是遇到一个大于等于pivot1的值，并把那个位置标记为less;右边遍历的停止条件是遇到一个小于等于pivot2的值，
    并把那个位置标记为great
  g.然后从less位置向后遍历，遍历的位置用k表示，会遇到以下几种情况：
	i. k位置的值比pivot1小，那就交换k位置和less位置的值，并是less的值加1;这样就使得less位置左边的值都小于pivot1,而less位置和k位置之间的值大于等于pivot1
	ii. k位置的值大于pivot2,那就从great位置向左遍历，遍历停止条件是遇到一个小于等于pivot2的值，假如这个值小于pivot1,就把这个值写到less位置，把less位置的值写道k位置，
	    把k位置的值写道great位置，最后less++,great--;加入这个值大于等于pivot1,就交换k位置和great位置，之后great--。
  h. 完成上述过程之后，带排序的子区间就被分成了三段（pivot2），最后分别对这三段采用递归就行了


