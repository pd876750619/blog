# java集合

## List

### ArrayList

```java
//内部使用对象数组持有对象
transient Object[] elementData; 

    private void grow(int minCapacity) {
      //扩容至指定最小容量
        // overflow-conscious code
        int oldCapacity = elementData.length;
      //为什么默认先扩1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
      //每次指定下标添加，都需要对下标后的对象做复制
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }}
		
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
          	//删除也是一样，需要复制
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }

    public void sort(Comparator<? super E> c) {
        final int expectedModCount = modCount;
        Arrays.sort((E[]) elementData, 0, size, c);
        if (modCount != expectedModCount) {
          //排序时会校验是否并发修改过
            throw new ConcurrentModificationException();
        }
        modCount++;
    }
```

## Set

### HashSet

```java
//底层用HashMap实现
private transient HashMap<E,Object> map;
//用于put到map中的常量
private static final Object PRESENT = new Object();
```

奇特：

构造方法-LinkedHashMap实现，包私有构造方法，为LinkedHashSet使用

```java
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```