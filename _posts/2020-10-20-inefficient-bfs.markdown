---
layout: post
title:  "一种错误(低效)的bfs实现方式"
date:   2020-10-20 18:24:26 +0800
categories: algorithm
---

今天试图去做一道题:  
https://leetcode.com/problems/cut-off-trees-for-golf-event/

这道题最后会归结为给定m*n的矩阵, 去掉值为0的那些格子, 其它每个格子作为一个顶点, 求其中两点间的最短路径的问题

我一开始是这么写的, 在一个node出q的时候标记它为visited:  

    bfs(from, to) {
      q = new Queue
      q.add(from)
      visited = new Set

      while (q not empty) {
        node = q.poll
        if (node == to) found!!
        add node to visited

        for (each neighbor of node) {
          if (not in visited) {
            add to q
          }
        }
      }

      not found
    }

这个做法对几个数据测试了一下似乎是正确的, 但是提交后竟然是超时

仔细看看, 上面这个算法很容易发生往q里重复添加node的问题  
以V = [S, A, B, C, D], E = {S->{AB}, A->C, B->C, C->D} 为例, q和visited的内容变化情况是  

    S, {}
    AB, S
    BC, SA
    CC, SAB
    CD, SABC
    DD, SABC
    D, SABCD
    {}, SABCD

C和D都被添加了两次

如果在遍历node的邻居之前检查一下node是否已经visit过, 如果visit过了, 就跳过它的邻居, 实际运行时间会好很多

    bfs(from, to) {
      q = new Queue
      q.add(from)
      visited = new Set

      while (q not empty) {
        node = q.poll
        if (node == to) found!!
        if (node in visited) continue;  //+++++
        add node to visited

        for (each neighbor of node) {
          if (not in visited) {
            add to q
          }
        }
      }

      not found
    }

但是还是会有重复添加

    S, {}
    AB, S
    BC, SA
    CC, SAB
    CD, SABC
    D, SABCD
    {}, SABCD
  
在处理q中的第二个C的时候, D不再被重复添加, 但是C本身是A和B的邻居, 被重复添加了



正确的做法是: 在一个node入q的时候就应该标记为visited, 这样保证一个node不会被重复添加

    bfs(from, to) {
      q = new Queue
      q.add(from)
      visited = new Set
      add from to visited
      
      while (q not empty) {
        node = q.poll
        if (node == to) found!!

        for (each neighbor) {
          if (not in visited) {
            add neighbor to q
            add neighbor to visited
          }
        }
      }

      not found
    }