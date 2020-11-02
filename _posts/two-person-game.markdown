---
layout: post
title:  "Properties of Dijkstra's Algorithm"
date:   2020-10-26 18:24:27 +0800
categories: algorithm
---

zemelo's 定理

0: losing 
x: if moving some n=k^2, y = x-n is losing, then x is winning
x: if moving any n=k^2, y = x-n is winning, then x is losing

if exist some x that is neither winning nor losing, wlog, x is the smallest such number

then if moving some n leads to losing, then x is winning
if moving any n leads to winning, then x is losing, contradiction


so every x is either winning or losing