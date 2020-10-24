---
layout: post
title:  "leetcode 132 Pattern"
date:   2020-10-24 18:24:26 +0800
categories: algorithm
---

https://leetcode.com/problems/132-pattern

Basically we are given an array and we want to find 3 numbers a, c, b in the array from left to right, such that the "peak" c is in the middle, the smallest a is on the left, and b, whose value is strictly bewteen a and c is on the right

This problem is assigned a medium level, but to find an O(n) algorithm, it's pretty hard. Here I'll present one method that is O(N) in both time and space compelexity and is also not too tricky

The idea is as follows: if we scan the array from left to right, and suppose we somehow can keep a pair of (a, c), and we want to find the b to form a 132 triple, then when we consider the next number t, 
there are 5 cases, and 4 of them are trivial, but the last one will require us to keep more info than merely a pair of (a, c)
    
    * if t > c, then we replace c with t, so we have a larger interval
    * if t == c, same as above, or we skip this t
    * if t is in (a, c) then return true
    * if t == a, then skip t, becasue if this t were the starting point of a 132 pattern, we can also form the pattern with the a we have (but of course the c we currently have would not be the c in the found pattern)
    * if t < a, this is the non trivial case

for the last case, t could be a new starting point of a 132 pattern, but we can not throw away the (a, c) we currently have, because the next number after t could be the b we wanted with (a, c).  
Which means keeping one pair of (a, c) is not enough, but we have to keep a list of descending (a, c)s


now we form the algorithm, as follows:

    we keep a list of descending intervals, represented by two arrays: highs and lows, such that highs[i] > lows[i] >= highs[i+1] > lows[i+1]...
    we keep the mininum value so far in x, since it's the min value, if the intervals are not empty, we have x <= min(lows)  
    there is also an invariant that helps to visualize the whole process: that x is either the low of the current rightmost interval, or the index of x is to the right of all intervals

    first let x = nums[0]
    for i = 1 until n, let t = nums[i]
    if t < x, replace x with t and continue
    else if t == x, continue
    else if t > x
        scan the intervals from right to left, until we find the first interval whose high is > t
            case 1: we can't find such an interval, that is, either there is no intervals, or t >= highs[0], then we clear all intervals and add (x, t) as a superseding "larger" interval
            case 2: we found the first high that is > t
                if the coresponding low is < t, then we found a 132 pattern which is (low, high, t)
                if the coresponding low is >= t, then we clear all intervals to the right of this interval, and we add (x, t) as a replacement

    if we exhausted all nums, we are left with a bunch of intervals, then we return false


Complete code(which finished in 1ms in leetcode):

    public boolean find132pattern(int[] nums) {
        int n = nums.length;
        if (n < 3) return false;

        int[] highs = new int[n];
        int[] lows = nums;  // reusing nums as lows
        int s = 0;  // size of intervals
        int x = nums[0];
        int i = 1;
        while (i < n) {
            int t = nums[i++];
            if (t <= x) {
                x = t;
                continue;
            }

            // t > x
            int k = s - 1;
            while (k >= 0 && t >= highs[k]) {
                k--;
            }

            if (k >= 0 && t > lows[k]) return true;
            s = k + 1;

            lows[s] = x;
            highs[s] = t;
            s++;

        }


        return false;
    }


Time space analysis:

space: we need space to store highs and lows, worst case is every number is either a high or a low, for example [5,6,3,4,1,2], so worst case O(N), if we use ArrayLists to store highs and highs, best case can be O(1)

time: to scan from left to right we need O(N) time in the worst case, however in some cases, we also need to compare t with exisiting intervals, the number of comparisons done is the same as the number of intervals discarded(replaced by the superseding interval (x, t)), and int the whole process we can discard at most N/2 intervals, so total time is also worst case O(N). Similar to space, best case is O(1)

This algorithm at this point is very intuitive and simple, however at first it's hard to make sure what exactly to do with t in the case of t > min