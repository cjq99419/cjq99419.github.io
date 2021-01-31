# Leetcode 206.反转链表



## 反转链表的n种实现（go）

### 一、双指针迭代法

```go
func reverseList(head *ListNode) *ListNode {
	if head == nil {
		return nil
	} else {
		var tmp *ListNode
		cur := head.Next
		pre := head
		for cur != nil {
			tmp = cur.Next
			cur.Next = pre
			pre = cur
			cur = tmp
		}
		head.Next = nil
		return pre
	}
}
```

![image-20200921114708051](https://tva1.sinaimg.cn/large/007S8ZIlly1giy4lt2sypj30e702wwet.jpg)

比较好理解的算法，即设置两个指针，其中**cur表示当前节点，pre表示前一个节点，tmp用作中间状态保存**。

首先由于cur需要指向head的next，因而需要**单独处理head为nil的情况**，初始状态即设置cur为head.next，pre为head

每次进行循环，当cur为nil，即pre为末尾节点时停止循环，每次循环中的目的就是把cur的next指向pre，并且更新pre和cur

由于cur设置next之后，无法向下移动，所以需要用tmp保存cur的next节点。

即tmp保存cur的next，将cur的next设置为pre，pre设置为cur，再把tmp保存的原next给cur，实现cur和pre的前移动。

**注意，最后要把尾节点head的next设置为nil，由于head最开始为pre，所以依然指向第二个节点，进而形成一二节点的环**。



### 二、递归法

```go
func reverseList(head *ListNode) *ListNode {
	if head == nil || head.Next == nil {
		return head
	} else {
		cur := reverseList(head.Next)
		head.Next.Next = head
		head.Next = nil
		return cur
	}
}
```

![image-20200921114753212](https://tva1.sinaimg.cn/large/007S8ZIlly1giy4mjz7jij30do02yglx.jpg)

建议看leetcode的题解，这里简单说明

递归法即首先递归到尾节点，之后通过递归栈道保存状态向前遍历。

每次将头节点的next的next设置为头，如链表1-2-3-4-5，当头为4时，将4.next即5的next设置为4，等于把指针方向反转一下，在返回递归状态来向前处理节点3，最后实现整个链表的反转。

注意返回的cur实际上并不变化，保存的就是链表的最后节点，其实可以定义全局量，在触发边界条件的时候保存，之后返回也可。

### 

### 三、对比与拓展

由于递归法隐式使用递归栈，实际上空间复杂度会稍高，两者时间复杂度相差不大，都在O(n)级别。

而且递归法不太好理解，详细可以看 [leetcode  206.反转链表](https://leetcode-cn.com/problems/reverse-linked-list/solution/)有几个题解讲的很细致。

此外也可以使用栈或者中间节点方式，如定义一个栈，第一次遍历进行push操作，之后pop再连接即可，但同样空间复杂度较高。

也可以定义一个空头节点，遍历到结尾递归返回的时候，把每个节点接上去，或者不递归直接迭代插入到首元素也可。





