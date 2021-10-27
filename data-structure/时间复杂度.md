# 时间复杂度(Big O notation)

## 常见时间复杂度

+ O(1): Constant Complexity 常数复杂度
+ O(log n): Logarithmic Complexity 对数复杂度
+ O(n): Linear Complexity 线性时间复杂度
+ O(n^2) N square Complexity 平方
+ O(n^3) N square Complexity 立方
+ O(2^n) Exponential Growth 指数
+ O(n!) Factorial 阶乘
+ 其中不考虑系数,只考虑最高复杂度的运算

## 时间复杂度曲线

![image-20211027114439956](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211027114439956.png)

## 递归主定理 (Master Theorem)

![image-20211027141710501](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211027141710501.png)

+ 二分查找, O(logn)
+ 二叉树遍历, O(n) (tips: 因为二叉树每个数据只访问一次,所以可以理解为 O(n))
+ 排好序的矩阵进行二分查找, O(n)
+ 归并排序, O(nlogn)

## References

+ [如何理解算法时间复杂度的表示法](http://www.zhihu.com/question/21387264)

+ [Master theorem](http://en.wikipedia.org/wiki/Master_theorem_(analysis_of_algorithms))
+ [主定理](http://zh.wikipedia.org/wiki/主定理)

