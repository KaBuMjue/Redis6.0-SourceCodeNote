虽然字典在很多高级语言中都有，用途很广泛，但C语言却是没有，所以还得自己"造轮子"。接下来就看看Redis是如何实现字典数据结构的，相关文件为*dict.h，dict.c*。



##  dict.h

在dict.h中定义了多种数据结构用于实现字典。

* dict   -- 字典

  ```c
  typedef struct dict {
      dictType *type;		// 指明该字典使用的特定函数，如使用的哈希函数
      void *privdata;		// 私有数据
      dictht ht[2];		// 两个哈希表
      long rehashidx; 	// 如果当前没有在rehash,值为-1
      unsigned long iterators; 	// 当前字典正在使用的迭代器数目
  } dict;
  ```

  这里的两个哈希表通常只使用ht[0]，ht[1]只在rehash的时候使用，下文会详述。

  

* dictType   -- 字典类型

  ```c
  typedef struct dictType {
      // 使用的哈希函数，用来计算key对应的哈希值
      uint64_t (*hashFunction)(const void *key);
      
      // 复制键的函数
      void *(*keyDup)(void *privdata, const void *key);
      
      // 复制值的函数
      void *(*valDup)(void *privdata, const void *obj);
      
      // 键比较函数
      int (*keyCompare)(void *privdata, const void *key1, const void *key2);
      
      // 键销毁函数
      void (*keyDestructor)(void *privdata, void *key);
      
      // 值销毁函数
      void (*valDestructor)(void *privdata, void *obj);
  } dictType;
  ```

  

* dictht   -- 哈希表

  ```c
  typedef struct dictht {
      dictEntry **table;		// 哈希表数组
      unsigned long size;		// 哈希表大小
      unsigned long sizemask;	// 掩码，用来计算索引值，通常为size-1
      unsigned long used;		// 哈希表已使用节点数
  } dictht;
  ```

  

* dictEntry   -- 哈希表节点

  ```c
  typedef struct dictEntry {
      void *key;		// 键
      union {			// 值
          void *val;
          uint64_t u64;
          int64_t s64;
          double d;
      } v;
      struct dictEntry *next;	// 指向下一节点，形成链表
  } dictEntry;
  ```

  值是union类型的，所以针对不同类型的节点，底层使用的值类型可以灵活变化。

  因为Redis处理哈希冲突的方法是*开链法*，所以哈希值相同的节点以链表的形式连接在一起。

* dictIterator   -- 字典迭代器

  ```c
  typedef struct dictIterator {
      dict *d;	// 迭代器对应的字典
      long index;	
      int table, safe;
      dictEntry *entry, *nextEntry;
      /* unsafe iterator fingerprint for misuse detection. */
      long long fingerprint;
  } dictIterator;
  ```

  safe属性指示这个迭代器是否为*安全*的，为1则是安全的，可以在迭代时使用dictAdd、dictFind等函数，为0则是不安全的，迭代时只能使用dictNext函数。

  

  ## dict.c

  *Redis6.0默认使用的哈希算法是SipHash，在siphash.c定义。具体实现比较复杂，感兴趣的可以去看看，SipHash的优点是在输入的key值很小时也能产生随机性较好的输出。*

  

  * _dictReset   -- 重置哈希表

    ```c
    static void _dictReset(dictht *ht)
    {
        ht->table = NULL;
        ht->size = 0;
        ht->sizemask = 0;
        ht->used = 0;
    }
    ```

    

  * _dictInit   -- 初始化字典

    ```c
    int _dictInit(dict *d, dictType *type,
            void *privDataPtr)
    {
        _dictReset(&d->ht[0]);	
        _dictReset(&d->ht[1]);
        d->type = type;
        d->privdata = privDataPtr;
        d->rehashidx = -1;
        d->iterators = 0;
        return DICT_OK;		//返回0，代表成功
    } 
    ```

  

  * dictCreate   -- 创建一个字典

    ```c
    dict *dictCreate(dictType *type,
            void *privDataPtr)
    {
        dict *d = zmalloc(sizeof(*d));
    
        _dictInit(d,type,privDataPtr);
        return d;
    }
    ```

    这三个函数用来创建、初始化一个字典，代码很简单不需再解释。

  

  * dictExpand   -- 扩展哈希表大小为size

    ```c
    int dictExpand(dict *d, unsigned long size)
    {
      /* 如果当前正在rehash或者size小于哈希表大小，返会DICT_ERR（即1）代表失败 */
      if (dictIsRehashing(d) || d->ht[0].used > size)
          return DICT_ERR;
    
      dictht n; /* 新的哈希表 */
      unsigned long realsize = _dictNextPower(size);	/* realsize为第一个大于等于size的													 * 2的整数次方 */
    
      /* realsize与当前哈希表大小相同，不需要改动 */
      if (realsize == d->ht[0].size) return DICT_ERR;
    
      /* 给新的哈希表分配空间，并初始化相关属性 */
      n.size = realsize;
      n.sizemask = realsize-1;
      n.table = zcalloc(realsize*sizeof(dictEntry*));
      n.used = 0;
    
      /* 如果当前的哈希表为空，无需rehash，直接设置ht[0]即可 */
      if (d->ht[0].table == NULL) {
          d->ht[0] = n;
          return DICT_OK;
      }
    
      /* 设置ht[1]，准备rehash */
      d->ht[1] = n;
      d->rehashidx = 0;
      return DICT_OK;
    }
    ```
  
    这个函数可以用来扩展/收缩哈希表，使得哈希表的负载因子在一个合理的范围内，便于后续的rehash。



