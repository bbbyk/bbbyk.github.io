---
title: '1223. Dice Roll Simulation [Medium]'
date: 2019-10-13 16:23:36
categories: Leetcode
tags:
---
## 题目
A die simulator generates a random number from 1 to 6 for each roll. You introduced a constraint to the generator such that it cannot roll the number i more than rollMax[i] (1-indexed) consecutive times.

Given an array of integers rollMax and an integer n, return the number of distinct sequences that can be obtained with exact n rolls.

Two sequences are considered different if at least one element differs from each other. Since the answer may be too large, return it modulo 10^9 + 7.

Input: n = 2, rollMax = [1,1,2,2,2,3]
Output: 34
Explanation: There will be 2 rolls of die, if there are no constraints on the die, there are 6 * 6 = 36 possible combinations. In this case, looking at rollMax array, the numbers 1 and 2 appear at most once consecutively, therefore sequences (1,1) and (2,2) cannot occur, so the final answer is 36-2 = 34.

Constraints:
* 1 <= n <= 5000
* rollMax.length == 6
* 1 <= rollMax[i] <= 15

本题目是Contest 158第3题，第一次做Leetcode的Contest只对了前两题，对第三题没有头绪，后经过discuss大佬的题解还是觉得分析清楚了并没想象中的那么难。
[参考：　[Chinese/C++] 1223. 【动态规划】有限制的掷骰子（详解）][c0b8b0ea]

  [c0b8b0ea]: https://leetcode.com/problems/dice-roll-simulation/discuss/403736/ChineseC%2B%2B-1223. "[Chinese/C++] 1223. 【动态规划】有限制的掷骰子（详解）"
