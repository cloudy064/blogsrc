---
title: 云064带你刷leetcode(1)
date: 2016-10-18 18:21:29
categories: leetcode
tags: [algorithm,c++,leetcode]
---

## 1. 题目描述
[Two Sum](https://leetcode.com/problems/two-sum/)
输入：数组nums，数字target
输出：两个下标x和y，使得数组a中这两个下标的数的和为t (a[x]+a[y]==t)
注意：题目中说明了，假设数组中只有一组下标符合条件

## 2. 示例
> 输入 nums=[2,7,11,15], target=9
> 因为 nums[0] + nums[1] = 2 + 7 = 9
> 所以 输出 [0,1]

<!--more-->

---

## 3. 题目解答
### 3.1 排序
#### 3.1.1 思路
算法思路描述如下：
1. 对数组有大到下进行排序（题目并未指出输入的数组是否为已排序）
2. 将头部(`a[i]`)与尾部(`a[j]`)的和`sum`作为初始值
3. 如果`sum`大于`target`，说明值取大了，则将尾部指针向前移动一位(`j -= 1`)，继续执行第2步，否则执行第4步
4. 如果`sum`小于`target`，说明值取小了，则将头部指针向后移动一位(`i += 1`)，继续执行第2步，否则执行第5步
5. 找到了`i`和`j`，直接返回

但是这里有一个问题，题目要求返回的是原数组中的下标，如果我们直接给他排序了，下标就丢失了，所以我们需要另外创建一个结构体保存对应数字的下标，或者也可以用一个`map`来存储数字对应的下表

#### 3.1.2 算法复杂度
时间复杂度： `O(nlogn)`
空间复杂度： `O(n)`

#### 3.1.3 代码实现
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

---

### 3.2 哈希表
#### 3.2.1 思路
算法思路描述如下：
1. 遍历一次数组，以具体的数字作为`key`，以下表作为`value`，记录到`map`中
2. 在遍历一次数组，针对每一个数字`a`，计算出`target-a`，在`map`中查找`target-a`是否存在，如果存在则直接返回`a`的下标和`map[target-a]`，否则重复执行2

#### 3.2.2 算法复杂度
时间复杂度： `O(n)`
空间复杂度： `O(n)`

#### 3.2.3 代码实现
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
