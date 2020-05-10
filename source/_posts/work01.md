---
title: 记一次服务启动时失败的问题
date: 2019-06-03 22:23:47
categories: 生产故障
tags: [Redis, Cache]
---

> 记录一次线上account服务第一次启动时，有效访问阻塞，启动失败的问题。<!--more-->

#### 起因

&emsp;&emsp;在今天修改了account服务之后，发布到生产之后，发现携程的查询请求全部报错，阻塞在了account服务中。我们就对问题进行了排查。首先，预估有这个原因：

1. Redis挂了，线程池没有起到作用。

2. 在重新发布时，供应商信息没有存放在缓存中。

   

#### 问题排查

&emsp;&emsp;检查代码，发现代码没有问题，查看redis的线程池设计。我们用的redis线程池是用的commons-pool。查看获取对象的方法：

```
public T borrowObject() throws Exception {
        long starttime = System.currentTimeMillis();
        GenericObjectPool.Latch<T> latch = new GenericObjectPool.Latch();
        byte whenExhaustedAction;
        long maxWait;
        synchronized(this) {
            whenExhaustedAction = this._whenExhaustedAction;
            maxWait = this._maxWait;
            this._allocationQueue.add(latch);
        }

        this.allocate();

        while(true) {
        // 添加同步锁
            synchronized(this) {
                this.assertOpen();
            }

            if (latch.getPair() == null && !latch.mayCreate()) {
                switch(whenExhaustedAction) {
                case 0:
                		// 同步锁
                    synchronized(this) {
                        if (latch.getPair() == null && !latch.mayCreate()) {
                            this._allocationQueue.remove(latch);
                            throw new NoSuchElementException("Pool exhausted");
                        }
                        break;
                    }
                case 1:
                    try {
                        synchronized(latch) {
                            if (latch.getPair() != null || latch.mayCreate()) {
                                break;
                            }
                            
                            // 等待时间
                            if (maxWait <= 0L) {
                                latch.wait();
                            } else {
                                long elapsed = System.currentTimeMillis() - starttime;
                                long waitTime = maxWait - elapsed;
                                if (waitTime > 0L) {
                                    latch.wait(waitTime);
                                }
                            }
                        }

                        if (this.isClosed()) {
                            throw new IllegalStateException("Pool closed");
                        }
                    } catch (InterruptedException var51) {
            ...
    }
```

而我们的问题就出现在了这个等待时间的问题上。

&emsp;&emsp;当account服务重新启动时，redis会从线程池中获取redis连接。而在第一次启动时，<font color='red'>redisPool定义的最大资源数、最小空闲资源数时，不会真的把Jedis连接放到连接池里。第一次使用时，没有资源使用，会先创建一个新的链接，然后放到池子里，会有一定的时间开销。</font>而在这一段时间里，携程等OTA的搜索量非常大，要在account服务查询大量的供应商信息，造成堵塞。所以在account服务启动前3分钟时，会造成携程的搜索失败率很高，有时直接account服务启动失败。所以我们考虑<font color='red'>要在account服务启动时，先要为redisPool进行预热。</font>

&emsp;&emsp;携程的机票搜索一直调用account服务，但是只是在account服务查询供应商的信息，而这些供应商信息一般都是固定的。所以最后我们决定使用本地缓存。在每一个对外查询接口进行添加本地缓存，同时，为了防止突然修改供应商信息，需要刷新本地缓存，我们修改供应商信息时会通知给zookeeper，一但本地缓存的程序发现供应商信息修改了，会自动刷新本地缓存。