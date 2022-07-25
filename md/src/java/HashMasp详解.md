**HashMasp详解**

``1. 关于初始化容量 new HashMap(17)会发生什么？``

<img src="C:\Users\tracy\AppData\Roaming\Typora\typora-user-images\image-20220704180538320.png" alt="image-20220704180538320" style="zoom: 67%;" />

​	这里的tableSizeFor主要是获得一个最小的n，使得17 > 2^n

​    因为获取这样的size，可以使得在进行hash计算时，使得元素存放的下标更为散列。



``2. 负载因子``

​	最大阈值,决定元素存放多少后开始扩容，从而减少hash碰撞

​	（1） 扩容产生的问题1：原来的元素怎么办？

​			JDK7中需要重新进行hash计算，而JDK8已经进行优化，不再进行重新计算