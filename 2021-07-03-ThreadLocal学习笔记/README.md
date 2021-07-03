`Object object = new Object();`

![](1.png)

`object = null;`

![](2.png)

ThreadLocal 与 Thread 的关系

![](3.png)

ThreadLocal 线程隔离

![](4.png)

ThreadLocal 引用关系

![](5.png)

ThreadLocal 内存泄漏

![](6.png)

只要线程依然存活，就存在如下引用关系

```
thread ref -> Thread -> ThreadLocalMap -> Entry -> value -> Object
```

因此，Object 的内存空间不会被回收，从而导致内存泄漏。
