---
layout: post
title:  "Long的读写不是原子操作"
date:   2020-08-21 18:20:26 +0800
categories: concurrency
---
其实把自己学到的知识公开发布不是一件历史悠久的事情
对99%的人来说，所谓写博客无非是一种抬高自己身价好把自己卖个好价钱的行为罢了，否则用自己的印象笔记之类的软件就够了


jvm中不保证long/double的写操作是原子性操作：
17.7. Non-atomic Treatment of double and long
For the purposes of the Java programming language memory model, a single write to a non-volatile long or double value is treated as two separate writes: one to each 32-bit half. This can result in a situation where a thread sees the first 32 bits of a 64-bit value from one write, and the second 32 bits from another write.

Writes and reads of volatile long and double values are always atomic.

Writes to and reads of references are always atomic, regardless of whether they are implemented as 32-bit or 64-bit values.

Some implementations may find it convenient to divide a single write action on a 64-bit long or double value into two write actions on adjacent 32-bit values. For efficiency's sake, this behavior is implementation-specific; an implementation of the Java Virtual Machine is free to perform writes to long and double values atomically or in two parts.

Implementations of the Java Virtual Machine are encouraged to avoid splitting 64-bit values where possible. Programmers are encouraged to declare shared 64-bit values as volatile or synchronize their programs correctly to avoid possible complications.
不过貌似没提到读的操作

运行一下验证一下：
    public class UnsafeLong {
        long value;
        
        public static void main(String[] args) {
            new UnsafeLong().main();
        }
        public void main() {
            String arch = System.getProperty("os.arch");
            System.out.println(arch);

            if (arch.contains("64")) {
                return;
            }

            new Thread(new Runnable() {
                @Override
                public void run() {
                    while (true) {
                        value = 1;
                    }
                }
            }).start();

            new Thread(new Runnable() {
                @Override
                public void run() {
                    while (true) {
                        value = Long.MIN_VALUE;
                    }
                }
            }).start();

            while (true) {
                long v = value;
                if (v != 1 && v != Long.MIN_VALUE) {
                    System.out.println("bad value!");
                }
            }
        }
    }

发现没有输出。。。
因为机器是64位的
大概64位机器上long的处理是一个原子性操作，反正有人是这么说的：https://stackoverflow.com/a/1191390/5637606

只好新建了一个32位的虚拟机来验证了，我发现virutal box真的挺烂的，还是用安卓的emulator吧：

    public class LongNotAtomicActivity extends AppCompatActivity {

        long value;

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_long_not_atomic);
        }

        public void runIt(View view) {
            String arch = System.getProperty("os.arch");
            Log.d("os.arch " + arch);
            if (arch != null && arch.contains("64")) {
                Toast.makeText(this, "running on 64bit machine, return", Toast.LENGTH_SHORT).show();
                return;
            }

            Thread writer1 = new Thread(new Runnable() {
                @Override
                public void run() {
                    while (!Thread.interrupted()) {
                        value = 1;
                    }
                }
            });


            Thread writer2 = new Thread(new Runnable() {
                @Override
                public void run() {
                    while (!Thread.interrupted()) {
                        value = Long.MIN_VALUE;
                    }
                }
            });
            writer1.start();
            writer2.start();

            new Thread(new Runnable() {
                @Override
                public void run() {
                    while (true) {
                        long v = value;
                        if (v != 1 && v != Long.MIN_VALUE) {
                            Log.d("bad value found! " + v);
                            runOnUiThread(new Runnable() {
                                @Override
                                public void run() {
                                    Toast.makeText(LongNotAtomicActivity.this, "bad value found! " + v, Toast.LENGTH_SHORT).show();
                                }
                            });

                            writer1.interrupt();
                            writer2.interrupt();
                            break;
                        }
                    }
                }
            }).start();

        }
    }


发现确实是的：运行了两次的结果：
bad value found! 0
bad value found! -9223372036854775807
而且出现bad value的速度很快

给value加上volatile后, 跑了几秒后确实没有bad value