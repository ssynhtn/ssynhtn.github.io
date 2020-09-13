---
layout: post
title:  "leetcode 46 permutations"
date:   2020-09-13 18:21:26 +0800
categories: leetcode
---

link: https://leetcode.com/problems/permutations/

需求: 给定n个不同的数, 输出这n个数的所有排列

solution: 

    class Permutations {
        fun permute(nums: IntArray): List<List<Int>> {
            val collect = mutableListOf<List<Int>>()
            permute(nums, 0, collect)
            return collect
        }


        fun permute(nums: IntArray, fixCount: Int, collect: MutableList<List<Int>>) {
            if (fixCount >= nums.size - 1) {
                collect.add(nums.toList())
                return
            }

            permute(nums, fixCount + 1, collect)

            for (i in fixCount + 1 until nums.size) {
                val temp = nums[fixCount]
                nums[fixCount] = nums[i]
                nums[i] = temp
                permute(nums, fixCount + 1, collect)
                nums[i] = nums[fixCount]
                nums[fixCount] = temp
            }
        }
    }

    fun main(args: Array<String>) {
        val nums = intArrayOf(1, 2, 3)
        val ps = Permutations().permute(nums)
        println(ps)
        println(nums.toList())


    }


典型的回溯算法, 递归的permute函数会向collect中添加前fixCount个元素固定, 其余元素进行排列得到的序列
base case为fixCount为全部
非base case, 考虑第fixCount + 1这个元素, 它的取值是剩余元素之一, 对每种情况进行递归后, 将它放回原位