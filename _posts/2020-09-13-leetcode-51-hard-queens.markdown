---
layout: post
title:  "leetcode 51 N皇后问题"
date:   2020-09-13 18:23:26 +0800
categories: leetcode
---

link: https://leetcode.com/problems/n-queens/

solution: 

    class NQueens {
        fun solveNQueens(n: Int): List<List<String>> {
            val table = Array<CharArray>(n, {i -> CharArray(n)})
            for (i in 0 until n) {
                for (j in 0 until n) {
                    table[i][j] = '.'
                }
            }

            val collection = mutableListOf<List<String>>()

            putQueens(table, 0, collection)

            return collection

        }

        private fun putQueens(table: Array<CharArray>, fixedRows: Int, collection: MutableList<List<String>>) {
            if (fixedRows == table.size) {
                collection.add(table.map { chs -> String(chs) })
                return
            }

            for (col in 0 until table.size) {
                if (checkHasConflictWithPrev(table, fixedRows, col)) continue
                table[fixedRows][col] = 'Q'
                putQueens(table, fixedRows + 1, collection)
                table[fixedRows][col] = '.'

            }
        }

        private fun checkHasConflictWithPrev(table: Array<CharArray>, row: Int, col: Int): Boolean {
            for (i in 0 until row) {
                var j = col - row + i
                if (j >= 0 && j < table.size) {
                    if (table[i][j] == 'Q') {
                        return true
                    }
                }
                j = row + col - i
                if (j >= 0 && j < table.size) {
                    if (table[i][j] == 'Q') {
                        return true
                    }
                }
                j = col
                if (j >= 0 && j < table.size) {
                    if (table[i][j] == 'Q') {
                        return true
                    }
                }
            }
            return false
        }
    }



这个问题曾经给我带来不小的伤害

当时我什么都不懂的时候打开了zju online judge, 随便打开了一个看上去是easy级别的题目(话说这个在leetcode上竟然是hard级别的, 可以证明leetcode是相当之不hard core了)

先是被英文题干给震撼到了

然后看了题目, 在棋盘上如何放置8个皇后可以防止她们相互攻击到

当时在脑内画了半天, 怎么都无法想出来, 因为当你尝试给棋盘上不同的行/列放皇后的时候, 放了后面的就会忘了前面的

well, 在明白了递归/回溯的方法后, 这类问题就是很常规的了

putQueens这个函数会在给定fixCount个皇后的情况下, 放置后面皇后, 将结果加入collection, 并且当方法返回时会恢复棋盘上的内容和输入时一致