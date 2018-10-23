---
layout:     post
title:      Java并发控制
subtitle:   -CountDownLatch
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
# CountDownLatch用法

CountDownLatch类位于java.util.concurrent包下，利用它可以实现类似计数器的功能。比如有一个任务A，它要等待其他4个任务执行完毕之后才能执行，此时就可以利用CountDownLatch来实现这种功能了

### 一、CountDownLatch提供的一个唯一构造器

    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

参数count为计数器

### 二、3个CountDownLatch中最重要的方法

    //调用await的线程会被挂起，它会等待直到count值为0才会继续执行
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    public void countDown() {  //将count减1
        sync.releaseShared(1);
    }

### 三、实例

    import java.util.concurrent.CountDownLatch;

    public class TestCountDownLatch {
        public static void main(String[] args) {
            final CountDownLatch latch = new CountDownLatch(2);

            new Thread(){
                public void run() {
                    try {
                        System.out.println("子线程"+Thread.currentThread().getName()+"正在执行");
                        Thread.sleep(3000);
                        System.out.println("子线程"+Thread.currentThread().getName()+"执行完毕");
                        latch.countDown();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                };
            }.start();

            new Thread(){
                public void run() {
                    try {
                        System.out.println("子线程"+Thread.currentThread().getName()+"正在执行");
                        Thread.sleep(3000);
                        System.out.println("子线程"+Thread.currentThread().getName()+"执行完毕");
                        latch.countDown();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                };
            }.start();

            try {
                System.out.println("等待2个子线程执行完毕...");
                latch.await();
                System.out.println("2个子线程已经执行完毕");
                System.out.println("继续执行主线程");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

执行结果：

    等待2个子线程执行完毕...
    子线程Thread-0正在执行
    子线程Thread-1正在执行
    子线程Thread-1执行完毕
    子线程Thread-0执行完毕
    2个子线程已经执行完毕
    继续执行主线程
