# 数组,链表,跳表

## 1. 数组

### 1.1 硬件底层实现

![image-20211028101657527](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211028101657527.png)

### 1.2 特点

+ 申请时在内存中开辟一段连续的地址

+ 随地访问时间复杂度 O(1)

+ 插入/删除复杂度 O(n)

  ![image-20211028115427481](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211028115427481.png)

## 2.链表

### 2.1 实现方式

![image-20211028110553834](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211028110553834.png)

```java
class LinkedList {
  Node head;
  
  class Node {
    int data;
    Node next;
    Node(int d) {data = d;}
  }
}
```



### 2.2 特点

+ 每个元素使用一个 class/struct 实现, 一般名为 node

+ next 指针指向下一个元素

+ 添加/删除复杂度 O(1)

+ 随机访问复杂度O(n)

  ![image-20211028115451798](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211028115451798.png)

## 3. 跳表

### 3.1 设计目的/思路

+ 弥补链表查询的不足
+ 对链表进行升维, 使用空间换时间

### 3.2 特点

+ 查询时间复杂度Log2(n) - 1 = O(logn)

+ 通过添加 log2n个索引,加速遍历

  ![image-20211028150623753](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211028150623753.png)

  ![image-20211028154343658](http://keepcoding.oss-cn-beijing.aliyuncs.com/uPic/image-20211028154343658.png)

+ 在实际使用中, 跳表会随着元素的增加和删除, 增加和减少索引,增加删除退化为 O(logn)
+ 空间复杂度为 O(n)

## Reference

+ [Java 源码分析（ArrayList）](http://developer.classpath.org/doc/java/util/ArrayList-source.html)
+ [Linked List 的标准实现代码](http://www.geeksforgeeks.org/implementing-a-linked-list-in-java-using-class/)
+ [Linked List 示例代码](http://www.cs.cmu.edu/~adamchik/15-121/lectures/Linked Lists/code/LinkedList.java)
+ [Java 源码分析（LinkedList）](http://developer.classpath.org/doc/java/util/LinkedList-source.html)
+ [ LRU 缓存机制](http://leetcode-cn.com/problems/lru-cache)
+ [为啥 Redis 使用跳表（Skip List）而不是使用 Red-Black？](http://www.zhihu.com/question/20202931)

