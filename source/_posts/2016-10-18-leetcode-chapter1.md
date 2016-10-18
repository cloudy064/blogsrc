---
title: 云064带你刷leetcode(一)
date: 2016-10-18 18:21:29
categories: algorithm
tags: [algorithm,c++,leetcode]
---

## 前言
感觉好久没有更新博客了，最近一直在忙（其实你特么一直在沉迷《基三》），最近终于有点空来更新博客了。应大部分人的需求，我会尽量以每日一题的速度来写博客，并且保证都是原创（如果有借鉴我会写明出处）。而本人其实也很久没有练过算法了，大概有三年了吧，本次系列的题目都是从`leetcode`上摘录的（我会把题目的意思提炼出来），也是很多程序员面试之前必刷的题库，希望对大家有帮助。闲话不多刷，来看第一题吧。

## 题目描述
[Two Sum](https://leetcode.com/problems/two-sum/)
输入：数组nums，数字target
输出：两个下标x和y，使得数组a中这两个下标的数的和为t (a[x]+a[y]==t)
注意：题目中说明了，假设数组中只有一组下标符合条件

<!--more-->

## 示例
> 输入 nums=[2,7,11,15], target=9
> 因为 nums[0] + nums[1] = 2 + 7 = 9
> 所以 输出 [0,1]

## 题目分析
感觉`leetcode`上已经没有比这个更简单的了，当然你要是暴力求用两个循环来做，肯定会超时。遇到这种找数字的题目，其实第一个应该想到的就是对这个数组进行一次排序。之后的做法就是取两端的数相加，比较和`target`的大小，然后不断做调整(哎，我觉得都没啥好说的了)。所以这题的算法的伪代码如下：
```
sortnum <- sort(nums)
length <- sortnum.length
i <- 1
j <- length
while sortnum[i] + sortnum[j] != target
    if sortnum[i] + sortnum[j] > target 
        then j <- j-1
    else if sortnum[i] + sortnum[j] < target 
        then i <- i+1

return [sortnum[i] in nums, sortnum[j] in nums]
```

但是这里记住，题目要求返回的是原数组中的下标，如果我们直接给他排序了，下标就丢失了，所以我们需要另外创建一个结构体保存对应数字的下标。所以实际`C++`实现代码如下：
```cpp
struct Num {
    int num;    //原来的数
    int index;  //原来的下标

    bool operator<(const Num& rhs) {
        return num < rhs.num;
    }
};

class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        vector<Num> numsWithIndex(nums.size());

        // 初始化，保存原来的数和下标
        for (int i = 0; i < nums.size(); ++i) {
            numsWithIndex[i].num = nums[i];
            numsWithIndex[i].index = i;
        }

        sort(numsWithIndex.begin(), numsWithIndex.end());

        int smaller = 0;
        int larger = numsWithIndex.size() - 1;
        while (true) {
            int sum = numsWithIndex[smaller].num + numsWithIndex[larger].num;
            if (sum == target) {
                break;
            }

            if (sum > target) {
                --larger;
            }
            else {
                ++smaller;
            }
        }

        int indexa = numsWithIndex[smaller].index;
        int indexb = numsWithIndex[larger].index;

        // 这里还需要注意的就是返回要求从小到达的顺序
        vector<int> result;
        if (indexa > indexb) {
            result.push_back(indexb);
            result.push_back(indexa);
        }
        else {
            result.push_back(indexa);
            result.push_back(indexb);
        }
        return result;
    }
};
```

## 后面的话
第一章算法就这么被我水过去了，用了大概半个小时搞定了，嘿嘿嘿，其实只是因为题目简单。当然这题还有其他的解法，不需要排序，只需要一个哈希表就可以搞定了，但是最终跑出来的结果很震惊，耗时比排序版本的高很多，所以可见`leetcode`所用的`STL`对`sort`的优化，有机会分析下源码。

用哈希表的思路就是，因为对于一个特定的`target`，总能找到唯一的一个`a`使得`a+b`的结果为`target`。所以我们需要做的就是遍历这个数组，遇到一个数字`a`就看哈希表中`target-a`的`key`上有没有`value`，如果没有就将`a`加入到哈希表中，否则直接返回即可。算法的时间复杂度是`O(1)`。代码实现如下：
```cpp
class Solution
{
public:
	vector<int> twoSum(vector<int>& nums, int target) {
		vector<int> result;
		unordered_map<int, int> record;
		for (int i = 0; i < nums.size(); ++i)
		{
			int to_find = target - nums[i];
			unordered_map<int, int>::iterator it = record.find(to_find);
			if (it != record.end())
			{
				result.push_back(it->second);
				result.push_back(i);
				break;
			} 
			else
			{
				record.insert(make_pair(nums[i], i));
			}
		}

		return result;
	}
};
```

最终运行的结果比较如下：
{% asset_img solution_using_sort.png 排序 %}

{% asset_img solution_using_hashmap.png 哈希 %}
