## ConcurrentHashMap 实现解析

```
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

实现了并发的线程安全，所以每一步都考虑有多个线程同时操作的情况

**实现逻辑**

要点：

- 代码主体是一个不退出的循环（确保节点能最终有位置插入）

1. 如果Map还没有初始化，初始化（如果多个线程同时进入这个条件：初始化操作是CAS保证线程安全）```initTable()```

2. 如果Hash得到的地址值为空，CAS进行尝试插入，成功就退出循环，失败就进行下一次循环（显然下一次就不为空了）

   ```java
   casTabAt(tab, i, null,
                new Node<K,V>(hash, key, value, null))
   ```

3. 如果要插入的节点在进行迁移（链表向红黑树转换？），则帮助进行迁移（CAS保证安全）

   ```java
   tab = helpTransfer(tab, f);
   ```

4. （以上情况都不是，表示发生了Hash冲突）

          1. （以下方法在同步块中```synchronized (f)```）,且是以**操作的节点**进行加锁。
          2. 如果是普通Node
             - key值重复进行覆盖
             - 顺序再散列解决冲突，找到为空的位置插入
             - 和HashMap一样，记录冲突个数（用于判断是否转为红黑树）
       3. 如果是TreeBin（和HasHMap中TreeNode一个概念）
          - 则按照红黑树的方式插入

5. 最后判断冲突个数，如需要进行红黑树的转换```treeifyBin(tab, i)```

   - 如果表中节点数量过少，则执行扩容
   - 否则转为红黑树，（使用了syn同步）

