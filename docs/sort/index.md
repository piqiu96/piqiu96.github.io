# 排序算法


# 排序算法

## 一、算法分类
- 插入排序
    - 直接插入排序
    - 希尔排序
- 交换排序
    - 冒泡排序
    - 快速排序
- 选择排序
    - 简单选择排序
    - 堆排序
- 归并排序
    - 二路归并排序（递归实现）

## 二、算法实现
### 2.1、插入排序
#### 2.1.1、直接插入排序
- 数组：12,15,9,20,6,31,24,2,17,35
    - 有序区{12,15}
    - 无序区{9,20,6,31,24,2,17,35}
- 思路：
    1. 将无序区的第一个元素，称为候选元素。
    2. 将候选元素与有序区元素比较，选择适当位置插入。
```java
public class InsertionSort {
     public static void sort(int [] array){
           for(int i=1;i<array.length;i++){
                for(int j=i;j>=1 && array[j]<array[j-1];j--){
                     int temp =array[j];
                     array[j]=array[j-1];
                     array[j-1]=temp;
                }
           }
     }

     public static void main(String [] args){
           int [] array={12,15,9,20,6,31,24,2,17,35};
           InsertionSort.sort(array);
           System.out.println(Arrays.toString(array));
     }
}
```

#### 2.1.2、希尔排序
- 产生：直接插入排序在基本正序的时候，排序效率高，所以把数组拆分很多个小数组，进行简单排序
- 思路：
    1. 将数组分组，间距从[d=n/2-1]，变换d=d/2
    2. 分别将不同分区进行直接插入排序。
    3. 当d=1，合并所有的分区进行直接插入排序
```java
public class ShellSort {
     //采取分区
     public static void sort(int [] array){
           for(int d=array.length/2;d>=1;d=d/2){
                insertSort(array,d);
           }
     }

     //插入排序
     public static void insertSort(int [] array,int d){
           for(int i=d;i<array.length;i++){
                for(int j=d;j>=d && array[j]<array[j-d];j=j+d){
                     int temp=array[j];
                     array[j]=array[j-d];
                     array[j-d]=temp;
                }
           }
     }

     public static void main(String [] args){
           int [] array={12,15,9,20,6,31,24,2,17,35,5};
           InsertionSort.sort(array);
           System.out.println(Arrays.toString(array));
     }
}
```


### 2.2、交换排序
#### 2.2.1、冒泡排序
- 数组：12,15,9,20,6,31,24,2,17,35
    - 无序区：{12,15,9,20,6,31,24,2,17}
    - 有序区：{35}
- 问题：
    1. 如何确定需要冒泡排序范围，保证有序记录不参加下一趟的排序？
    2. 如何判断冒泡排序结束？
- 思路：
    1. 从数组头开始，候选元素跟后继元素比较，通过交换选出最大元素，直到无序区选出最大元素放入有序区的。
    2. 记录有序区的头索引，将不参加下一趟排序。
    3. 当没有元素交换，冒泡排序结束。
```java
public class BubbleSort {
     public static void sort(int [] array){
           int exchange=array.length;
           //判断无序列是否有交换，冒泡排序结束标志
           while(exchange!=0){
                //无序区的边界
                int bound=exchange;
                exchange=0;
                for(int i=1;i<bound;i++){
                     if(array[i]<array[i-1]){
                           int temp=array[i];
                           array[i]=array[i-1];
                           array[i-1]=temp;
                           exchange=i;
                     }
                }
           }
     }

     public static void main(String [] args){
           int [] array={12,15,9,20,6,31,24,2,17,35,5};
           BubbleSort.sort(array);
           System.out.println(Arrays.toString(array));
     }
}
```
#### 2.2.2、快速排序
- 数组：12,15,9,20,6,31,24,2,17,35
- 思路：
    1. 一次快速划分。选择轴值，最终轴值左侧小于轴值，轴值右侧大于轴值。
    2. 例如：最初选择轴值为12，通过一次快速划分，{2,6,9,12,15,20,17,35,31}
    3. 对轴值左侧、轴值右侧。 递归调用一次快速划分。
