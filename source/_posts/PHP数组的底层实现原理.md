---
title: PHP数组的底层实现原理
date: 2019-07-05 10:03:24
tags: PHP
---

# 基本概念
由于PHP的数组基于 `HashTable` 实现，这里列举一下实现中出现的基本概念。 哈希表是一种通过哈希函数，将特定的键映射到特定值的一种数据结构，它维护键和值之间一一对应关系。
- 键(key)：用于操作数据的标示，例如PHP数组中的索引，或者字符串键等等。
- 槽(slot/bucket)：哈希表中用于保存数据的一个单元，也就是数据真正存放的容器。
- 哈希函数(hash function)：将key映射(map)到数据应该存放的slot所在位置的函数。
- 哈希冲突(hash collision)：哈希函数将两个不同的key映射到同一个索引的情况。


# PHP实现
查看PHP源码中声明的结构如下
## HashTable 结构
```c
typedef struct _hashtable { 
    uint nTableSize;        // hash Bucket的大小，最小为8，以2x增长。
    uint nTableMask;        // nTableSize-1 ， 索引取值的优化
    uint nNumOfElements;     // hash Bucket中当前存在的元素个数，count()函数会直接返回此值 
    ulong nNextFreeElement;    // 下一个数字索引的位置
    Bucket *pInternalPointer;   // 当前遍历的指针（foreach比for快的原因之一）
    Bucket *pListHead;          // 存储数组头元素指针
    Bucket *pListTail;          // 存储数组尾元素指针
    Bucket **arBuckets;         // 存储hash数组
    dtor_func_t pDestructor;    // 在删除元素时执行的回调函数，用于资源的释放
    zend_bool persistent;       //指出了Bucket内存分配的方式。如果persisient为TRUE，则使用操作系统本身的内存分配函数为Bucket分配内存，否则使用PHP的内存分配函数。
    unsigned char nApplyCount; // 标记当前hash Bucket被递归访问的次数（防止多次递归）
    zend_bool bApplyProtection;// 标记当前hash桶允许不允许多次访问，不允许时，最多只能递归3次
#if ZEND_DEBUG
    int inconsistent;
#endif
} HashTable;
```

## Bucket 结构
```c
typedef struct bucket {  
    ulong h;                   /* 4字节 对char *key进行hash后的值，或者是用户指定的数字索引值/* Used for numeric indexing */
    uint nKeyLength;           /* 4字节 字符串索引长度，如果是数字索引，则值为0 */  
    void *pData;               /* 4字节 实际数据的存储地址，指向value，一般是用户数据的副本，如果是指针数据，则指向pDataPtr,这里又是个指针，zval存放在别的地方*/
    void *pDataPtr;            /* 4字节 引用数据的存储地址，如果是指针数据，此值会指向真正的value，同时上面pData会指向此值 */  
    struct bucket *pListNext;  /* 4字节 整个哈希表的该元素的下一个元素*/  
    struct bucket *pListLast;  /* 4字节 整个哈希表的该元素的上一个元素*/  
    struct bucket *pNext;      /* 4字节 同一个槽，双向链表的下一个元素的地址 */  
    struct bucket *pLast;      /* 4字节 同一个槽，双向链表的上一个元素的地址*/  
    char arKey[1];             /* 1字节 保存当前值所对于的key字符串，这个字段只能定义在最后，实现变长结构体*/  
} Bucket;
```

- `Bucket` 结构体维护了两个双向链表：
  - `pNext` 和 `pLast` 指针分别指向本槽位所在的链表的关系。
  - `pListNext` 和 `pListLast` 指针指向的则是整个哈希表所有的数据之间的链接关系。 
- `HashTable` 结构体则维护了全局链表头尾元素的指针： 
  - `pListHead` 头元素指针
  - `pListTail` 尾元素指针

>**<span style="color:red"> NOTICE: <span>** <span style="color:gray"> PHP中数组的操作函数非常多，例如：array_shift()和array_pop()函数，分别从数组的头部和尾部弹出元素。 哈希表中保存了头部和尾部指针，这样在执行这些操作时就能在常数时间内找到目标。 PHP中还有一些使用的相对不那么多的数组操作函数：next()，prev()等的循环中， 哈希表的另外一个指针就能发挥作用了：pInternalPointer，这个用于保存当前哈希表内部的指针。 这在循环时就非常有用。 <span>

## 总结结构关系如下图：
![结构图](/images/php_hashtable.drawio.png)

# 问题点

## 顺序读取
由于 `HashTable` 是无序的，为了实现顺序读取，PHP在 `HashTable` 中维护了一个全局链表，按**插入顺序**将所有 `bucket` 全部串联起来， `HashTable` 中只有一个全局链表。

## 哈希冲突
当不同 `key` 经过哈希函数计算后，得出的哈希值相同，就是哈希冲突。

### 解决哈希冲突的常用方法：

#### **链接法（PHP采用此方法）**
- 在冲突位置构造一个双向链表，将哈希值相同的 `bucket` 放到相同 `slot` 对应的链表中。
- 通过哈希函数定位到对应的 `bucket` 链表时，需要遍历链表，逐个对比 `key` 值，继而找到目标元素。
- 每个 `bucket` 之间的链接则是将原 `value` 的下标保存到新 `value` 的 `zval.u2.next` 里，新 `value` 放在当前位置上，从而形成一个单向链表。

#### 开放寻址法
- 开放寻址法是槽本身直接存放数据， 在插入数据时如果 `key` 所映射到的索引已经有数据了，这说明发生了冲突，这时会寻找下一个槽， 如果该槽也被占用了则继续寻找下一个槽，直到寻找到没有被占用的槽，在查找时也使用同样的策略来进行。
- 由于开放寻址法处理冲突的时候占用的是其他槽位的空间,这可能会导致后续的 `key` 在插入的时候更加容易出现哈希冲突，所以采用开放寻址法的哈希表的装载因子不能太高，否则容易出现性能下降。

    > **<span style="color:red"> NOTICE: <span>** <span style="color:gray">装载因子是哈希表保存的元素数量和哈希表容量的比，通常采用链接法解决冲突的哈希表的装载 因子最好不要大于1，而采用开放寻址法的哈希表最好不要大于0.5。</span>

## 扩容
### 触发时机
当插入元素并且没有空闲空间时，就会触发自动扩容机制，扩容后再执行插入。

### 扩容流程
1. 如果已删除元素所占比例达到阈值，则会移除已被逻辑删除的 `Bucket`，然后将后面的 `Bucket` 向前补上空缺的 `Bucket`，因为 `Bucket` 的下标发生了变动，所以还需要更改每个元素在中间映射表中储存的实际下标值。
2. 如果未达到阈值，PHP 则会申请一个大小是原数组两倍的新数组（**2mb以内每次扩容翻倍，2mb以上每次扩容增加2mb**），并将旧数组中的数据复制到新数组中，因为数组长度发生了改变，所以 key-value 的映射关系需要重新计算，这个步骤为重建索引。

## Rehash