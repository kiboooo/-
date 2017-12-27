## Java——快速排序

> 平均时间复杂度：O（nlogn）
>
> 最快时间复杂度：O（nlogn）
>
> 最慢：O（n^2）
>
> 空间复杂度：O (nlogn)
>
> 冒泡的变种算法；
>
> 思想：
>
> 冒泡+二分+分治+递归

核心算法：

+ 确定哨兵；通常设置数组的第一个为哨兵

+ 设置 low 和 hight 两个游标；

+ hight 不断和哨兵比较，往低处走，直到找到比哨兵小的元素并停下；与 low 处的数据进行交换。

+ low 不断和哨兵比较，往高处走，直到找到比哨兵大的元素并停下；与 hight  处的数据进行交换。

  ​

代码：

```java
/**
 * 快排：是冒泡排序的变种；
 * 运用了冒泡+二分+分治的思想来构建
 * 二分：就是折半搜索思想
 */
public class Quick {

    private static int getMiddle(int[] data ,int low,int high) {

        int temp = data[low];//取数组的第一个为枢纽值;

        //算法执行的条件是满足高低没有相遇
        while (low < high) {

            while (low < high && data[high] >= temp) {
                //从高位开始依次往下，直到找到比枢纽值小的数停下
                high--;
            }

            data[low] = data[high];//把比枢纽值小的数放到低位；

            while (low < high && data[low] < temp) {
                //再从低位开始依次往上，直到找到比枢纽值大的数停下
                low++;
            }

            data[high] = data[low];//把比枢纽值大的数放到高位；
        }

        data[low] = temp;//当高低相遇。就把枢纽值放到相遇的位置

        return low;//返回相遇位置的下标，作为下一次分隔符的标志，是二分的主要体现
    }

    public static void Quicksort(int[] data, int low, int hight) {

        if(low < hight){
            //核心代码，排序交互的具体实现
            int Middle = getMiddle(data, low, hight);

            //分治法的体现，与递归结合；
          
          	//对低字段的表进行递归排序；
            Quicksort(data, 0, Middle - 1);
          
          	//对于高字段的表进行递归排序；
            Quicksort(data, Middle + 1, hight);
        }
    }
}

```

