---
layout: post
title:  "在final字段初始化之前访问它会怎么样"
date:   2020-08-21 18:20:27 +0800
categories: concurrency
---

一个并不是太重要到事情，但是它确实是这样：
final字段允许延迟到构造函数中赋值, 但是初始化它之前，却是可以访问它未初始化到值（0），compiler不会抱怨

    public class VisitFinalFieldBeforeSet {
        final int value;

        public VisitFinalFieldBeforeSet(int value) {
            new Visitor().visit(this);
            this.value = value;
        }

        @Override
        public String toString() {
            return "VisitFinalFieldBeforeSet@" + System.identityHashCode(this) +
                    "{" +
                    "value=" + value +
                    '}';
        }

        public static class Visitor {
            public void visit(VisitFinalFieldBeforeSet set) {
                System.out.println(set);
            }

        }

        public static void main(String[] args) {
            VisitFinalFieldBeforeSet set = new VisitFinalFieldBeforeSet(100);
            System.out.println(set);
        }
    }

output:
VisitFinalFieldBeforeSet@1846274136{value=0}
VisitFinalFieldBeforeSet@1846274136{value=100}