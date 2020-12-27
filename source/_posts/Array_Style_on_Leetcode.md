---
title: 刷题总结-数组类型
date: 2020-08-11 00:20:00
tags:
---
<!-- toc -->

--------最近发现刷题还是分类刷着有效率，也便于总结。找到一个[LeetCode的题目分类清单](https://blog.csdn.net/CowBoySoBusy/article/details/82559651)。貌似只涉及前400题，题量很适合我，两三周一个类型刷完，照着清单开始练习加总结。

## 1 数组类型题目分类

因为题量不多，目前涉及的有几类：

**一维数组**

- 正常遍历
- 两个指针遍历
- 查找
- 排序
- 字典序
- 移动顺序

**二维数组**

- 合并范围
- 变形
- 二维查找
- 查找后替换
- 引入中间状态替换
- 特殊顺序遍历
- 杨辉三角

**涉及到的主要概念**

- 快排
- 桶排序
- 二分查找
- 字典序
- 杨辉三角
- 图形遍历

## 2 概念掌握

### 2.1 快排

**时间复杂度**: nlogn **空间复杂度**: logn

基于分治，选出一个数，将大于这个数的放一边，小于这个数的放另一边，再对两个区域分别重复上述步骤，直到区间只有一个数。

算法核心分为两部分：

- 将数组按照基准值分区
- 对照分区重复上述步骤

伪代码如下：

```java
int dispatchByKey(int[] array, int from, int to) {
  int keyValue = int[from];
  int i=from;
  int j=to;
  while(i<j) {
    while(i<j && array[j] >= x)
      j--
    if (i<j) array[i++] = array[j]
    while (i<j && array[i] < x)
      i++
    if (i<j) array[j--] = array[i]
  }
  array[i] = keyValue;
}

void quickSort(int[] array, int from, int to) {
  if (from < to) {
    int i = dispatchByKey(array, from, to);
    quickSort(array, from, i-1);
    quickSort(array, i+1, to);
  }
}
```

### 2.2 桶排序

**时间复杂度**: N*(logN - logM) + N **空间复杂度** N+M

还是基于分治，设定桶（分区数量），根据元素上下界，产生每个桶（区间）边界，将元素放到对应桶中，再分别排序。

桶内的排序算法，可以自选。

伪代码如下：

```java
List<Integer> bucketSort(int[] array, int from, int to, int bucketCount) {
  int min = array.min;
  int max = array.max;
  int bucketSize = (max-min)/bucketCount;
  List<List<Integer>> bucket = new ArrayList<>(bucketCount);
  for (int i=0;i<bucketCount;i++) {
    bucket.add(new LinkedList<>());
  }
  for (int i=0;i<array.length;i++) {
    bucket.get(array[i]/bucketCount - 1).add(array[i])
  }
  for (int i=0;i<bucketCount;i++) {
    quickSort(bucket.get(i));
  }
  return combine(bucket);
}
```

### 2.3 二分查找

**时间复杂度**: log2n

针对有序列表进行查找，每次从中间挑选一个数取值，如果目标大于它，则在大于它的区间重复上述操作，否则在小小于它的区间进行上述操作。

伪代码如下：

```java
int binarySearch(int[] sorted, int from, int to, int target) {
  if (sorted[from] == target){
    return from;
  }
  if (sorted[to] == target) {
    return to;
  }
  int nextCenter = (from + to)/2;
  if (sorted[nextCenter] > target) {
    return binarySearch(sorted, from, nextCenter, target);
  } else if (sorted[nextCenter] < target) {
    return binarySearch(sorted, nextCenter, to, target);
  } else {
    return nextCenter;
  }
}
```

### 2.4 字典序

一段数字存在一个全排列，字典序就是定义全排列顺序的一种约定。通常这里需要求指定排列的下一个排列，或将某个排列按字典序进行排序。

**举例**：1234，顺序从小到大为1234, 1243, 1324, 1342, 1423, 1432, 2143, ...., 4321

**定义**：最小为从左至右依次升高的排列，最大为从左至右依次降低的排列

由上述定义可知，算法的几个核心要素：

**求下一个排列**

- 从右向左找到第一个左邻小于右的数，找不到说明排列到最后了
- 找到后，再从右到左找到第一个大于这个数的数，交换两数
- 将交换到前面的这个数的右边，从小到大排序

伪代码如下：

```java
void nextSeq(int[] array) {
  int firstLeftLessThanRightIndex = -1;
  for (int i=array.length-1;i>0;i--) {
    if (array[i-1] < array[i]) {
      firstLeftLessThanRightIndex = i-1;
      break;
    }
  }
  if (firstLeftLessThanRightIndex == -1) return;
  int firstGreaterIndex = -1;
  for (int i=array.length-1;i>0;i--) {
    if (array[i] > array[firstLeftLessThanRightIndex]) {
      firstGreaterIndex = i;
      break;
    }
  }
  if (firstGreaterIndex == -1) return;
  
  // swap
  int temp = array[firstGreaterIndex];
  array[firstGreaterIndex] = array[firstLeftLessThanRightIndex];
  array[firstLeftLessThanRightIndex] = temp;
  
  // sort
  quickSort(array, firstLeftLessThanRightIndex+1, array.length);
}
```

将全排列按字典序排序则只需要其中部分判断逻辑即可

### 2.5 杨辉三角

这个似乎没啥说的，了解下杨辉三角的定义即可。

### 2.6 图形化的遍历

有的题目比如要求按照螺旋状遍历二维数组，那么其实就照着这个样子在图上遍历即可，注意边界条件

## 3 总结

数组类型的题目涉及的概念都很基础，理解上也没什么难度。主要记住几个重点，当然也不仅是数组类型题目的重点：

- 寻求分治手段降低复杂度
- 寻求特殊规律降低复杂度
- 注意所有可能的边界条件并针对性处理
