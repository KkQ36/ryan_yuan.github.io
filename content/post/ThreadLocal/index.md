---      
author: Ryan Yuan      
title: ThreadLocal、InheritableThreadLocal 源码解析      
date: 2024-11-05      
image: image.png          
---      
      
      
> `ThreadLocal` ，从字面含义上来看就是："线程本地"，这形象的说明了它的用处： `ThreadLocal` 正是 Java 提供的一种线程本地存储机制，可以利用该就只将数据还存在某个线程的内部，该线程可以在任意的时刻、任意的方法中去获取缓存的数据。      
### 基本介绍      
`ThreadLocal` 共有两种类型， `ThreadLocal` 本身和他的子类 `InheritableThreadLocal`。      
**普通类型的 `ThreadLocal`**： 这种 `ThreadLocal` 可以存储任意类型的数据，例如：      
```java      
ThreadLocal<String> threadLocalString = new ThreadLocal<>();      
ThreadLocal<Integer> threadLocalInteger = new ThreadLocal<>()      
```      
普通类型的 `ThreadLocal` 可以用于在每个线程中存储不同类型的数据，并且每个线程访问该`ThreadLocal` 对象时，都可以获取到其线程私有的变量副本。      
`InheritableThreadLocal` 类型的 `ThreadLocal`： `InheritableThreadLocal` 是 `ThreadLocal` 的一个子类；它允许子线程访问父线程设置的本地变量。      
具体原理是：当子线程创建时，它会继承父线程中 `InheritableThreadLocal` 的值。例如：      
```java      
InheritableThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<>();      
```      
`ThreadLocal` 底层是通过 `ThreadLocalMap` 实现的，每个 `Thread` 对象（注意不是 `ThreadLocal` 对象，这些数据都是 `Thread` 对象来引用的）中都有一个 `ThreadLocalMap`，Map 中的 Key 为 `ThreadLocal` 对象，Map 的 value 为缓存需要的值。      
```java      
	/*      
	ThreadLocal values pertaining to this thread. This map is maintained      
	 * by the ThreadLocal class.      
	 */      
	ThreadLocal.ThreadLocalMap threadLocals = null;      
```      
通过下面的图可以很清晰的看出线程和 `ThreadLocal`  的对应关系      
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/172fc40237d9481da0c7980405436eb7.png)      
## ThreadLocal 篇      
### ThreadLocal 的 set 方法      
下面我们通过 debug 下面这段代码，来看看 `ThreadLocal` 的执行逻辑：      
```java      
public static void main(String[] args) {      
    threadLocal.set("Hello World!");      
    System.out.println(threadLocal.get());      
}      
```      
首先调用的是 `void ThreadLocal.set(T value)`      
```java      
public void set(T value) {      
    Thread t = Thread.currentThread();      
    ThreadLocalMap map = getMap(t);      
    if (map != null) {      
        map.set(this, value);      
    } else {      
        createMap(t, value);      
    }      
}      
```      
先通过 `native` 本地方法获取到当前请求的线程，然后将获得的线程作为参数传递给 `getMap(Thread t)` 方法，这个方法就是用来获取 `Thread` 对象的 `ThreadLocalMap` 属性：      
```java      
ThreadLocalMap getMap(Thread t) {      
    return t.threadLocals;      
}      
      
/* ThreadLocal values pertaining to this thread. This map is maintained      
 * by the ThreadLocal class. */      
ThreadLocal.ThreadLocalMap threadLocals = null;      
```      
`threadLocals` 的初始值为空，它正式的创建时机就是第一次调用 `ThreadLocal` 的 `set()` 方法的时候，所以可以看到上面还有一个 `createMap` 方法;      
如果是第二次添加，此时的 `map` 不为空，就直接通过调用 `map` 的 `set` 方法来添加一个新的键值对。      
这里要特别关注键值对的内容：键是 `this` 也就是 `ThreadLocal` 对象本身，而值就是上面传入的 `value`，从这里可以看出，`ThreadLocal` 是作为键值对的键存在的，真实的数据其实就是存储在线程对象 `Thread` 中的。      
      
