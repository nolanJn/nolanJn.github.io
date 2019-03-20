---
layout:     post
title:      Java并发控制
subtitle:   -Semaphore
date:       2019-03-19
author:     Nolan
header-img: img/post-bg-ios10.jpg
catalog: true
tags:
    - java
    - JDK1.8
    - 并发控制
    - 多线程
    - 信号量 
    
---
# Semaphore用法

Semaphore翻译成字面意思为信号量，Semaphore可以控制同时访问的线程个数，通过acquire()获取一个许可，如果没有就等待，而release释放一个许可。

### 一、Semaphore位于java.util.concurrent包下，Semaphore提供了两个构造器

    public Semaphore(int permits) {          //参数permits表示许可数目，即同时可以允许多少线程进行访问
    sync = new NonfairSync(permits);
    }

    public Semaphore(int permits, boolean fair) {    //这个多了一个参数fair表示是否是公平的，即等待时间越久的越先获取许可
        sync = (fair)? new FairSync(permits) : new NonfairSync(permits);
    }

### 二、比较重要的几个方法：

    public void acquire() throws InterruptedException {  }     //获取一个许可

    public void acquire(int permits) throws InterruptedException { }    //获取permits个许可

    public void release() { }          //释放一个许可

    public void release(int permits) { }    //释放permits个许可

acquire()用来获取一个许可，若无许可能够获得，则会一直等待，直到获得许可。

release()用来释放许可。注意，在释放许可之前，必须先获获得许可。

这4个方法都会被阻塞，如果想立即得到执行结果，可以使用下面几个方法：

    //尝试获取一个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
    public boolean tryAcquire() { };    

    //尝试获取一个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
    public boolean tryAcquire(long timeout, TimeUnit unit) throws InterruptedException { };  
    
    //尝试获取permits个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
    public boolean tryAcquire(int permits) { }; 
    
    //尝试获取permits个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
    public boolean tryAcquire(int permits, long timeout, TimeUnit unit) throws InterruptedException { }; 

另外还可以通过availablePermits()方法得到可用的许可数目。

### 三、具体使用示例

假若一个工厂有5台机器，但是有8个工人，一台机器同时只能被一个工人使用，只有使用完了，其他工人才能继续使用。那么我们就可以通过Semaphore来实现：

    public class TestCountDownLatch {
        public static void main(String[] args) {
            int N = 8;            //工人数
            Semaphore semaphore = new Semaphore(5); //机器数目
            for(int i=0;i<N;i++)
                new Worker(i,semaphore).start();
        }

        static class Worker extends Thread{
            private int num;
            private Semaphore semaphore;
            public Worker(int num,Semaphore semaphore){
                this.num = num;
                this.semaphore = semaphore;
            }

            @Override
            public void run() {
                try {
                    semaphore.acquire();
                    System.out.println("工人"+this.num+"占用一个机器在生产...");
                    Thread.sleep(2000);
                    System.out.println("工人"+this.num+"释放出机器");
                    semaphore.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    运行结果：
    工人1占用一个机器在生产...
    工人0占用一个机器在生产...
    工人2占用一个机器在生产...
    工人3占用一个机器在生产...
    工人4占用一个机器在生产...
    工人2释放出机器
    工人1释放出机器
    工人4释放出机器
    工人0释放出机器
    工人5占用一个机器在生产...
    工人6占用一个机器在生产...
    工人7占用一个机器在生产...
    工人3释放出机器
    工人6释放出机器
    工人5释放出机器
    工人7释放出机器
