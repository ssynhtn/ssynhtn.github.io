---
layout: post
title:  "leetcode 47 permutations 2"
date:   2020-09-13 18:22:26 +0800
categories: leetcode
---

link: https://leetcode.com/problems/permutations-ii/

需求: 给定n个数(可能相同), 输出这n个数的所有排列, 相同的排列只输出一次

solution: 

    class Permutations {
        fun permuteUnique(nums: IntArray): List<List<Int>> {
            nums.sort()
            val collection = mutableListOf<List<Int>>()
            permute(nums, 0, collection)
            return collection

        }

        private fun permute(nums: IntArray, fixCount: Int, collection: MutableList<List<Int>>) {
        

            if (fixCount == nums.size) {
                collection.add(nums.toList())
                return
            }

            permute(nums, fixCount + 1, collection)
            val index = fixCount
            for (i in index + 1 until nums.size) {
                if (nums[i] == nums[i - 1]) continue

                shiftRight(nums, index, i)
                permute(nums, fixCount + 1, collection)
                shiftLeft(nums, index, i)
            }
        }

        private fun shiftLeft(nums: IntArray, index: Int, i: Int) {
            val temp = nums[index]
            System.arraycopy(nums, index + 1, nums, index, i - index)
            nums[i] = temp
        }

        private fun shiftRight(nums: IntArray, index: Int, i: Int) {
            val temp = nums[i]
            System.arraycopy(nums, index, nums, index + 1, i - index)
            nums[index] = temp
        }
    }


解释: 和46题一样, 唯一的区别是始终保持剩余数字为递增的顺序, 这样对fixCount+1这位上的数进行选择时, 跳过那些和前一位相同的数