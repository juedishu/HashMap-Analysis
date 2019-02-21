# HashMap原理分析
数组存储区间连续，查找方便，但是插入和删除效率低下；链表存储区间离散，插入删除方便，但是查找困难。大家肯定会问，有没有一种结构，既能做到查找便捷，又能做到插入删除方便呢？答案就是我们今天要跟大家说的主角：哈希表。 </br>  我们先来看一下哈希表的百度定义</br>
 散列表（Hash table，也叫哈希表），是根据关键码值(Keyvalue)而直接进行访问的数据结构。也就是说，它通过把关键码值映射到表中一个位置来访问记录，以加快查找的速度。这个映射函数叫做散列函数，存放记录的数组叫做散列表。 </br> 给定表M，存在函数f(key)，对任意给定的关键字值key，代入函数后若能得到包含该关键字的记录在表中的地址，则称表M为哈希(Hash）表，函数f(key)为哈希(Hash)函数。</br>
 
左边是数组，右边是链表，感觉十分像用数组把链表串起来，在一个长度为16的数组中，每个元素存储的是一个链表的头结点。接下来我们一起来分析一下HashMap的源码实现。</br>
HashMap的继承关系</br>
     
通过HashMap的继承关系，我们可以得知HashMap继承自抽象类AbstractMap，该Map又实现了Map接口，我们下来看一下Map接口包含哪些方法。</br>
可以看出包含了我们常用的HashMap中的一些方法，接着我们来看HashMap的父类AbstractMap。
`  public abstract class AbstractMap<K, V> implements Map<K, V> {</br>
    // 用懒加载的方式定义了Set集合类型的键，表明HashMap键是不能重复的
    Set<K> keySet;`
    
    // 用懒加载的方式定义了Collection集合类型的值，表明HashMap值是可以重复的
    Collection<V> valuesCollection;
    ……
    }
     这里重点要看明白一开始定义的键和值，键是不能重复的，值可以重复。 </br> 然后定义了两个静态的实体类</br>
     // 维护键和值的 Entrystatic class AbstractMap.SimpleEntry<K,V>  // 维护不可变的键和值的 Entrystatic class</br> AbstractMap.SimpleImmutableEntry<K,V>           </br>
 继续下面实现了接口Map中的方法：clear、put、get、equals、size、hashCode等等。</br>
