# php哈希表实现

## HashTable的结构
```
typedef struct _hashtable { 
    uint nTableSize;        // hash Bucket的大小，最小为8，以2x增长。
    uint nTableMask;        // nTableSize-1 ， 索引取值的优化
    uint nNumOfElements;    // hash Bucket中当前存在的元素个数，count()函数会直接返回此值 
    ulong nNextFreeElement; // 下一个数字索引的位置
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

## **HashTable初始化**
```$xslt
ZEND_API int _zend_hash_init(HashTable *ht, uint nSize, hash_func_t pHashFunction,
                    dtor_func_t pDestructor, zend_bool persistent ZEND_FILE_LINE_DC)
{
    uint i = 3;
    //...
    if (nSize >= 0x80000000) {
        /* prevent overflow */
        ht->nTableSize = 0x80000000;
    } else {
        while ((1U << i) < nSize) {
            i++;
        }
        ht->nTableSize = 1 << i;
    }
    // ...
    ht->nTableMask = ht->nTableSize - 1;
 
    /* Uses ecalloc() so that Bucket* == NULL */
    if (persistent) {
        tmp = (Bucket **) calloc(ht->nTableSize, sizeof(Bucket *));
        if (!tmp) {
            return FAILURE;
        }
        ht->arBuckets = tmp;
    } else {
        tmp = (Bucket **) ecalloc_rel(ht->nTableSize, sizeof(Bucket *));
        if (tmp) {
            ht->arBuckets = tmp;
        }
    }
 
    return SUCCESS;
}
```