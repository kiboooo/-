 堆排序
---
> 堆的概念：
> 是一棵顺序存储的完全二叉树。
> 实现逻辑：都是用数组作为储存单位，利用数组的下标作为树节点的描述；
> 时间复杂度为(最快和最坏都是)：O（nlogn）
>
> 空间负载度：O（1）
>
> 一种不稳定的排序算法
> 本质：利用堆的数据结构来达到调整出有序数列的目的

大顶堆：每个节点都大于等于他的所有孩子节点，根节点是最大值；
> 公式：Ri >= R2i+1 且 Ri >= R2i+2 (大根堆)

小顶堆：每个节点都小于等于它的所有孩子节点，根节点是最小值；
> 公式： Ri <= R2i+1 且 Ri <= R2i+2 (小根堆)

堆排序的过程：
+ 建立堆（建堆的渗透函数）
+ 堆顶与堆的最后一个元素交换位置 （反复调用的渗透函数实现排序的函数)


#### 注意：
大顶堆，设当前元素在数组中以R[i]表示，那么，
+ (1) 它的左孩子结点是：R[2*i+1];
+ (2) 它的右孩子结点是：R[2*i+2];
+ (3) 它的父结点是：R[(i-1)/2];
+ (4) R[i] <= R[2*i+1] 且 R[i] <= R[2i+2]。

如何判断最后一个非叶子节点？
设二叉树的节点数为n,最后一个非叶子节点为n/2向下取整。


#### 应用场景：
当想得到一个序列中第k个最小的元素之前的部分排序序列，最好采用堆排序。

#### 代码：
```java
import java.util.Arrays;

public class HeapSort {

    //进行Heap排序
    public static void ToSort(int[] data){
        int Length = data.length;
        //循环建堆，输出最后一个节点，达到排序的目的
        for (int i = 0; i < Length - 1; i++) {
            //按条件调整堆：
            HeadAdjust(data, Length - i - 1);
            System.out.println("第"+(i+1)+"次调整："+Arrays.toString(data));
            //交换根节点和最后一个节点
            Swap(data, 0, Length - i - 1);
            //输出最后一个节点
            System.out.println("第"+(i+1)+"次排序："+Arrays.toString(data));
        }
    }

    //调整堆：按照具体排序要求调整为 小顶堆 或 大顶堆,以下调整以大顶堆为例
    private static void HeadAdjust(int[] data,int lastIndex)  {
        //由于向下取整的特性，lastIndex - 1 后可以优化一些情况下的循环次数；
        //从lastIndex处节点（最后一个节点）的父节点开始
        for (int i = (lastIndex-1) / 2; i >= 0; i--) {

           int k = i ; //记录当前比对的位置

            while (k * 2 + 1 <= lastIndex) {//判断是否存在孩子，由于是完全二叉树，不存在左孩子为空，右孩子存在的情况；

                int biggerForChilden = 2 * k + 1;//暂时存放存在的左孩子

                //判断右孩子存不存在；
                if (biggerForChilden + 1 <= lastIndex) {
                    //比较两个孩子那个比较大
                    if (data[biggerForChilden] < data[biggerForChilden + 1]) {

                        //取两者的最大值
                        biggerForChilden += 1;
                    }
                }

                //孩子中最大者 与父亲比较，
                if (data[k] < data[biggerForChilden]) {

                    //比父亲大则交换
                    Swap(data, k, biggerForChilden);

                    //把交换后的下标赋值给K，进行下一次循环，保证该节点下的子树满足条件
                    k = biggerForChilden;
                }else
                    break;
            }
        }
    }

    //交换方法
    private static void Swap(int[] data, int i, int j) {
        int temp = data[i];
        data[i] = data[j];
        data[j] = temp;
    }
}

```