HashMap源码解析</br>
现在我们开始分析HashMap的源码，走起┏ (゜ω゜)=☞</br>
元素定义</br>
 `    
     private static final int MINIMUM_CAPACITY = 4;// HashMap最小容量为4
     private static final int MAXIMUM_CAPACITY = 1 << 30;// HashMap最大容量1073741824，往右移除2，往左移是乘2
     // 实体数组
     private static final Entry[] EMPTY_TABLE
     = new HashMapEntry[MINIMUM_CAPACITY >>> 1];// 一个空的键值对实体数组最小容量是2
      static final float DEFAULT_LOAD_FACTOR = .75F;// 默认的容量扩展因子是0.75
     transient HashMapEntry<K, V>[] table;// 键值对的数组     
     transient HashMapEntry<K, V> entryForNullKey;// 没有键的键值对     
     transient int size;// 非空元素长度
     transient int modCount;// 计数器
     private transient int threshold;// 容量因子的极限
     // Views - lazily initialized 父类继承下来的
     private transient Set<K> keySet;
     private transient Set<Entry<K, V>> entrySet;
     private transient Collection<V> values;
     构造方法
     /**
     * Constructs a new empty {@code HashMap} instance.
     */
     @SuppressWarnings("unchecked")
     public HashMap() {
     table = (HashMapEntry<K, V>[]) EMPTY_TABLE;
     threshold = -1; // Forces first put invocation to replace EMPTY_TABLE
     }
     /**
     * 指定初始容量的构造方法
     */
     public HashMap(int capacity) {
     if (capacity < 0) {
     throw new IllegalArgumentException("Capacity: " + capacity);
     }
     if (capacity == 0) {
     @SuppressWarnings("unchecked")
     HashMapEntry<K, V>[] tab = (HashMapEntry<K, V>[]) EMPTY_TABLE;
     table = tab;
     threshold = -1; // Forces first put() to replace EMPTY_TABLE
     return;
     }
     if (capacity < MINIMUM_CAPACITY) {
     capacity = MINIMUM_CAPACITY;
     } else if (capacity > MAXIMUM_CAPACITY) {
     capacity = MAXIMUM_CAPACITY;
     } else {
     capacity = Collections.roundUpToPowerOfTwo(capacity);
     }
     makeTable(capacity);
     }
     /**
     * 指定初始容量和扩展因子的构造方法
     */
     public HashMap(int capacity, float loadFactor) {
     this(capacity);
     if (loadFactor <= 0 || Float.isNaN(loadFactor)) {
     throw new IllegalArgumentException("Load factor: " + loadFactor);
     }
     /*
     * Note that this implementation ignores loadFactor; it always uses
     * a load factor of 3/4. This simplifies the code and generally
     * improves performance.
     */
     }
     /**
     * 构造一个映射关系与指定 Map 相同的新 HashMap。所创建的 HashMap 具有默认加载因子 (0.75) 和足  以容纳指定 Map 中映射关系的初始容量。 
     */
     public HashMap(Map<? extends K, ? extends V> map) {
     this(capacityForInitSize(map.size()));
     constructorPutAll(map);
     }
     put方法
     /**
     * Maps the specified key to the specified value.
     * @param key the key.
     * @param value the value.
     * @return the value of any previous mapping with the specified key or
     *         {@code null} if there was no such mapping.
     */
     @Override 
     public V put(K key, V value) {
     if (key == null) {// HashMap的key是可以为空的，如果为空，放一个NullKey
     return putValueForNullKey(value);
     }
     // 定义一个int型的hash，将key的hashCode计算后再次得到Hash值赋予它，即二次哈希
     int hash = Collections.secondaryHash(key);
     HashMapEntry<K, V>[] tab = table;// 存储键值对的数组
     // index是下标，tab数组长度-1即数组下标最大值，接着做&运算得到下标最大的长度，避免溢出
     int index = hash & (tab.length - 1);
     // 遍历索引下面的整个链表，tab[index]整个链表的头结点，如果index索引处的Entry不为 null，通过循环不断遍历e元素的下一个元素  
     for (HashMapEntry<K, V> e = tab[index]; e != null; e = e.next) {
     // 如果指定key与需要放入的key两个键相同，进行覆盖，新的覆盖老的
     if (e.hash == hash && key.equals(e.key)) {
     preModify(e);
     V oldValue = e.value;
     e.value = value;
     return oldValue;
     }
     }
     // 如果index索引处的Entry为null，表明此处还没有 Entry 
     modCount++;// 计数器++
     if (size++ > threshold) {// 尺寸++大于容量因子的极限，则扩容
     tab = doubleCapacity();// 容量扩大两倍，HashMap容量最小是4，然后两倍两倍的扩容
     index = hash & (tab.length - 1);// 重新计算一遍
     }
     addNewEntry(key, value, hash, index);// 把新的键值对添加进来
     return null;
     }`
     Hash相同，Key不一定相同，如果Key相同，Hash值一定相同。</br>
     总结put方法的基本过程如下： </br> （1）对key的hashcode进行二次hash计算，获取应该保存到数组中的index。 </br> （2）判断index所指向的数组元素是否为空，如果为空则直接插入。 </br> （3）如果不为空，则依次查找e中next所指定的元素，判读key是否相等，如果相等，则替换旧的值，并返回值。</br>  （4）跳出循环，判断容量是否超出，如果超出进行扩容 </br> （5）执行addNewEntry()添加方法；</br>
     get方法</br>
     `
     /**
     * Returns the value of the mapping with the specified key.
     * @param key the key.
     * @return the value of the mapping with the specified key, or {@code null}
     *         if no mapping for the specified key is found.
     */
     public V get(Object key) {
     if (key == null) {// 如果key为空，把entryForNullKey赋值给e
     HashMapEntry<K, V> e = entryForNullKey;
     return e == null ? null : e.value;
     }
     // 根据key的hashCode值计算它的hash码  
     int hash = Collections.secondaryHash(key);
     HashMapEntry<K, V>[] tab = table;
     // 直接取出tab数组中指定索引处的值
     for (HashMapEntry<K, V> e = tab[hash & (tab.length - 1)];
     e != null; e = e.next) {
     //拿到每一个键值对中的key
     K eKey = e.key;
     // 去比较，如果两个key相等 或者 内存地址相等并且两个哈希值相等，则找到并返回值
     if (eKey == key || (e.hash == hash && key.equals(eKey))) {
     return e.value;
     }
     }
     return null;
     }
     remove方法
     /**
     * Removes the mapping with the specified key from this map.
     * @param key the key of the mapping to remove.
     * @return the value of the removed mapping or {@code null} if no mapping
     *         for the specified key was found.
     */
     @Override public V remove(Object key) {
     if (key == null) {// 如果key等于空，则移除空键值对
     return removeNullKey();
     }
     int hash = Collections.secondaryHash(key);
     HashMapEntry<K, V>[] tab = table;
     int index = hash & (tab.length - 1);
     // 遍历全部结点
     for (HashMapEntry<K, V> e = tab[index], prev = null;
     e != null; prev = e, e = e.next) {
     // 找到结点，如果Hash相等，并且key相等，找到了我们要移除的结点
     if (e.hash == hash && key.equals(e.key)) {
     if (prev == null) {// 遍历到最后结束时找到要删除的结点
     tab[index] = e.next;// 头结点从第二个结点开始算起
     } else {// 在中间任意地方找到了我们要删除的元素
     prev.next = e.next;// 这里使用了链表删除元素的套路
     }
     modCount++;
     size--;
     postRemove(e);
     return e.value;
     }
     }
     return null;
     }`