---      
### 真正负责存储的数据结构-ThreadLocalMap      
既然谈到了 `ThreadLocalMap` 类，这里来详细的看看它是如何实现键值对的对应关系的；以及在 Entry 如何通过虚引用（WeakReference）来解决内存泄露的问题的？      
```java      
static class ThreadLocalMap {}      
```      
`ThreadLocalMap` 是 `ThreadLocal` 的一个静态内部类，它并没有继承任何的父类，而是在内部自己实现了一个 `Map` ：      
```java      
        /**      
         * The table, resized as necessary.      
         * table.length MUST always be a power of two.      
         */      
        private Entry[] table;      
              
        static class Entry extends WeakReference<ThreadLocal<?>> {      
            /** The value associated with this ThreadLocal. */      
            Object value;      
      
            Entry(ThreadLocal<?> k, Object v) {      
                super(k);      
                value = v;      
            }      
        }      
```      
真实的数据就是存储在 `Entry` 对象中，这个对象除了存储 `value` 对象以外，还继承了 `WeakReference` **虚引用**，用来引用 `ThreadLocal` 对象，可以通过 `Entry.get()` 方法来获取虚引用指向的对象，当出现调用 Map 的 `get` 方法出现了哈西冲突的时候，可以通过 `Entry.get().threadLocalHashCode` 来获取到哈希值，与当前传入方法的 `key` 做一个比较，就能确定当前索引下的对象是不是所需要的了。      
而当对象仅被虚引用所指向的时候，JVM 进行 GC 垃圾回收的时候，**会直接将其回收**， `ThreadLocal<?> t = new ThreadLocal<>()` 这样的指向方式是一种强应用，当这个引用失效的时候，也就是 `ThreadLocal` 对象**彻底无法抵达**的时候， 此时，`Entry` 对象中的虚引用并不会影响 GC 的回收。      
当 `ThreadLocal` 被回收的时候，`Entry` 的引用对象就变为了 `null`；`ThreadLocalMap` 有一个检查机制，会有一个名为 `cleanSomeSlots(int i, int n)` 的方法会在 either a new element is added, or another stale one has been expunged. 的时候执行，也就是当新元素添加或者过期元素被清除的时候；具体来说就是这两个方法：      
```java      
// 替换过期的 KEY，将指定键的新条目存储在过期条目位置，无论该键是否已有值      
private void replaceStaleEntry(ThreadLocal<?> key, Object value,        
                               int staleSlot)      
private void set(ThreadLocal<?> key, Object value)                                     
```      
这个方法的具体内容是这样的：      
```java      
private boolean cleanSomeSlots(int i, int n) {        
    boolean removed = false;        
    Entry[] tab = table;        
    int len = tab.length;        
    do {        
        i = nextIndex(i, len);        
        Entry e = tab[i];        
        // 检测 Entry 是否引用了 null      
        if (e != null && e.refersTo(null)) {        
            n = len;        
            removed = true;        
            i = expungeStaleEntry(i);        
        }        
    } while ( (n >>>= 1) != 0);  // 对数级别的时间复杂度      
    return removed;        
}      
```      
以大约 `O(logn)` 的时间复杂度去检测数组中元素是否存在空引用。      
      
既然要实现一个 Map，那就绕不开 `key`  的映射问题以及如何解决哈希冲突了， `ThreadLocalMap` 的映射方式是这样的:      
```java      
int i = key.threadLocalHashCode & (len-1);      
```      
直接将 `ThreadLocal` 对象的 `HashCode` 与数组的长度减一做一个按位与的操作，将其映射到数组中的一个位置。      
在 `ThreadLocalMap` 中，采用了**线性探测的方式**来解决哈希冲突，当出现了哈西冲突的时候，会调用 `nextIndex` 方法来获取下一个存放位置：      
```java      
private static int nextIndex(int i, int len) {      
    return ((i + 1 < len) ? i + 1 : 0);      
}      
```      
就是在原位置的基础上不断加一，当越界的时候就返回下标为 0 的位置继续寻找。      
      
---      
### ThreadLocal 的 get 方法      
看完了上面的 set 方法，猜一下也能知道 get 方法就是根据 `ThreadLocal` 对象来从**线程对象**的 `ThreadLocalMap` 中去取得对应的值：      
      
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
      
