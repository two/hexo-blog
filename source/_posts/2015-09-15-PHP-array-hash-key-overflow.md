layout: title
title: "PHP数组的key溢出问题"
date: 2015-09-15 10:52:49
tags: php
---

>作为PHP最重要的数据类型HashTable其key值是有一定的范围的，如果设置的key值过大就会出现溢出的问题，下面根据其内部结构及实现原理详细探讨一下key值溢出问题。

下面先给出一个key溢出的例子:
```php
<?php
$arr[1] = '1';
$arr[18446744073708551617333333333333] = '18446744073708551617333333333333';
$arr[] = 'test';
$arr[4294967296] = 'test';
$arr[9223372036854775807] = 'test';
$arr[9223372036854775808] = 'test';
var_dump($arr);
```

上面代码的输出结果如下:
```php
array(6) {
  [1]=>
  string(1) "1"
  [-999799117276250112]=>
  string(32) "18446744073708551617333333333333"
  [2]=>
  string(4) "test"
  [4294967296]=>
  string(4) "test"
  [9223372036854775807]=>
  string(4) "test"
  [-9223372036854775808]=>
  string(4) "test"
}
```
我们可以看到当key值比较小是没有问题，当key值很大时输出的值溢出了，临界点是`9223372036854775807`这个数字。
下面分析一下原因 。首先我们先分析一下HashTable的结构(本文分析的是php-5.5.15版本的源码),可以通过源码看一下:

```c
/* file: Zend/zend_hash.h */
typedef struct bucket {
    ulong h;                        /* Used for numeric indexing */ /*对char *key进行hash后的值，或者是用户指定的数字索引值，可能会溢出*/
    uint nKeyLength; /*hash关键字的长度，如果数组索引为数字，此值为0*/
    void *pData; /*指向value,一般是用户数据的副本，如果是指针数据，则指向pDataPtr*/
    void *pDataPtr; /*如果是指针数据，此值会指向真正的value,同时上面pData会指向此值*/
    struct bucket *pListNext; /*整个hash表的下一个元素*/
    struct bucket *pListLast; /*整个hash表该元素的上一个元素*/
    struct bucket *pNext; /*存放在同一个hash Bucket的下一个元素*/
    struct bucket *pLast; /*同一个hash bucket的上一个元素*/
    const char *arKey; /*保存当前值所对于的key字符串,这个字段只能定义在最后,实现变长结构体*/
} Bucket;

typedef struct _hashtable {
    uint nTableSize; /*hash Bucket的大小，最小为8，最以2*x增长*/
    uint nTableMask; /*nTableSize-1, 索引取值的优化*/
    uint nNumOfElements; /*hash Bucket中当前存在的元素个数， count()函数会直接返回此值*/
    ulong nNextFreeElement; /*下一个数字索引的位置*/
    Bucket *pInternalPointer;   /* Used for element traversal ,当前遍历的指针，foreach比for快的原因之一,这个指针指向当前激活的元素*/
    Bucket *pListHead; /*存储数组头元素指针*/
    Bucket *pListTail; /*存储数组尾元素指针*/
    Bucket **arBuckets; /*存储hash数组*/
    dtor_func_t pDestructor; /*在删除元素时执行的回调函数，用于资源的释放*/
    zend_bool persistent; /*指出了Bucket内存分配的方式。如果persistent为TRUE，则使用操作系统本身的内存分配函数为Bucket分配内存，否则使用PHP的内存分配函数*/
    unsigned char nApplyCount; /*标记当前hash Bucket被递归访问的次数(防止多次递归)*/
    zend_bool bApplyProtection; /*标记当前hash桶允许不允许多次访问。不允许时，最多只能递归3次*/
#if ZEND_DEBUG
    int inconsistent;
#endif
} HashTable;

```
假设我们已经对源码有了一定的了解了，我们可以知道`bucket.h`就是我们存储的key值，`bucket.h`的生成方法是根据`time33`算法获取的,对应到代码实现如下:
```c
//对于字符串类型的key
ZEND_API int _zend_hash_add_or_update(HashTable *ht, const char *arKey, uint nKeyLength, void *pData, uint nDataSize, void **pDest, int flag ZEND_FILE_LINE_DC)
{
    ulong h;
    uint nIndex;
    Bucket *p;
#ifdef ZEND_SIGNALS
    TSRMLS_FETCH();
#endif

    IS_CONSISTENT(ht);

    ZEND_ASSERT(nKeyLength != 0);

    CHECK_INIT(ht);

    //算出来hash key后需要根据hashTable的长度，把nIndex限制在这个长度内(通过nTableMask)
    h = zend_inline_hash_func(arKey, nKeyLength);
    nIndex = h & ht->nTableMask;

    p = ht->arBuckets[nIndex];
   ...
}
//对于数字类型的key
ZEND_API int _zend_hash_index_update_or_next_insert(HashTable *ht, ulong h, void *pData, uint nDataSize, void **pDest, int flag ZEND_FILE_LINE_DC)
{
    uint nIndex;
    Bucket *p;
#ifdef ZEND_SIGNALS
    TSRMLS_FETCH();
#endif

    IS_CONSISTENT(ht);
    CHECK_INIT(ht);

    /* 如果是新增元素(如$arr[] = 'hello'), 则使用nNextFreeElement值作为hash值,否则直接使用传入的key h 最为hash值 */
    if (flag & HASH_NEXT_INSERT) {
        h = ht->nNextFreeElement;
    }
    nIndex = h & ht->nTableMask;

    p = ht->arBuckets[nIndex];
    ...
}
//字符串的hash函数
static inline ulong zend_inline_hash_func(const char *arKey, uint nKeyLength)
{
    register ulong hash = 5381; //这个常量是哪儿来的？

    /* variant with the hash unrolled eight times */
    for (; nKeyLength >= 8; nKeyLength -= 8) {
        hash = ((hash << 5) + hash) + *arKey++;
        hash = ((hash << 5) + hash) + *arKey++;
        hash = ((hash << 5) + hash) + *arKey++;
        hash = ((hash << 5) + hash) + *arKey++;
        hash = ((hash << 5) + hash) + *arKey++;
        hash = ((hash << 5) + hash) + *arKey++;
        hash = ((hash << 5) + hash) + *arKey++;
        hash = ((hash << 5) + hash) + *arKey++;
    }
    switch (nKeyLength) {
        case 7: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
        case 6: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
        case 5: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
        case 4: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
        case 3: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
        case 2: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
        case 1: hash = ((hash << 5) + hash) + *arKey++; break;
        case 0: break;
EMPTY_SWITCH_DEFAULT_CASE()
    }
    return hash;
}
```

