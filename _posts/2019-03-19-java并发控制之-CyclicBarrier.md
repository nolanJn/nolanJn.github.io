---
layout:     post
title:      Java并发控制
subtitle:   -CyclicBarrier
date:       2019-03-19
author:     Nolan
header-img: img/post-bg-ios10.jpg
catalog: true
tags:
    - java
    - JDK1.8
    - 并发
    - 多线程
    
---
# CyclicBarrier用法

字面意思回环栅栏，通过它可以实现让一组线程等待至某个状态之后再全部同时执行。叫做回环是因为当所有等待线程都被释放以后，CyclicBarrier可以被重用。我们暂且把这个状态就叫做barrier，当调用await()方法之后，线程就处于barrier了

### 一、CyclicBarrier位于java.util.concurrent包下，CyclicBarrier提供了两个构造器

    public CyclicBarrier(int parties, Runnable barrierAction) {
    }
     
    public CyclicBarrier(int parties) {
    }

参数parties指让多少个线程或者任务等待至barrier状态；参数barrierAction为当这些线程都达到barrier状态时会执行的内容。

### 二、CyclicBarrier中最重要的方法就是await方法，它有2个重载版本：

    public int await() throws InterruptedException, BrokenBarrierException { };

    public int await(long timeout, TimeUnit unit)throws InterruptedException,BrokenBarrierException,TimeoutException { };

第一个版本比较常用，用来挂起当前线程，直至所有线程都到达barrier状态再同时执行后续任务；

第二个版本是让这些线程等待至一定的时间，如果还有线程没有到达barrier状态就直接让到达barrier的线程执行后续任务。

### 三、实例

    public class TestCyclicBarrier {
        public static void main(String[] args) {
            int N = 4;
            CyclicBarrier barrier  = new CyclicBarrier(N, new Runnable() {
                @Override
                public void run() {
                    System.out.println("所有线程写入完毕，继续处理其他任务...");
                }
            });
            for(int i=0;i<N;i++)
                new Writer(barrier).start();
        }

        static class Writer extends Thread{
            private CyclicBarrier cyclicBarrier;
            public Writer(CyclicBarrier cyclicBarrier) {
                this.cyclicBarrier = cyclicBarrier;
            }

            @Override
            public void run() {
                System.out.println("线程"+Thread.currentThread().getName()+"正在写入数据...");
                try {
                    Thread.sleep(5000);      //以睡眠来模拟写入数据操作
                    System.out.println("线程"+Thread.currentThread().getName()+"写入数据完毕，等待其他线程写入完毕");
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }catch(BrokenBarrierException e){
                    e.printStackTrace();
                }
            }
        }
    }

执行结果：

    线程Thread-0正在写入数据...
    线程Thread-1正在写入数据...
    线程Thread-2正在写入数据...
    线程Thread-3正在写入数据...
    线程Thread-3写入数据完毕，等待其他线程写入完毕
    线程Thread-2写入数据完毕，等待其他线程写入完毕
    线程Thread-1写入数据完毕，等待其他线程写入完毕
    线程Thread-0写入数据完毕，等待其他线程写入完毕
    所有线程写入完毕，继续处理其他任务...

### CyclicBarrier是可重用的

    public class TestCyclicBarrierReusing {
        public static void main(String[] args) {
            int N = 4;
            CyclicBarrier barrier  = new CyclicBarrier(N);

            for(int i=0;i<N;i++) {
                new Writer(barrier).start();
            }

            try {
                Thread.sleep(25000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("CyclicBarrier重用");

            for(int i=0;i<N;i++) {
                new Writer(barrier).start();
            }
        }
        static class Writer extends Thread{
            private CyclicBarrier cyclicBarrier;
            public Writer(CyclicBarrier cyclicBarrier) {
                this.cyclicBarrier = cyclicBarrier;
            }

            @Override
            public void run() {
                System.out.println("线程"+Thread.currentThread().getName()+"正在写入数据...");
                try {
                    Thread.sleep(5000);      //以睡眠来模拟写入数据操作
                    System.out.println("线程"+Thread.currentThread().getName()+"写入数据完毕，等待其他线程写入完毕");

                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }catch(BrokenBarrierException e){
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName()+"所有线程写入完毕，继续处理其他任务...");
            }
        }
    }
