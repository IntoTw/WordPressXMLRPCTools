---
title: '分布式系统：分布式锁'
date: 2020-10-14
lastmod: 2020-10-14
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["分布式锁","分布式"]
categories: ["技术"]
lightgallery: true
---

## 什么是分布式锁
锁的含义，一般就是为了独占资源，防止并发冲突，一般锁的实现，都依赖于计算机资源，如CPU，内存等，但是在跨系统时，各系统独立，如果需要锁，就需要一种分布式锁的实现方案。让各系统之间对相同资源的并发操作不会出现并发问题。
## 分布式锁的应用场景
虽然理论上，分布式锁适用于任何多应用需要独占资源或者要进行串行操作的场景，但是一般业务中，从我经验来看，主要是在下个场景使用的：
1. 业务幂等以及防重，以创建订单举例，同时2个创建订单请求打到2台机器上，如果没有资源独占，就会进行2次创建，结果就是重复扣费，此时要么使用数据库的串行化事务来保证不出错，但吞吐低到令人不可接受的程度，要么就使用分布式锁，针对**业务:业务主键id**这种形式，去获取锁，在具体某个业务某个订单号的维度去进行业务幂等。
2. 定时任务调度独占，在定时任务场景，很多定时任务触发时，是需要对一定范围的数据或一定资源进行独占的，此时会使用分布式锁，进行定时任务执行的独占或者资源的锁定。

## 分布式锁的实现
分布式锁的主要实现，目前流行3种方案：数据库，Redis，Zookeeper。数据库和Zookeeper因为自身原因，吞吐低，可用性较低，并且性能也不如缓存，所以目前主要都是使用Redis作为分布式锁。

### Redis分布式锁最基础的实现
Redis分布式锁的实现主要依赖setnx()以及expire()操作，setnx操作类似于Java中ConcurrentHashMap的putIfAbsent操作，即如果是空，则更新，并返回更新成功与否。expire()操作即过期操作，给用来加锁的那个Key值设置一个自动过期时间，防止因为程序原因，手动释放锁时失败了会导致死锁的问题。但是这个东西不建议自己实现，因为要考虑的情况非常多，比如：

1. 如果执行时间过长，此时已经过了自动过期时间，相当于锁释放掉了，那么此时还是会出现2个线程或2个应用同时认为自己持有了该锁，导致并发问题。
2. 如果获取锁成功，但是过期时间设置失败，此时假如再在手动释放锁时出现意外，那么又会出现死锁的情况。
3. 手动释放锁时，需要考虑各种情况，如该锁是否当前线程持有？该锁是否还存在？
4. redis锁是否要有可重入性？

### 通过Redisson来进行Redis锁的实现
Redisson应该是最流行的Java-RedisClient框架了，其中封装了很多Redis命令，包括使用Lua脚本对Redis锁有较好的实现。基本使用方法如下：
```java
    RLock lock=null;
    try {
        //1
        lock = redissonClient.getLock(id + "_" + suffix);
        //2
        if(lock.tryLock(1,1, TimeUnit.MILLISECONDS)) {
            //do something
        }
    }catch (Exception e){
        log.error("",e);
    }finally {
        //3
        if(lock!=null && lock.isLocked() && lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
```
1. getLock获取的就是redis键值，一般为了做防重，都是使用业务+主键id，这里的getLock只是声明，并未去获取锁。
2. tryLock即尝试去获取锁，支持最多3个参数，等待时间，自动释放时间，时间单位。
3. 这里也是上面提到的，也算是Redisson的一个小坑吧，**Redisson的unlock方法在解锁失败时会抛出异常**，如果你有事务，事务马上就回滚了，所以这里一定要加一堆条件来保证unlock尽量不会出异常，最理想的还是封装一个方法，里面是tryCatch块，专门用来解锁，防止异常导致了外部业务回滚。


### Redisson来进行Redis分布式锁的坑
**上一标题中第二点的自动释放时间有一个坑，就是自动释放时间，这个时间在实际业务中很难评估，有时业务就是会执行超时导致锁没有锁住，所以可以使用Redis的Watch Dog机制，即该参数使用-1或不填，则redis会产生一个30s的锁，并且每10s会自动对该锁进行续约，直到手动释放该锁或服务宕机导致续约的守护线程无法正常续约**
```java
//定时任务自动续约，每个leaseTime的3分之一时间执行一次，这个任务通过future的回调机制，在成功获取redis锁之后开始执行。
private void scheduleExpirationRenewal(final long threadId) {
        if (!expirationRenewalMap.containsKey(this.getEntryName())) {
            Timeout task = this.commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
                public void run(Timeout timeout) throws Exception {
                    RFuture<Boolean> future = RedissonLock.this.commandExecutor.evalWriteAsync(RedissonLock.this.getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN, "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then redis.call('pexpire', KEYS[1], ARGV[1]); return 1; end; return 0;", Collections.singletonList(RedissonLock.this.getName()), new Object[]{RedissonLock.this.internalLockLeaseTime, RedissonLock.this.getLockName(threadId)});
                    future.addListener(new FutureListener<Boolean>() {
                        public void operationComplete(Future<Boolean> future) throws Exception {
                            RedissonLock.expirationRenewalMap.remove(RedissonLock.this.getEntryName());
                            if (!future.isSuccess()) {
                                RedissonLock.log.error("Can't update lock " + RedissonLock.this.getName() + " expiration", future.cause());
                            } else {
                                if ((Boolean)future.getNow()) {
                                    RedissonLock.this.scheduleExpirationRenewal(threadId);
                                }

                            }
                        }
                    });
                }
            }, this.internalLockLeaseTime / 3L, TimeUnit.MILLISECONDS);
            if (expirationRenewalMap.putIfAbsent(this.getEntryName(), task) != null) {
                task.cancel();
            }

        }
    }
```
那么就带来了第二个问题，之前生产排查发现一个事务不该回滚的回滚了，回滚的原因就是redis锁被其他线程获取到了，在某线程在调用unlock解锁时，报了异常发现该锁不是被该线程持有的，那么为什么有了WatchDog机制，还会这样呢？

其实想想也很简单，无非就是watchDog机制失效了呗，WatchDog机制也是需要使用到cpu资源的，在系统负载较高的情况下，较低的leaseTime就会很容易出现续约逻辑来不及执行的情况，并且一旦续约逻辑失效，该锁就会自动被释放掉，其他的线程就可以获取到了。

不过目前看来这种情况发生的概率非常低，一年内也就出现了几次，所以后续我们只是针对这个业务手动设置了一个较长的leaseTime，并且在unlock前进行检查，防止异常抛到业务层。


## 分布式锁的思考
Redis分布式锁不是万能的，甚至来说他的一致性并不高，不过一般业务出于吞吐和性能，以及HA的考虑，才使用分布式锁，如果要一致性要求非常高，建议使用数据库锁和zookeeper锁，这两个都是CP的。

如果其中有事务，一定要在事务执行完成后，再释放Redis锁，即锁囊括的范围要适当偏大，但是锁的粒度最好细到具体某个用户的某个业务操作。

Redis锁可以根据具体的业务场景你对防重以及性能的容忍度，来评估等待时间和超时时间等。
如果实在因为各种原因没有锁住的情况偶尔发生，可以在业务上对重复的业务进行自动撤销，如我们就做了对重复支付订单的自动退款功能。