上面函数主要是插入或更新hashTable的函数，当插入的key是数字时，这个数字就是hastTable的索引值，其key值不经过hash算法，只经过`nIndex = h & ht->nTableMask;`来确保存储的值范围属于hastTable的范围内，所以可以看出索引值`key` ,与其对应的时`nIndex`这个值，正在存储的槽位就是`nIndex`这个地方。

这个key类型是`ulong`，也就是`unsigned long`类型。由于我们的机器是64位的，所以`unsigned long`类型的取值范围应该是`0~1844674407370955161`。PHP有两个预定义的变量`PHP_INT_MAX`和`PHP_INT_SIZE`对于64位的机器他们的值分别是9223372036854775807和8，这恰好是hasttable所能表示key的最大值,到这里也许你会有一个疑问:为什么`PHP_INT_MAX`的值比`key`的范围不一致?
要回答这个问题首先要知道，hastTable的key输出可以是负值，这是怎么做到的呢？其实一个hashTable的hash值一定是一个正整数才行，但是输出的数和hash值只是一个对应关系，不需要都为正整数， 虽然我们定义的参数为`unsigned long`,其实我们却可以传一个负数,比如`$arr[-1] = 'test'`，这时候也是和传递一个正数的处理过程是一样的。这时候`h`的值其实是`-1`的补码。再回到上面的问题，为什么`PHP_INT_MAX`的值比`key`范围不一致。当我们负值 PHP_INT_MAX时，其值是`9223372036854775807`，当赋值再比这个大时,输出的却是负数。这其实跟我们使用`var_dump`这个函数有关系, 下面代码是使用var_dump输出数组时所使用的方法:
```c
static int php_array_element_dump(zval **zv TSRMLS_DC, int num_args, va_list args, zend_hash_key *hash_key) /* {{{ */
{
    int level;

    level = va_arg(args, int);

    if (hash_key->nKeyLength == 0) { /* numeric key */
        php_printf("%*c[%ld]=>\n", level + 1, ' ', hash_key->h);
    } else { /* string key */
        php_printf("%*c[\"", level + 1, ' ');
        PHPWRITE(hash_key->arKey, hash_key->nKeyLength - 1);
        php_printf("\"]=>\n");
    }
    php_var_dump(zv, level + 2 TSRMLS_CC);
    return 0;
}
```
可以看到，当key为数字时输出的格式时`%ld`,值是`hash_key->h`，这就是问题所在了，存储的是一个`unsigned long`，输出的却是`long`，当值比`long`大时，自然输出的就是负数了。

总结: PHP的hastTable是通过链表法实现的，按说是不会存在溢出的问题，但是其索引值表示的范围有限，当超出索引值时就会造成溢出，这个溢出只存在当索引值为数字时，输入的数字为正，输出却为负值的原因是函数参数与输出的类型不一致导致的。