* dictRehash   -- **渐进式**的rehash

  ```c
  int dictRehash(dict *d, int n) {
      int empty_visits = n*10;	 /* 每次rehash最多能遍历的空桶数目 */
      if (!dictIsRehashing(d)) return 0;	/* 如果未在rehash，返回0 */
  
      while(n-- && d->ht[0].used != 0) {
          dictEntry *de, *nextde;
  
          /* Note that rehashidx can't overflow as we are sure there are more
           * elements because ht[0].used != 0 */
          assert(d->ht[0].size > (unsigned long)d->rehashidx);
          while(d->ht[0].table[d->rehashidx] == NULL) {
              d->rehashidx++;
              if (--empty_visits == 0) return 1; /* 遍历的空桶数达到了empty_visits，返回1表												* 示还未完成rehash */
          }
          de = d->ht[0].table[d->rehashidx];
          /* Move all the keys in this bucket from the old to the new hash HT */
          while(de) {
              uint64_t h;
  
              nextde = de->next;
              /* Get the index in the new hash table */
              h = dictHashKey(d, de->key) & d->ht[1].sizemask;
              de->next = d->ht[1].table[h];
              d->ht[1].table[h] = de;
              d->ht[0].used--;
              d->ht[1].used++;
              de = nextde;
          }
          d->ht[0].table[d->rehashidx] = NULL;
          d->rehashidx++;
      }
  
      /* Check if we already rehashed the whole table... */
      if (d->ht[0].used == 0) {
          zfree(d->ht[0].table);
          d->ht[0] = d->ht[1];
          _dictReset(&d->ht[1]);
          d->rehashidx = -1;
          return 0;
      }
  
      /* More to rehash... */
      return 1;
  }
  
  ```

  因为Redis是单线程的，所以如果我们有很多的数据需要rehash，想要一次性完成rehash会花费大量的时间，从而阻塞Redis服务器使之在这段时间内停止服务。而这绝不是我们希望的，所以Redis采用**分多次、渐进式的rehash策略**，慢慢的将ht[0]的数据rehash到ht[1]中。从这我们可以知道，*ht[0]是平常字典使用的哈希表，数据都存放在这，而ht[1]只在对ht[0]rehash时使用*。

  为了能够实现渐进式的rehash，Redis的dictRehash函数有一个参数n，代表本次执行rehash的节点(又叫桶)数，当rehash的节点数到n后就结束本次rehash。

  但也可能提前结束rehash，通过检查所遍历到的空桶节点数目是否到达了一个阈值（上述代码中的empty_visits，为n*10），到达这个阈值就返回，等待下次再执行rehash。这样做有效避免了当哈希表中存在大量空桶节点时函数执行时间过长，导致服务器阻塞的情况发生。
  
  在rehash过程中，一个桶节点中可能挂有多个节点(因为使用开链法解决哈希冲突)，所以需要将该桶下的所有节点都转移到新哈希表中。
  
  完成rehash后，需要交换ht[0]和ht[1]（不要忘记ht[0]才是字典常使用的哈希表）。



