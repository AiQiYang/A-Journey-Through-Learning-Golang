# 链表

## Hot100练习题解
### 21. [合并两个有序链表](https://leetcode.cn/problems/merge-two-sorted-lists/description/?envType=study-plan-v2&envId=top-100-liked)
#### 思路
我们只需要同时循环遍历两个链表，并比较两个链表的元素，判断哪个链表的元素小，就把它放进新的链表中。最后当有一个链表到达表尾（nil）后，将还有元素的那个链表中的所有元素放进新链表中。  

这里需要注意的是，可以先创建一个哨兵节点（dummy），作为合并后的新链表中的第一个节点。这样子做可以避免在循环初期，单独处理头节点的操作，从而简化代码。

#### 代码
````go
func mergeTwoLists(list1 *ListNode, list2 *ListNode) *ListNode {
    dummy := &ListNode{Val: 0, Next: nil}
    cur := dummy

    // 同时循环两个链表，将元素小的那方加入新链表中
    for list1 != nil && list2 != nil {
        if list1.Val < list2.Val {
            cur.Next = list1
            list1 = list1.Next
        } else {
            cur.Next = list2
            list2 = list2.Next
        }
        cur = cur.Next
    }

    // 判断那个链表还有节点，并将它们合并到新链表中
    if list1 != nil {
        cur.Next = list1
    } else {
        cur.Next = list2
    }
    return dummy.Next
}
````

#### 复杂度
+ 时间复杂度：$O(n + m)$，其中`m`是`list1`的长度，`m`是`list2`的长度
+ 空间复杂度：$O(1)$