```java
public class QuickSort {
     public static int partition(int [] array,int low,int high){
           int i=low;
           int j=high;
           int temp;
           //当i==j,结束一次快速划分
           while(i<j){
                //向右侧扫描
                while(i<j && array[i]<array[j])
                     j--;
                //说明轴值大于array[j],交换轴值变成array[j]
                if(i<j){
                     temp=array[i];
                     array[i]=array[j];
                     array[j]=temp;
                     i++;
                }

                //向左侧扫描
                while(i<j && array[i]<array[j])
                     i++;

                if(i<j){
                     temp=array[i];
                     array[i]=array[j];
                     array[j]=temp;
                     j--;
                }
           }
           return i;
     }

     public static void sort(int [] array,int low,int high){
           if(low<high){
                int mid =partition(array, low, high);
                sort(array, low, mid-1);
                sort(array, mid+1, high);
           }
     }

     public static void main(String [] args){
           int [] array={12,15,9,20,6,31,24,2,17,35,5};
           QuickSort.sort(array, 0, array.length-1);
           System.out.println(Arrays.toString(array));
     }

}
```

### 2.3、选择排序
#### 2.3.1、简单选择排序
- 数组：12,15,9,20,6,31,24,2,17,35
    - 有序区：{}
    - 无序区：{12,15,9,20,6,31,24,2,17,35 }
- 思路
    1. 把无序区的最小元素，采用选择方式放入有序区末尾
```java
public class SelectSort {
     public static void sort(int [] array){
           int index;
           for(int i=0; i<array.length; i++){
                index=i;
                for(int j=i; j<array.length; j++){
                     if(array[j] < array[index])
                           index = j;
                }
                if(index != i){
                     int temp = array[i];
                     array[i] = array[index];
                     array[index] = temp;
                }
           }
     }

     public static void main(String [] args){
           int [] array={12,15,9,20,6,31,24,2,17,35,5};
           SelectSort.sort(array);
           System.out.println(Arrays.toString(array));
     }
}
```

#### 2.3.2、堆排序
- 数组：12,15,9,20,6,31,24,2,17,35
- 思路：
    1. 把候选元素构建成大根堆。
    2. 从数组最后一个元素开始遍历，把大根堆的最大元素放入有序区。
    3. 去掉有序区元素，重新调整大根堆。
```java
public class HeapSort {
     //根据候选元素，构建成大根堆
     public static void adjustHeap(int [] array,int i,int len){
           int j=0;
           int temp=array[i];
           for(j=i*2;j<len;j=j*2){
                if( j<len && array[j]<array[j+1])
                     j++;

                if(temp>=array[j])
                     break;
                array[i]=array[j];
                i=j;
           }
           array[i]=temp;
     }

     public static void sort(int [] array){
           int i;
           //选非叶子节点，构造大根堆
           for(i=array.length/2-1;i>=0;i--){
                adjustHeap(array, i, array.length-1);
           }
           //不断选大根堆最大元素放入有序区
           for(i=array.length-1;i>=0;i--){
                int temp=array[0];
                array[0]=array[i];
                array[i]=temp;
                adjustHeap(array, 0, i-1);
           }
     }

     public static void main(String [] args){
           int [] array={12,15,9,20,6,31,24,2,17,35,5};
           HeapSort.sort(array);
           System.out.println(Arrays.toString(array));
     }

}
```

### 2.4、归并排序
#### 2.4.1、二路归并排序
- 数组：12,15,9,20,6,31,24,2,17,35
- 思路：
    1. 利用一次归并算法将两个分区元素，归并到一个数组里。
    2. 先把数组划分成长度为1的序列区间，利用递归的方式，归并到一个序列里。
```java
public class MergeSort {
     public static void merge(int [] array,int low,int mid,int high){
           int [] tempArray=new int[high-low+1];
           int i=low;
           int j=mid+1;
           int k=0;

           while(i<=mid && j<=high){    
                if(array[i]<=array[j]){
                     tempArray[k++]=array[i++];
                }else{
                     tempArray[k++]=array[j++];
                }
           }

           //把第一个分片序列，处理余下数据
           while(i<=mid){
                tempArray[k++]=array[i++];
           }

           //把第二个分片序列，处理余下数据
           while(j<=high){
                tempArray[k++]=array[j++];
           }

           //把临时数组的数据加入到原数组
           for(int k2=0;k2<tempArray.length;k2++){
                array[low+k2]=tempArray[k2];
           }
     }

     public static void sort(int [] array,int low, int high){
           int mid=(low+high)/2;
           if(low<high){
                sort(array, low, mid);
                sort(array, mid+1, high);
                merge(array, low, mid, high);
           }
     }

     public static void main(String [] args){
           int [] array={12,15,9,20,6,31,24,2,17,35,5};
           MergeSort.sort(array,0,array.length-1);
           System.out.println(Arrays.toString(array));
     }

}
```