* dictRehashMilliseconds   -- 在给定的时间(*毫秒为单位*)内进行rehash

  ```c
  int dictRehashMilliseconds(dict *d, int ms) {
      long long start = timeInMilliseconds();	 //获取当前时间
      int rehashes = 0;	//已rehash的节点数
  
      while(dictRehash(d,100)) {		//每次rehash100个节点
          rehashes += 100;
          if (timeInMilliseconds()-start > ms) break;	//到给定时间了，退出 
      }
      return rehashes;	//返回已rehash的节点数
  }
  ```




* _dictRehashStep   -- 单步rehash

  ```c
  static void _dictRehashStep(dict *d) {
      if (d->iterators == 0) dictRehash(d,1);
  }
  ```

  当没有迭代器指向当前这个字典时，执行单步rehash(即只rehash一个桶)。

  *在对字典进行查找、更新等操作时，如果该字典正处于rehash的状态，会顺带执行该函数。这样做将整个rehash消耗的时间部分分摊到了每次对字典的查找、更新操作上。*



* dictAddRaw   -- 向字典中添加一个节点(未设置值)

  ```c
  dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
  {
      long index;
      dictEntry *entry;
      dictht *ht;
  
      if (dictIsRehashing(d)) _dictRehashStep(d);	/* 若字典正处于rehash，												 * 执行单步rehash */
  
      /* Get the index of the new element, or -1 if
       * the element already exists. */
      if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
          return NULL;
  
      /* Allocate the memory and store the new entry.
       * Insert the element in top, with the assumption that in a database
       * system it is more likely that recently added entries are accessed
       * more frequently. */
      ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
      entry = zmalloc(sizeof(*entry));
      entry->next = ht->table[index];
      ht->table[index] = entry;
      ht->used++;
  
      /* entry->key = key,如果设置了keyDup,则调用keydup */
      dictSetKey(d, entry, key);
      return entry;
  }
  
  ```

  _dictKeyIndex获得节点key对应哈希表的下标，当节点已存在时返回-1，并令existing指向该节点。如果字典正在rehash，返回的则是ht[1]的下标。
  
  如果字典正在rehash，新加的节点会放在ht[1]里。



* dictAdd   -- 向字典中添加一个节点(设置值)

  ```c
  int dictAdd(dict *d, void *key, void *val)
  {
      dictEntry *entry = dictAddRaw(d,key,NULL);
  
      if (!entry) return DICT_ERR;
      dictSetVal(d, entry, val);
      return DICT_OK;
  }
  ```

  调用dictAddRaw后，若entry不为null，则设置值。



* dictReplace   -- 重写节点的值(如该节点不存在，添加之)

  ```c
  int dictReplace(dict *d, void *key, void *val)
  {
      dictEntry *entry, *existing, auxentry;
  
      /* Try to add the element. If the key
       * does not exists dictAdd will succeed. */
      entry = dictAddRaw(d,key,&existing);
      if (entry) {
          dictSetVal(d, entry, val);
          return 1;
      }
  
      /* Set the new value and free the old one. Note that it is important
       * to do that in this order, as the value may just be exactly the same
       * as the previous one. In this context, think to reference counting,
       * you want to increment (set), and then decrement (free), and not the
       * reverse. */
      auxentry = *existing;	
      dictSetVal(d, existing, val);
      dictFreeVal(d, &auxentry);
      return 0;
  }
  ```



* dictGenericDelete   -- 删除字典中的一个节点

  ```c
  static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
      uint64_t h, idx;
      dictEntry *he, *prevHe;
      int table;
  
      if (d->ht[0].used == 0 && d->ht[1].used == 0) return NULL;
  
      if (dictIsRehashing(d)) _dictRehashStep(d);
      h = dictHashKey(d, key);
  
      for (table = 0; table <= 1; table++) {
          idx = h & d->ht[table].sizemask;
          he = d->ht[table].table[idx];
          prevHe = NULL;
          while(he) {
              if (key==he->key || dictCompareKeys(d, key, he->key)) {
                  /* Unlink the element from the list */
                  if (prevHe)
                      prevHe->next = he->next;
                  else
                      d->ht[table].table[idx] = he->next;
                  if (!nofree) {
                      dictFreeKey(d, he);
                      dictFreeVal(d, he);
                      zfree(he);
                  }
                  d->ht[table].used--;
                  return he;
              }
              prevHe = he;
              he = he->next;
          }
          if (!dictIsRehashing(d)) break;
      }
      return NULL; /* not found */
  }
  ```

  