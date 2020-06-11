

## 1 定义

`ThreadLocal`为线程提供局部变量，每个线程都可以通过<code style='color: #ff502c;font-size: .87em;background-color: #fff5f5;'>set()</code>和`get()`来对这个局部变量进行操作，但不会和其他线程的局部变量进行冲突，**实现了线程的数据隔离**～。

- 每个Thread维护着一个ThreadLocalMap的引用
- ThreadLocalMap是ThreadLocal的内部类，用Entry来进行存储
- 调用ThreadLocal的set()方法时，实际上就是往ThreadLocalMap设置值，key是ThreadLocal对象，值是传递进来的对象
- 调用ThreadLocal的get()方法时，实际上就是往ThreadLocalMap获取值，key是ThreadLocal对象
- **ThreadLocal本身并不存储值**，它只是**作为一个key来让线程从ThreadLocalMap获取value**。

正因为这个原理，所以ThreadLocal能够实现“数据隔离”，获取当前线程的局部变量值，不受其他线程影响～

## 2 作用

- 管理JDCB的Connection链接，ThreadLocal能够实现**当前线程的操作都是用同一个Connection，保证了事务！**

- 避免一些参数传递

## 3 实现原理

```java
public void set(T value) {
    // 得到当前线程对象
    Thread t = Thread.currentThread();
    // 这里获取ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    // 如果map存在，则将当前线程对象t作为key，要存储的对象作为value存到map里面去
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

**ThreadLocalMap是ThreadLocal的一个内部类。用Entry类来进行存储**，我们的**值都是存储到这个Map上的，key是当前ThreadLocal对象**！如果该Map不存在，则初始化一个，如果该Map存在，则**从Thread中获取**！Thread维护了ThreadLocalMap变量。

**ThreadLocalMap是在ThreadLocal中使用内部类来编写的，但对象的引用是在Thread中**！**Thread为每个线程维护了ThreadLocalMap这么一个Map，而ThreadLocalMap的key是LocalThread对象本身，value则是要存储的对象**

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

## 4 避免内存泄露

ThreadLocal内存泄漏的根源是：**由于ThreadLocalMap的生命周期跟Thread一样长，如果没有手动删除对应key就会导致内存泄漏，而不是因为弱引用**。想要避免内存泄露就要**手动remove()掉**！

![](..\cache\img\162896ab1a1d1e2e.jpg)



