---
title: 面经总结
date: 2019-05-05 21:03:47
categories: "Blog"
comments: true
thumbnail: /gallery/interview/pic.jpg
tags:
- 算法
- React
- Blog
---

<!-- no node -->

<!-- more -->

四月，时常在思考现代年轻人职业发展规划的问题，有很多条路但是却又很难抉择，因为未来的不确定性导致。那么最终也得出了相应的结论，选择适合自己的路，不要让自己的青春有所遗憾，当然一生得开开心心的这才是最重要的，不要背弃自己的初衷，做自己喜欢的。

## React 源码阅读计划

在阅读完《 React 源码阅读计划》之 `0.3-stable` 后，我准备开始启动《 React 源码阅读计划》之 `16.8.6` 。这里会有很多新知识，截止目前阅读到 [`scheduler`](https://github.com/facebook/react/tree/v16.8.6/packages/scheduler) 部分，对双向链表的学习，理解调度是目前正在学习的任务。

## Ant Design 之奇淫技巧

我司后台项目也在有条不紊的快速开发中，在大型的 React 后台项目中利用 Redux + Immutable 带来的便利可是相当的明显。他让我在快速开发时只需关注业务层和视图层即可，并不需要我考虑如何连接和传递，不仅提高了我的开发效率也提升了我代码的容错率。可见是非常高效强大的工具啊。

当然啦，我司后台项目用到了 Ant Design ，在使用过程中，也遇到了写奇奇怪怪的问题，也用些奇淫技巧进行了解决，列举一个：

关于 `DatePicker` ， `year` 模式下无法触发 `onChange` 的情况，也可看相关 [issues #14017](https://github.com/ant-design/ant-design/issues/14017#issuecomment-481544170) ，已在此进行回复。

```javascript
class Demo extends Component {
  hankPicker = null
  hankYearChange = v => {
    // 更新 v
    // 强制关闭补丁
    this.hankPicker.picker.setState({ open: false });
  }
  render() {
    return (
      <DatePicker
        ref={ref => (this.hankPicker = ref)}
        onPanelChange={this.hankYearChange}
        mode="year"
        format="YYYY" />
    )
  }
}
```

## 省市区业务

除了 Ant Design 也有一些关于省市区的业务代码，利用基础数据：

```typescript
interface IAreaData {
  label: String
  value: String
  children: IAreaData[]
}
```

来生成一个通过省市区当前 `value` 来返回当前 `value` 对应的省市区关系 `value` 级联数组映射表，那么函数如下：

```javascript
/**
 * 生成中国省市区（县）三级联动末尾值对应数组表
 * @use generateChinaAreaCascadeValueMap(areaData);
 * @param areaData 中国省市区（县）数据
 * @param result 默认结果
 * @param parents 默认父节点
 * @return result
 */
export const generateChinaAreaCascadeValueMap = (areaData = [], result = {}, parents = []) => {
  for (const i of areaData) {
    result[i.value] = [...parents, i.value];
    if (i.children && i.children.length > 0) {
      generateChinaAreaCascadeValueMap(i.children, result, [...parents, i.value]);
    }
  }
  return result;
};
```

## 算法学习

### 冒泡排序（Bubble Sort）

冒泡排序是一种简单的排序算法。它重复地走访过要排序的数列，一次比较两个元素，如果它们的顺序错误就把它们交换过来。走访数列的工作是重复地进行直到没有再需要交换，也就是说该数列已经排序完成。这个算法的名字由来是因为越小的元素会经由交换慢慢“浮”到数列的顶端。

```javascript
function bubbleSort(arr) {
    var len = arr.length;
    for (var i = 0; i < len - 1; i++) {
        for (var j = 0; j < len - 1 - i; j++) {
            if (arr[j] > arr[j+1]) {        // 相邻元素两两对比
                var temp = arr[j+1];        // 元素交换
                arr[j+1] = arr[j];
                arr[j] = temp;
            }
        }
    }
    return arr;
}
```

### 选择排序（Selection Sort）

选择排序(Selection-sort)是一种简单直观的排序算法。它的工作原理：首先在未排序序列中找到最小（大）元素，存放到排序序列的起始位置，然后，再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。以此类推，直到所有元素均排序完毕。

```javascript
function selectionSort(arr) {
    var len = arr.length;
    var minIndex, temp;
    for (var i = 0; i < len - 1; i++) {
        minIndex = i;
        for (var j = i + 1; j < len; j++) {
            if (arr[j] < arr[minIndex]) {     // 寻找最小的数
                minIndex = j;                 // 将最小数的索引保存
            }
        }
        temp = arr[i];
        arr[i] = arr[minIndex];
        arr[minIndex] = temp;
    }
    return arr;
}
```

### 插入排序（Insertion Sort）

插入排序（Insertion-Sort）的算法描述是一种简单直观的排序算法。它的工作原理是通过构建有序序列，对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。

```javascript
function insertionSort(arr) {
    var len = arr.length;
    var preIndex, current;
    for (var i = 1; i < len; i++) {
        preIndex = i - 1;
        current = arr[i];
        while (preIndex >= 0 && arr[preIndex] > current) {
            arr[preIndex + 1] = arr[preIndex];
            preIndex--;
        }
        arr[preIndex + 1] = current;
    }
    return arr;
}
```

### 快速排序（Quick Sort）

快速排序的基本思想：通过一趟排序将待排记录分隔成独立的两部分，其中一部分记录的关键字均比另一部分的关键字小，则可分别对这两部分记录继续进行排序，以达到整个序列有序。

```javascript
function quickSort(arr, left, right) {
    var len = arr.length,
        partitionIndex,
        left = typeof left != 'number' ? 0 : left,
        right = typeof right != 'number' ? len - 1 : right;

    if (left < right) {
        partitionIndex = partition(arr, left, right);
        quickSort(arr, left, partitionIndex-1);
        quickSort(arr, partitionIndex+1, right);
    }
    return arr;
}

function partition(arr, left ,right) {     // 分区操作
    var pivot = left,                      // 设定基准值（pivot）
        index = pivot + 1;
    for (var i = index; i <= right; i++) {
        if (arr[i] < arr[pivot]) {
            swap(arr, i, index);
            index++;
        }
    }
    swap(arr, pivot, index - 1);
    return index-1;
}

function swap(arr, i, j) {
    var temp = arr[i];
    arr[i] = arr[j];
    arr[j] = temp;
}
```

## 从面试中的自我学习

在这次面试过程中，我可以得知我的知识面广度已经有所触及到，但是深度方面还需加强。

- React 源码学习及周边工具学习
- Vue 插槽（ `slot` ）学习
- 迎接 Vue 3.0
- 轻量级框架 [Zepto.js](http://www.zeptojs.cn/) 学习（技术选型：jQuery、Zepto.js）
- 工程化工具 [Gulp.js](https://gulpjs.com/) 学习（技术选型：Webpack、Gulp，他们的区别）
- 图形化学习（ CSS3、Canvas、SVG... ）

那么对于面试公司的了解，我也简单介绍并对其组织架构进行学习，公司名字我就不在这里说了。贵司分为： H5 组、后台组、基建组：

- H5 组主要负责对 C 的页面产出，采用 Vue / jQuery 技术栈。
- 后台组主要负责对内或是（对 B）的页面产出，采用 React 技术栈。
- 基建组主要负责贵司内部底层搭建，采用 Node 技术栈。

通过这里也可以衍生出我未来可能要面对的技术栈线路，也会影响到我的职业规划，当然不同公司的分型也是不同的，我希望了解的更多，学习的更多。对于我未来的职业规划，我会选择慢慢延伸知识广度，大力加深知识深度为主。

## 参考学习

- [React Scheduler 源码详解（1）](https://juejin.im/post/5c32c0c86fb9a049b7808665)
- [React Scheduler 源码详解（2）](https://juejin.im/post/5c61197ff265da2d9e173337)
- [十大经典排序算法（动图演示）](https://www.cnblogs.com/onepixel/articles/7674659.html)
