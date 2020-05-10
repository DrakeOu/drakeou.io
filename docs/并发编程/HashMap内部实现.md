## HashMap内部实现

```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

HashMap中，put方法分析HashMap如果工作的。

- 其中有两种结构体，

  

  ```
  Node(int hash, K key, V value, Node<K,V> next) {
      this.hash = hash;
      this.key = key;
      this.value = value;
      this.next = next;
  }
  ```

```
TreeNode(int hash, K key, V val, Node<K,V> next) {
    super(hash, key, val, next);
}
```

链表中有两种节点：普通的节点Node，红黑树的头节点TreeNode

put过程：

1. 如果Map没有初始化，则resize()
2. 如果hash值计算的位置为空，则new 一个新的Node插入到尾端
3. （发生了hash冲突，准备解决冲突）
   	1. 如果是key值重复，则进行替换
    	2. 如果冲突节点已经是TreeNode，按红黑树的方式进行节点插入（也可能Key重复，但是这里不管了）
    	3. （冲突的是一个普通Node）
        - 使用顺序的再散列，向后查找到为空的位置进行插入
        - 向后散列的时候检查key是否已经重复
        - 再散列的时候进行计数，最后插入时如果计数器大于8（再散列了7次，hash冲突为8），将表进行红黑树的转换。