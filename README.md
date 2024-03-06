前言：小弟学习一些东西的时候，遗忘曲线在我身上体现的十分的淋漓尽致。这次回过头看CAS算法的时候，网上的文章有很多，但是我却偶尔回忆不起来作者表述的东西是什么含义，今天我再一次梳理了一下CAS算法的底层原理，再自己帮助自己记录的时候，方便自己回想起来，另外要是能帮到各位同学的话，那就再好不过了。

# 什么是CAS算法？
如果一个线程失败或挂起不会导致其他线程也失败或挂起，那么这种算法就被称为**非阻塞算法**，也叫**无锁算法**。而CAS算法就是对非阻塞算法的一种实现。

# 什么是非阻塞算法？
要说明什么是非阻塞算法，我们就要知道什么是阻塞算法。我们通常在Java代码里面使用的**synchronized**关键字实现的算法，就是一种阻塞式。不过可能有同学会说，在java1.6之后进行了改进，有偏向锁的状态，这个时候偏向锁实际上式一种无锁状态，它仅仅是记录了当前对象的所处的线程ID。不过我们这里讲的是整个**synchronized**的实现，当出现线程竞争的时候，通过锁的升级，就会演变成轻量级锁和重量级锁，这个时候就是阻塞方式的。说了这么多，还没说清楚，什么是阻塞式的？所谓阻塞式，就是当某个资源被某一个线程占有的时候，另一个线程想要访问这个资源的时候，必须等当前占有资源的这个线程释放对资源的占有的时候，这个时候新的线程才能获取到该资源的使用权。如下：


<p align=center><img src="https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/beada54b42f34fe396f2e1182558e4e8~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=435&h=281&s=16319&e=png&b=fefefe" alt="image.png"  /></p>

## 那么CAS算法是如何实现非阻塞式的？
CAS算法通过比较旧的预期值P，内存值V，和新的待写入的值R的比较，来实现非阻塞式。这里我需要先说明一下内存当中的以上三个值的模型，这样大家好明白一点。如下图：

<p align=center><img src="https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/41b7ede7bc824817a13d20e4df26c808~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=451&h=391&s=15709&e=png&b=ffffff" alt="image.png"  /></p>

每一个线程当获取到该内存当中的值的时候，都会在自己的线程栈中保留一个副本。因此，当线程A和线程B同时去读取内存当中的值Value的时候，都会在自己的线程保留对象的副本，Value_A和Value_B，只有先清楚这个，才能说明白CAS算法里面的预期值，内存值和新的待写入的值的含义。
还是以上面这个图为例，假设，内存当中的value初始值为1，那么一开始线程A和线程B保存的副本都是1，这个时候三者保存的值都是一样的。但是这个时候，线程A修改了自己线程的Value_A的值为2，为了保证所有的值都是一样的，需要以下几个步骤。
1. 修改自己的Value_A = 2
2. 将Value_A的值同步到内存Value中，使Value=2
3. 线程B主动再读取一次内存的值或者内存主动通知线程B，使Value_B =2。

只有当上面这个几个步骤都生效的时候，才能保证三者的值都是实时同步的。

明白了上面的这个步骤，我们在看什么是预期值P，内存值V，和新的待写入的值R。
预期值P：就是线程里面保存的副本，Value_A和Value_B
内存值V：就是内存当中的Value
新的待写入的值R：需要更新的新值，这里我们以线程A举例子，就是New_Value_A。

# Compare-And-Swap
CAS算法的核心就是Compare-And-Swap。先比较，再交换。什么意思，假设我New_Value_A = 2,这个时候，我想要更新内存当中的值，我就会先比较预期值P（Value_A）和内存值Value（V）是否相等，如果相等，我们就认为这个时候，我们本地的值是最新的，没有其他线程进行更改，我们可以进行更新（这里其实并不完全正确，有一个ABA的问题，我们后面再说）。否则，我们认为，当前线程里的值并不是最新的，不做任何操作。

# ABA问题
其实上面的CAS算法有一个问题，就是ABA。什么是ABA问题？以上述的模型作例子，我们都知道，我们只有在满足P和V值相同的时候，我们才会将线程A里面的N值更新到内存当中，但是如果线程B在线程A更新N值之前，他已经更新了2次内存的值，先将Value设置成2，再将Value设置1，这个时候，对于线程A，他是不可预知的，他仍然认为本地的值是最新的，然而内存当中的值已经被更改过2次了。

# 实际使用
其实在日常的使用当中，我们并不会直接使用cas方法，因为它是通过**sun.misc.Unsaf**去实现的。
```
    // CAS 操作的核心对象
    private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();

    public final int getAndIncrement() {
        return U.getAndAddInt(this, VALUE, 1);
    }

    /**
     * Atomically decrements by one the current value.
     *
     * @return the previous value
     */
    public final int getAndDecrement() {
        return U.getAndAddInt(this, VALUE, -1);
    }

    /**
     * Atomically adds the given value to the current value.
     *
     * @param delta the value to add
     * @return the previous value
     */
    public final int getAndAdd(int delta) {
        return U.getAndAddInt(this, VALUE, delta);
    }

    /**
     * Atomically increments by one the current value.
     *
     * @return the updated value
     */
    public final int incrementAndGet() {
        return U.getAndAddInt(this, VALUE, 1) + 1;
    }

    /**
     * Atomically decrements by one the current value.
     *
     * @return the updated value
     */
    public final int decrementAndGet() {
        return U.getAndAddInt(this, VALUE, -1) - 1;
    }

    /**
     * Atomically adds the given value to the current value.
     *
     * @param delta the value to add
     * @return the updated value
     */
    public final int addAndGet(int delta) {
        return U.getAndAddInt(this, VALUE, delta) + delta;
    }
```

但是java提供了我们很多可供间接使用的类
```
java.util.concurrent.atomic
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f74bed331624499fbe2a7d85fa551c44~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=456&h=664&s=122086&e=png&b=372c21)

这里具体的使用方式，就不再细说了，各位同学可以自己试着操作。