首先通过 `getMap()` 方法获取线程对象的 `ThreadLocalMap` ，然后通过 `getEntry()` 方法从 map 中获取对应的数据，也就是通过 `ThreadLocal` 的哈希值，去映射到`ThreadLocalMap` 的 Entry 数组的一个下标，来将对应的对象取出。      
### 在线程池中，ThreadLocal 为什么会引发内存问题？      
在线程池中使用 `ThreadLocal` 可能会引发内存问题，主要是因为**线程池中的线程是复用的**，而 `ThreadLocal` 变量的设计初衷是每个线程拥有一个独立的副本。当线程执行完任务后，如果没有正确地清理 `ThreadLocal` 中的数据，这些数据就会保留在内存中，从而引起内存泄漏。      
1. **生命周期差异**：`ThreadLocal`变量的生命周期理论上与创建它的线程相同，但在线程池中，**线程的生命周期远远长于单个任务的生命周期**。如果任务没有明确地清除`ThreadLocal` 变量的内容，那么这些变量会持续占用内存，直到线程池被销毁或JVM退出。      
2. **垃圾回收障碍**：当 `ThreadLocal` 变量引用了任务相关的数据，而这些数据又引用了其他对象时，即使任务已经完成，只要线程还存活，这些对象就不会被垃圾回收，因为从根节点（线程）到这些对象的引用路径依然存在。      
3. **静态`ThreadLocal`变量**：如果`ThreadLocal`是静态的，那么它在整个应用程序的生命周期内都会存在，即使线程已经不再使用它。这可能导致大量无用数据长期驻留内存。      
      
为了避免这些问题，可以采取以下措施：      
- **显式清理**：在任务结束时，显式地调用`ThreadLocal`的`remove()`方法来清理不再需要的数据。这可以断开`ThreadLocal`变量和任务数据之间的引用，使垃圾回收器能够回收不再使用的对象。      
- **合理设计**：尽量减少`ThreadLocal`变量的使用，只在确实需要线程隔离的状态时使用，并确保每个线程的任务结束后能够安全地释放这些变量所持有的资源。      
除了上面的措施，其实还有最后一种优化的方式，它适用于子线程需要继承父线程的 `ThreadLocal` 对象的时候：也就是我们前面提到的 `InheritableThreadLcoal` —— 可继承的 `ThreadLocal` 。      
## InheritableThreadLocal 篇      
> 使用`InheritableThreadLocal`并不能直接避免`ThreadLocalMap`的内存泄漏问题，但它提供了一个额外的机制，可以配合正确的使用和清理策略来帮助减少内存泄漏的风险，尤其是在线程池和多线程环境下。`ThreadLocal`和`InheritableThreadLocal`都会在线程的`ThreadLocalMap`中存储数据，而内存泄漏的风险主要来源于线程长时间存活时`ThreadLocal`变量未被清理，导致`ThreadLocalMap`中积累了大量不再需要的条目。在线程池中，线程往往复用多次，这种情况下如果没有适当的清理策略，`ThreadLocalMap`中的数据会逐渐累积，导致内存泄漏。      
      
