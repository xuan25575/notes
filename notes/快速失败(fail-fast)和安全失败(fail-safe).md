#   快速失败(fail-fast)和安全失败(fail-safe)

### **1.fail-fast和fail-safe比较**

Iterator的安全失败是基于对底层集合做拷贝，因此，它不受源集合上修改的影响。

java.util包下面的所有的集合类都是快速失败的，而java.util.concurrent包下面的所有的类都是安全失败的。快速失败的迭代器会抛出ConcurrentModificationException异常，而安全失败的迭代器永远不会抛出这样的异常。

### 2.如何解决fail-fast机制

- 出现fail-fast事件
  - 当多个线程对同一个集合进行操作的时候，某线程访问集合的过程中，该集合的内容被其他线程所改变(即其它线程通过add、remove、clear等方法，改变了modCount的值)；这时，就会抛出ConcurrentModificationException异常，产生fail-fast事件。（迭代集合时候修改了集合）

fail-fast机制，是一种错误检测机制。它只能被用来检测错误，因为JDK并不保证fail-fast机制一定会发生**。**若在多线程环境下使用fail-fast机制的集合，建议使用“java.util.concurrent包下的类”去取代“java.util包下的类”。

### 3. fail-fast原理

> 若 “**modCount 不等于 expectedModCount**”，则抛出ConcurrentModificationException异常，产生fail-fast事件

源码分析

```java
 public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {
      ...
      // AbstractList中唯一的属性
      // 用来记录List修改的次数：每修改一次(添加/删除等操作)，将modCount+1
      protected transient int modCount = 0;
    
     ...
         
        private class Itr implements Iterator<E> {      
                int cursor = 0;
                int lastRet = -1;

              // modCount
                int expectedModCount = modCount;

                public boolean hasNext() {
                    return cursor != size();
                }

                public E next() {
                    checkForComodification();
                    try {
                        int i = cursor;
                        E next = get(i);
                        lastRet = i;
                        cursor = i + 1;
                        return next;
                    } catch (IndexOutOfBoundsException e) {
                        checkForComodification();
                        throw new NoSuchElementException();
                    }
                }

                public void remove() {
                    if (lastRet < 0)
                        throw new IllegalStateException();
                    checkForComodification();

                    try {
                        AbstractList.this.remove(lastRet);
                        if (lastRet < cursor)
                            cursor--;
                        lastRet = -1;
                        expectedModCount = modCount;
                    } catch (IndexOutOfBoundsException e) {
                        throw new ConcurrentModificationException();
                    }
                }

                final void checkForComodification() {
                   //以后每次遍历List中的元素的时候，都会比较expectedModCount和modCount是否相等；
                   // 若不相等，则抛出ConcurrentModificationException异常，产生fail-fast事件。
                    if (modCount != expectedModCount)
                        throw new ConcurrentModificationException();
                }
            }
     
        ...
 }
```

