---
title: 云064带你刷leetcode(2)
date: 2018-06-26 18:21:29
categories: leetcode
tags: [algorithm,c++,leetcode]
---

## 1. 题目描述
[两数相加](https://leetcode-cn.com/problems/add-two-numbers/description/)
给定两个非空链表来表示两个非负整数。位数按照逆序方式存储，它们的每个节点只存储单个数字。将两数相加返回一个新的链表。

你可以假设除了数字 0 之外，这两个数字都不会以零开头。

## 2. 示例
> 输入：(2 -> 4 -> 3) + (5 -> 6 -> 4)
> 输出：7 -> 0 -> 8
> 原因：342 + 465 = 807

<!--more-->

---

## 3. 题目解答
### 3.1 思路
本题比较简单，单纯模拟纸上计算两个数字相加即可，只需要遵循十进制加法，然后注意一下进位和边界问题就行了。思路描述如下：

1. 如果`l1`或者`l2`为空时，返回另外一个链表即可
2. 定义`carry`变量，用于记录前一位加法的进位，初始化为0
3. 遍历两个链表`l1`和`l2`，得到`node1`和`node2`
4. 如果`node1`和`node2`都为空，并且`carry`为0，则计算结束，返回结果链表
5. 如果节点`node`为`nullptr`，则记录值为`0`，否则记录值为`node->val`。可以得到`node1`和`node2`对应的两个值`val1`和`val2`
6. 计算`val1+val2+carry`的结果`result`
7. 如果大于`10`，则取对`10`的余数`mod`，新增一个`val`为`mod`的节点添加到结果链表中；对`10`的商记录到`carry`中。继续执行第3步
8. 如果小于`10`且大于`0`，则直接新增一个`val`为`result`的节点添加到结果链表中；`carry`赋值为0。继续执行第3步

### 3.2 算法复杂度
时间复杂度： `O(m+n)`
空间复杂度： `O(m+n)`

### 3.3 代码实现
```cpp
class Solution {
public:
    ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
        if (l1 == nullptr) return l2;
        if (l2 == nullptr) return l1;

        int carry = 0;
        ListNode* result = nullptr;
        ListNode* prev = nullptr;

        ListNode* node1 = l1;
        ListNode* node2 = l2;
        while (node1 != nullptr || node2 != nullptr || carry != 0) {
            int val1 = (node1 == nullptr ? 0 : node1->val);
            int val2 = (node2 == nullptr ? 0 : node2->val);
            int sum = val1 + val2 + carry;
            int val = sum % 10;
            carry = sum / 10;
            ListNode* node = new ListNode(val);
            if (prev == nullptr) {
                result = node;
            } else {
                prev->next = node;
            }

            prev = node;
            node1 = (node1 != nullptr ? node1->next : nullptr);
            node2 = (node1 != nullptr ? node2->next : nullptr);
        }

        return result;
    }
};
```
