---
title: 剑指offer 
date: 2019-08-09
categories: 
- leetcode 
---
> 题目列表，当做一个速查列表好了
# 链表-8道

[剑指 Offer 06. 从尾到头打印链表](https://leetcode-cn.com/problems/cong-wei-dao-tou-da-yin-lian-biao-lcof/)

````go
//递归打印
package reverse_print

/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */

type ListNode struct {
	Val  int
	Next *ListNode
}

func ReversePrint(head *ListNode) []int {
	var res []int
	ReversePrintRecv(head, &res)
	return res
}

func ReversePrintRecv(head *ListNode, res *[]int) {
	if head != nil {
		ReversePrintRecv(head.Next, res)
		*res = append(*res, head.Val)
	}
}
````













# 剑指Offer（三）：从尾到头打印链表
# 剑指Offer（十四）：链表中倒数第k个结点
# 剑指Offer（十五）：反转链表
# 剑指Offer（十六）：合并两个排序的链表
# 剑指Offer（二十五）：复杂链表的复制
* 第一步：复制复杂指针的label和next。但是这次我们把复制的结点跟在元结点后面，而不是直接创建新的链表；
* 第二步：设置复制出来的结点的random。因为新旧结点是前后对应关系，所以也是一步就能找到random；
* 第三步：拆分链表。奇数是原链表，偶数是复制的链表。
 ![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20190602182910381_1897425017.png)
# 剑指Offer（三十六）：两个链表的第一个公共结点
![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20190602183040563_1940658650.png)
* 方法一：
    * 我们可以把两个链表拼接起来，一个pHead1在前pHead2在后，一个pHead2在前pHead1在后。这样，生成了两个相同长度的链表，那么我们只要同时遍历这两个表，就一定能找到公共结点。时间复杂度O(m+n)，空间复杂度O(m+n)。
    ![](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/20190602184139386_668299914.png)
* 方法二：
    * 我们也可以先让把长的链表的头砍掉，让两个链表长度相同，这样，同时遍历也能找到公共结点。此时，时间复杂度O(m+n)，空间复杂度为O(MAX(m,n))。

# 剑指Offer（五十五）：链表中环的入口结点

# 剑指Offer（五十六）：删除链表中重复的结点

# 二叉树(12道)：
剑指Offer（四）：重建二叉树
剑指Offer（十七）：树的子结构
剑指Offer（十八）：二叉树的镜像
剑指Offer（二十二）：从上往下打印二叉树
剑指Offer（二十四）：二叉树中和为某一值的路径
剑指Offer（三十八）：二叉树的深度
剑指Offer（三十九）：平衡二叉树
剑指Offer（五十七）：二叉树的下一个结点
剑指Offer（五十八）：对称的二叉树
剑指Offer（五十九）：按之字顺序打印二叉树
剑指Offer（六十）：把二叉树打印成多行
剑指Offer（六十一）：序列化二叉树
二叉搜索树(3道)：
剑指Offer（二十三）：二叉搜索树的后序遍历序列
剑指Offer（二十六）：二叉搜索树与双向链表
剑指Offer（六十二）：二叉搜索树的第k个结点
数组(11道)：
剑指Offer（一）：二维数组中的查找
剑指Offer（六）：旋转数组的最小数字
剑指Offer（十三）：调整数组顺序使奇数位于偶数前面
剑指Offer（二十八）：数组中出现次数超过一半的数字
剑指Offer（三十）：连续子数组的最大和
剑指Offer（三十二）：把数组排成最小的数
剑指Offer（三十五）：数组中的逆序对
剑指Offer（三十七）：数字在排序数组中出现的次数
剑指Offer（四十）：数组中只出现一次的数字
剑指Offer（五十）：数组中重复的数字
剑指Offer（五十一）：构建乘积数组
字符串(8道)：
剑指Offer(二)：替换空格
剑指Offer（二十七）：字符串的排列
剑指Offer（三十四）：第一个只出现一次的字符
剑指Offer（四十三）：左旋转字符串
剑指Offer（四十四）：翻转单词顺序序列
剑指Offer（四十九）：把字符串转换成整数
剑指Offer（五十二）：正则表达式匹配
剑指Offer（五十三）：表示数值的字符串
栈(3道)：
剑指Offer（五）：用两个栈实现队列
剑指Offer（二十）：包含min函数的栈
剑指Offer（二十一）：栈的压入、弹出序列
递归(4道)：
剑指Offer（七）：裴波那契数列
剑指Offer（八）：跳台阶
剑指Offer（九）：变态跳台阶
剑指Offer（十）：矩形覆盖
回溯法(2道)：
剑指Offer（六十五）：矩阵中的路径
剑指Offer（六十六）：机器人的运动范围
其他(15道)：
剑指Offer（十一）：二进制中1的个数
剑指Offer（十二）：数值的整数次方
剑指Offer（十九）：顺时针打印矩阵
剑指Offer（二十九）：最小的K个数
剑指Offer（三十一）：整数中1出现的次数（从1到n整数中1出现的次数）
剑指Offer（三十三）：丑数
剑指Offer（四十一）：和为S的连续正数序列
剑指Offer（四十二）：和为S的两个数字
剑指Offer（四十五）：扑克牌顺子
剑指Offer（四十六）：孩子们的游戏（圆圈中最后剩下的数）
剑指Offer（四十七）：求1+2+3+…+n
剑指Offer（四十八）：不用加减乘除的加法
剑指Offer（五十四）：字符流中第一个不重复的字符
剑指Offer（六十三）：数据流中的中位数
剑指Offer（六十四）：滑动窗口的最大值
