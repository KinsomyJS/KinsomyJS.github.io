
### 1.常用并发集合

* ConcurrentHashMap:高效的并发HashMap。线程安全的HashMap。
* CopyOnWriteArrayList：适用在读多写少的场景，性能优于Vector。
* ConcurrentLinkedQueue：高效的并发队列，使用链表实现，线程安全的LinkedList。
* BlockingQueue：一个接口，JDK内部通过链表、数组等方式实现了这个接口。表示阻塞队列，适用于作为数据共享的通道。
* ConcurrentSkipListMap：跳表的实现，使用跳表的数据结构进行快速查找。

### 2.HashMap变为线程安全
Collections类下有一个`Collections.synchronizedMap(Map<K, V> var0)`方法，将HashMap转换成一个`Collections.SynchronizedMap`。
```java
public static <K, V> Map<K, V> synchronizedMap(Map<K, V> var0) {
        return new Collections.SynchronizedMap(var0);
    }

private static class SynchronizedMap<K, V> implements Map<K, V>, Serializable {
        private static final long serialVersionUID = 1978198479659022715L;
        private final Map<K, V> m;
        final Object mutex;
        private transient Set<K> keySet;
        private transient Set<Entry<K, V>> entrySet;
        private transient Collection<V> values;

        SynchronizedMap(Map<K, V> var1) {
            this.m = (Map)Objects.requireNonNull(var1);
            this.mutex = this;
        }

        SynchronizedMap(Map<K, V> var1, Object var2) {
            this.m = var1;
            this.mutex = var2;
        }

        public int size() {
            Object var1 = this.mutex;
            synchronized(this.mutex) {
                return this.m.size();
            }
        }

        public boolean isEmpty() {
            Object var1 = this.mutex;
            synchronized(this.mutex) {
                return this.m.isEmpty();
            }
        }
        ...省略
}
```
再看看SynchronizedMap的代码，它是使用代理模式，将构造传入的HashMap对象作为Map成员对象m的具体实现，同时使用`synchronized(this.mutex)`对每次操作加锁，保证了线程安全。

这是一个较为初级的线程安全HashMap的实现，因为每次操作都要获得mutex的锁，性能低下。