继承关系：      
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a64bc971898a4481a379745441fcdc6f.png)      
### InheritableThreadLocal 相较于直接赋值有什么好处？      
`InheritableThreadLocal`本身的设计目的是允许子线程继承父线程的`ThreadLocal`值。它和`ThreadLocal`一样，需要在不再需要时显式调用`remove()`方法来清理。然而，`InheritableThreadLocal`提供了几个特性，可以帮助更好地管理线程间的数据传递和清理：      
1. **自动清理**：当一个新线程启动并继承了父线程的`InheritableThreadLocal`值时，这些值会被复制到新线程的`ThreadLocalMap`中，相当于是一个深层次的赋值，而不是仅仅存储原来的引用；这意味着父线程的`InheritableThreadLocal`值不再需要被子线程引用，从而减少了父线程的`ThreadLocalMap`的引用关系，有助于垃圾回收。      
2. **生命周期管理**：虽然`InheritableThreadLocal`本身不会自动清除值，但它的设计鼓励了更细粒度的线程生命周期管理。例如，在Web应用服务器中，每个HTTP请求可能在单独的线程中处理，使用`InheritableThreadLocal`可以确保每个请求的上下文数据在请求结束后被正确清理，从而避免了数据在`ThreadLocalMap`中的累积。      
3. **可配置的清理策略**：在某些框架和应用中，可以配置特定的清理逻辑，例如在Servlet容器中，可以在请求结束时自动调用`InheritableThreadLocal`的`remove()`方法，以确保每个请求的线程在重用前被清理干净。      
因此，虽然`InheritableThreadLocal`本身并不能直接防止`ThreadLocalMap`的内存泄漏，但结合正确的清理策略和生命周期管理，它可以作为整体解决方案的一部分，帮助减少内存泄漏的风险。在实践中，无论是使用`ThreadLocal`还是`InheritableThreadLocal`，都应该遵循良好的编码实践，如在适当的时机调用`remove()`方法，以确保不再需要的数据能够被及时释放。      
### InheritableThreadLocal 是如何实现继承的？      
`InheritableThreadLocal` 继承了父类 `ThreadLocal` 并且重写了它的 `getMap()` 和 `createMap()` 等方法      
```java      
    /**      
     * Get the map associated with a ThreadLocal.      
     *      
     * @param t the current thread      
     */      
    ThreadLocalMap getMap(Thread t) {      
       return t.inheritableThreadLocals;      
    }      
      
    /**      
     * Create the map associated with a ThreadLocal.      
     *      
     * @param t the current thread      
     * @param firstValue value for the initial entry of the table.      
     */      
    void createMap(Thread t, T firstValue) {      
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);      
    }      
          
	  ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;      
```      
当创建和获取的时候，都使用的是 `ThreadLocal` 中的 `inheritableThreadLocals` 属性，关于继承和使用的逻辑，也都是围绕这个属性去实现的，我们跟着下面的调试代码来具体的看一下：      
```java      
public class Main {      
      
    static InheritableThreadLocal<String> threadLocal = new InheritableThreadLocal<>();      
    public static void main(String[] args) {      
        Thread parentThread = new Thread(() -> {      
            threadLocal.set("Hello World");      
            Thread sonThread = new Thread(() -> {      
                System.out.println(threadLocal.get());      
            });      
            sonThread.start();      
        });      
        threadLocal.remove();      
        parentThread.start();      
    }      
}      
```      
当调用父线程调用 `threadLocal.set()` 方法的时候，调用的还是 `ThreadLocal` 中的 `set()` 方法：      
```java      
    public void set(T value) {      
        Thread t = Thread.currentThread();      
        ThreadLocalMap map = getMap(t);      
        if (map != null) {      
            map.set(this, value);      
        } else {      
            createMap(t, value);      
        }      
    }      
```      
只是此时通过 `getMap()` 获取到的不再是线程对象的 `threadLocals` 而是 `inheritableThreadLocals` 属性。      
通过上面的内容，我们知道了在线程对象中，`InheritableThreadLocal` 的值和 `ThreadLocal` 是**存储在不同属性中**的；当子线程创建的时候，就会去检测父线程中的`InheritableThreadLocal` 属性中是否存在值，如果存在会将其复制一份，并且保存在自己对应的属性中：      
下面展示的就是创建线程的方法，我们掠过和本节内容关系不大的部分：      
```java      
    private void init(ThreadGroup g, Runnable target, String name,      
                      long stackSize, AccessControlContext acc,      
                      boolean inheritThreadLocals) {      
        // 。。。。。。      
      
        Thread parent = currentThread();      
			        
			  // 。。。。。。      
      
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)      
            this.inheritableThreadLocals =      
                ThreadLocal.createInhritedMap(parent.inheritableThreadLocals);      
				      
			  // 。。。。。。      
    }      
          
    static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {      
        return new ThreadLocalMap(parentMap);      
    }      
```      
当判断父线程中的 `inheritableThreadLocals` 存在值的时候，会调用 `ThreadLocal` 中的 `createInhritedMap` 方法；这个方法是静态内部类 `ThreadLocalMap` 的 `private` 构造方法，它负责将参数中的 `ThreadLocalMap` 复制一份到当前创建的对象之中：      
```java      
        private ThreadLocalMap(ThreadLocalMap parentMap) {      
            Entry[] parentTable = parentMap.table;      
            int len = parentTable.length;      
            setThreshold(len);      
            table = new Entry[len];      
      
            for (int j = 0; j < len; j++) {      
                Entry e = parentTable[j];      
                if (e != null) {      
                    @SuppressWarnings("unchecked")      
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();      
                    if (key != null) {      
                        Object value = key.childValue(e.value);      
                        // 是复制一份，而不是直接存储      
                        Entry c = new Entry(key, value);      
                        int h = key.threadLocalHashCode & (len - 1);      
                        while (table[h] != null)      
                            h = nextIndex(h, len);      
                        table[h] = c;      
                        size++;      
                    }      
                }      
            }      
        }      
              
		// 控制子线程的继承内容        
		protected T childValue(T parentValue) {      
			return parentValue;      
		}      
```      
通过上面的 `Entry c = new Entry(key, value);` 就可以看出，是将 `Entry` 直接复制了一份，而不是单纯的指向。      
这样，子线程就继承了父亲线程的的 `InheritableThreadLocal` 中存储的内容。      