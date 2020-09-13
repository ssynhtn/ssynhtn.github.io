---
layout: post
title:  "leetcode 53 求连续元素之和最大值"
date:   2020-09-13 18:24:26 +0800
categories: leetcode
---

link: https://leetcode.com/problems/maximum-subarray

问题描述: 给定非空数组, 求它的连续元素之和的最大值, 要求连续元素的个数不为0(也就是所有元素都是负数的情况下, 要返回最大的负数)

solution: 

    fun maxSubArray(nums: IntArray): Int {
        var max = Int.MIN_VALUE

        var sum = 0
        for (num in nums) {
            if (num > max) {
                max = num
            }

            sum += num
            if (sum > max) {
                max = sum
            }

            if (sum < 0) {
                sum = 0
            }
        }

        return max
    }



我一开始想是不是可以用动态规划

首先为了让问题简单一些, 不考虑那种全是负数的情况

从第一个数开始
  * 如果它是负数, 那么应该抛弃它. 并且前面的连续负数都应该抛弃
  * 如果它是正数, 那么接下来的连续正数都应该加进来, 可以把它们压缩成一个数
  * 接着后面我们又会遇到连续的负数, 这里只要sum为正数, 那么就不放弃, 直到我们再次遇到正数. 如果我们遇到了sum为负数的情况, 那么抛弃, sum从0开始

严格证明就略了吧...

我觉得这个问题挺有意思的, 为什么是easy呢?

题目还要求给一个分治算法

如下:

    fun maxSubArrayDivConquer(nums: IntArray): Int {
        return maxSubArrayDivConquer(nums, 0, nums.size)[0]
    }

    // end > start
    // returns max, left max, right max, total, each requiring at least length one
    fun maxSubArrayDivConquer(nums: IntArray, start: Int, end: Int): IntArray {
        if (end - start == 1) {
            val ans = nums[start]
            return intArrayOf(ans, ans, ans, ans)
        }

        val mid = (start + end) / 2
        val max1 = maxSubArrayDivConquer(nums, start, mid)
        val max2 = maxSubArrayDivConquer(nums, mid, end)

    //        val total = max1[3] + max2[3]
    //        val leftMax = Math.max(max1[1], max1[3] + max2[1])
    //        val rightMax = Math.max(max2[2], max2[3] + max1[2])
    //        val max = Math.max(max1[0], Math.max(max2[0], max1[2] + max2[1]))

        max1[0] = Math.max(max1[0], Math.max(max2[0], max1[2] + max2[1]))
        max1[1] = Math.max(max1[1], max1[3] + max2[1])
        max1[2] = Math.max(max2[2], max2[3] + max1[2])
        max1[3] = max1[3] + max2[3]

        return max1

    }

maxSubArrayDivConquer这个函数返回四个数, 分别是最大的subarray, 包含最左边元素的最大的subarray, 包含最右边元素的最大的subarray, 整个array的和