1.两数之和
----------------------
把计算值作为key,把index作为value

121.买卖股票的最佳时机
-----------------------
遍历，同时记录最低股票价格，和当天最大利润。然后取最大利润值。如果一点利润都没有，返回0

146.LRU缓存
------------------------
用linkedHashMap实现即可

189.轮转数组
-----------------------
+k取模，然后复制数组

45.跳跃游戏II
--------------------------
不断探寻距离终点最远的位置，找到后把终点改为该位置。终点从右，寻找点从左开始

283.移动零
--
双指针法，把所有的非零数都拷贝到最左边，最后把指针j右边的数全部置为0

54.螺旋矩阵
--------
螺旋取值，上下左右。

23.合并k个升序链表
----
使用优先队列实现

62.不同路径
------------
动态规划

394.字符串解码
-----
栈 获取连续数字  字符串翻转

104.二叉树的最大深度
-----
深度优先搜索 max(left,right)+1

102.二叉树的层序遍历
-----
广度优先搜索

101.对称二叉树
------------
dfs,left.left == right.right

72.编辑距离
---------------
动态规划，用空字符辅助两个字符串，形成坐标系

160.相交链表
---------
命里有时终须有，形同陌路又如何？

33.搜索旋转排序数组
-----------
二分查找，有序判断

198.打家劫舍
--------
劫舍有两种策略，你知道咩？

142.环形链表II
---------
这TM不是数学题吗

141.环形链表
-------------
快慢指针，有手你就来

226.翻转二叉树
-------------
做对了我能进google吗？