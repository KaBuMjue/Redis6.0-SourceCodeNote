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
      unsigned long iterators; 	// 当前字典正在使用的(安全)迭代器数目
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
      /* 用来检测是否滥用了不安全迭代器(比如调用了add,find等操作) */
      long long fingerprint;
  } dictIterator;
  ```

  safe属性指示这个迭代器是否为*安全*的，为1则是安全的，可以在迭代时使用dictAdd、dictFind等函数，为0则是不安全的，迭代时只能使用dictNext函数。

  

  ## dict.c

  *Redis6.0默认使用的哈希算法是SipHash，在siphash.c定义。具体实现比较复杂，感兴趣的可以去看看，SipHash的优点是在输入的key值很小时也能产生随机性较好的输出。*

  

  _dictReset   -- 重置哈希表

  ```c
  static void _dictReset(dictht *ht)
  {
      ht->table = NULL;
      ht->size = 0;
      ht->sizemask = 0;
      ht->used = 0;
  }
  ```

  

  _dictInit   -- 初始化字典

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

  

  dictCreate   -- 创建一个字典

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

  

  dictExpand   -- 扩展哈希表大小为size

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



* dictFind   -- 查找节点

  ```c
  dictEntry *dictFind(dict *d, const void *key)
  {
      dictEntry *he;
      uint64_t h, idx, table;
  
      if (dictSize(d) == 0) return NULL; /* dict is empty */
      if (dictIsRehashing(d)) _dictRehashStep(d);	//单步rehash
      h = dictHashKey(d, key);	// 获得节点的键对应的哈希值
      for (table = 0; table <= 1; table++) {
          idx = h & d->ht[table].sizemask;
          he = d->ht[table].table[idx];
          while(he) {
              if (key==he->key || dictCompareKeys(d, key, he->key))
                  return he;
              he = he->next;
          }
          if (!dictIsRehashing(d)) return NULL;
      }
      return NULL;
  }
  ```

  如果字典正在rehash，在ht[0]没找到该节点，会再去ht[1]寻找。没找到节点则返回null。

  这个函数只是返回节点，Redis还实现了dictFetchValue函数去获得节点对应的值:

  `dictEntry *he; he = dictFind(d,key); return he ? dictGetVal(he) : NULL; `



* dictAddRaw   -- 向字典中添加一个节点(未设置值)

  ```c
  dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
  {
      long index;
      dictEntry *entry;
      dictht *ht;
  
      if (dictIsRehashing(d)) _dictRehashStep(d);	/* 执行单步rehash */
  
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
  
      if (dictIsRehashing(d)) _dictRehashStep(d);	//单步rehash
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
                  if (!nofree) {			//是否需要释放该节点
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
          if (!dictIsRehashing(d)) break;	//如果正在rehash，ht[0]中没找到，要去ht[1]找该节点并删除(如果存在的话)
      }
      return NULL; /* not found */
  }
  ```

  该函数用来实现 dictDelete(dict* ht, const void* key) : `return dictGenericDelete(ht,key,0) ? DICT_OK : DICT_ERR;`  删除一个节点，并*释放其空间(key,val)*。
  
  以及 dictUnlink(dict* ht, const void* key) : `return dictGenericDelete(ht,key,1);`
  
  将该节点从dict中移除并返回给调用者，注意这是*不释放节点空间的*，因为稍后可能会再用到。使用完后，再通过dictFreeUnlinkedEntry来释放空间。
  
  

* _dictClear   -- 清空整个字典

  ```c
  int _dictClear(dict *d, dictht *ht, void(callback)(void *)) {
      unsigned long i;
  
      /* Free all the elements */
      for (i = 0; i < ht->size && ht->used > 0; i++) {
          dictEntry *he, *nextHe;
  
          if (callback && (i & 65535) == 0) callback(d->privdata);
  
          if ((he = ht->table[i]) == NULL) continue;
          while(he) {
              nextHe = he->next;
              dictFreeKey(d, he);
              dictFreeVal(d, he);
              zfree(he);
              ht->used--;
              he = nextHe;
          }
      }
      /* Free the table and the allocated cache structure */
      zfree(ht->table);
      /* Re-initialize the table */
      _dictReset(ht);
      return DICT_OK; /* never fails */
  }
  
  ```

  这个函数很简单，遍历每个桶，并释放桶上已链表形式连接起来的节点。但这里注意这里的一行代码：`if (callback && (i & 65535) == 0) callback(d->privdata);`
  
  就是每当遍历了65535个桶后，就执行一次callback函数。 为什么要这样做呢？因为Redis是**单线程**的，而如果要清除的字典中的节点数特别多的话，这个函数就要花费大量的时间从而阻塞服务器。添加这行代码后，Redis就能能在执行该函数期间能够间期性的执行其它操作(执行callback函数)。



* dictFingerprint   -- 获得字典指纹(fingerprint)

  ```c
  long long dictFingerprint(dict *d) {
      long long integers[6], hash = 0;
      int j;
  
      integers[0] = (long) d->ht[0].table;
      integers[1] = d->ht[0].size;
      integers[2] = d->ht[0].used;
      integers[3] = (long) d->ht[1].table;
      integers[4] = d->ht[1].size;
      integers[5] = d->ht[1].used;
  
      /* We hash N integers by summing every successive integer with the integer
       * hashing of the previous sum. Basically:
       *
       * Result = hash(hash(hash(int1)+int2)+int3) ...
       *
       * This way the same set of integers in a different order will (likely) hash
       * to a different number. */
      for (j = 0; j < 6; j++) {
          hash += integers[j];
          /* For the hashing step we use Tomas Wang's 64 bit integer hash. */
          hash = (~hash) + (hash << 21); // hash = (hash << 21) - hash - 1;
          hash = hash ^ (hash >> 24);
          hash = (hash + (hash << 3)) + (hash << 8); // hash * 265
          hash = hash ^ (hash >> 14);
          hash = (hash + (hash << 2)) + (hash << 4); // hash * 21
          hash = hash ^ (hash >> 28);
          hash = hash + (hash << 31);
      }
      return hash;
  }
  ```

  指纹是一些字典的属性集结而成的longlong类型的数字，*代表着某个时间点字典的状态*。

  指纹只在字典迭代器中使用，当一个**不安全**的迭代器初始化后，记录下当前字典指纹，在该迭代器释放前检查字典指纹是否改变。如果改变了，证明在使用这个迭代器期间执行了不被允许的操作(不安全的迭代器不能执行add、find操作)。



* dictGetIterator / dictGetSafeIterator    获取一个不安全/安全字典迭代器

  ```c
  dictIterator *dictGetIterator(dict *d)
  {
      dictIterator *iter = zmalloc(sizeof(*iter));
  
      iter->d = d;
      iter->table = 0;	// 默认为ht[0]
      iter->index = -1;
      iter->safe = 0;		// 为0代表不安全
      iter->entry = NULL;
      iter->nextEntry = NULL;
      return iter;
  }
  
  dictIterator *dictGetSafeIterator(dict *d) {
      dictIterator *i = dictGetIterator(d);
  
      i->safe = 1;	//为1代表安全
      return i;
  }
  ```




* dictReleaseIterator   -- 释放一个迭代器

  ```c
  void dictReleaseIterator(dictIterator *iter)
  {
      if (!(iter->index == -1 && iter->table == 0)) {
          if (iter->safe)
              iter->d->iterators--;
          else
              assert(iter->fingerprint == dictFingerprint(iter->d));
      }
      zfree(iter);
  }
  ```

  如果是已经使用过的迭代器(!(iter->index == -1 && iter->table == 0))，如果是*安全*的迭代器，将字典的iterators属性减1，不安全的迭代器比较指纹是否改变，改变则终止程序(使用了assert)。而后释放迭代器空间。

  

* dictNext   -- 获取迭代器iter当前指向节点的下一个节点

  ```c
  dictEntry *dictNext(dictIterator *iter)
  {
      while (1) {
          if (iter->entry == NULL) {
              dictht *ht = &iter->d->ht[iter->table];
              if (iter->index == -1 && iter->table == 0) {
                  if (iter->safe)
                      iter->d->iterators++;
                  else
                      iter->fingerprint = dictFingerprint(iter->d);
              }
              iter->index++;
              if (iter->index >= (long) ht->size) {
                  if (dictIsRehashing(iter->d) && iter->table == 0) {
                      iter->table++;
                      iter->index = 0;
                      ht = &iter->d->ht[1];
                  } else {
                      break;
                  }
              }
              iter->entry = ht->table[iter->index];
          } else {
              iter->entry = iter->nextEntry;
          }
          if (iter->entry) {
              /* We need to save the 'next' here, the iterator user
               * may delete the entry we are returning. */
              iter->nextEntry = iter->entry->next;
              return iter->entry;
          }
      }
      return NULL;
  }
  ```

  首先判断当前迭代器是否指向null，若指向null，判断迭代器是否为刚刚获取的(iter->index == -1 && iter->table == 0)，而且若为*安全*的迭代器就将字典的iterators属性加1，*不安全*的就记录下当前字典的指纹。将index+1，如果indx大于当前哈希表大小，判断是否正在rehash，如果是则让迭代器指向ht[1]，否则就退出循环返回null。剩下的就很简单了，不用多说。

  

* dictGetRandomKey   -- 随机获取一个节点（用于实现某些随机算法）

  ```c
  dictEntry *dictGetRandomKey(dict *d)
  {
    dictEntry *he, *orighe;
      unsigned long h;
      int listlen, listele;
  
      if (dictSize(d) == 0) return NULL;
      if (dictIsRehashing(d)) _dictRehashStep(d);	//单步rehash
      if (dictIsRehashing(d)) {
          do {
              /* We are sure there are no elements in indexes from 0
               * to rehashidx-1 */
              h = d->rehashidx + (random() % (d->ht[0].size +
                                              d->ht[1].size -
                                              d->rehashidx));
              he = (h >= d->ht[0].size) ? d->ht[1].table[h - d->ht[0].size] :
                                        d->ht[0].table[h];
          } while(he == NULL);
      } else {
          do {
              h = random() & d->ht[0].sizemask;
              he = d->ht[0].table[h];
          } while(he == NULL);
      }
  
      /* Now we found a non empty bucket, but it is a linked
       * list and we need to get a random element from the list.
       * The only sane way to do so is counting the elements and
       * select a random index. */
      listlen = 0;
      orighe = he;
      while(he) {
          he = he->next;
          listlen++;
      }
      listele = random() % listlen;
      he = orighe;
      while(listele--) he = he->next;
      return he;
  }
  ```

  如果字典正在rehash，[0,rehashidx-1]范围内的元素都已经完成了rehash自然是不能再作为下标，所以`d->ht[0].size + d->ht[1].size - d->rehashidx`就是只产生在[rehashidx, dictSize-1]范围内的下标，这里还要注意如果得到的下标大于ht[0]的大小，就要到ht[1]中获取该下标对应的(桶)节点。字典不在rehash的情况就很简单，不必多说。

  最后因为我们得到的只是该下标对应的桶节点，可能有一条链表“挂在”上面(开链法)，所以我们需要从链表中随机抽取一个元素返回。所以在先获得链表长度后，随机获得一个范围在[0,listlen-1]的下标，并返回位于该位置的节点。

  

* dictGetSomeKeys   -- 随机获取一些节点（可能返回重复的key）

  ```c
  unsigned int dictGetSomeKeys(dict *d, dictEntry **des, unsigned int count) {
      unsigned long j; /* internal hash table id, 0 or 1. */
      unsigned long tables; /* 1 or 2 tables? */
      unsigned long stored = 0, maxsizemask;
      unsigned long maxsteps;
  
      if (dictSize(d) < count) count = dictSize(d);
      maxsteps = count*10;
  
      /* Try to do a rehashing work proportional to 'count'. */
      for (j = 0; j < count; j++) {
          if (dictIsRehashing(d))
              _dictRehashStep(d);	//执行count次单步rehash
          else
              break;
      }
  
      tables = dictIsRehashing(d) ? 2 : 1;
      maxsizemask = d->ht[0].sizemask;
      if (tables > 1 && maxsizemask < d->ht[1].sizemask)
          maxsizemask = d->ht[1].sizemask;
  
      /* Pick a random point inside the larger table. */
      unsigned long i = random() & maxsizemask;
      unsigned long emptylen = 0; /* Continuous empty entries so far. */
      while(stored < count && maxsteps--) {
          for (j = 0; j < tables; j++) {
              /* Invariant of the dict.c rehashing: up to the indexes already
               * visited in ht[0] during the rehashing, there are no populated
               * buckets, so we can skip ht[0] for indexes between 0 and idx-1. */
              if (tables == 2 && j == 0 && i < (unsigned long) d->rehashidx) {
                  /* Moreover, if we are currently out of range in the second
                   * table, there will be no elements in both tables up to
                   * the current rehashing index, so we jump if possible.
                   * (this happens when going from big to small table). */
                  if (i >= d->ht[1].size)
                      i = d->rehashidx;
                  else
                      continue;
              }
              if (i >= d->ht[j].size) continue; /* Out of range for this table. */
              dictEntry *he = d->ht[j].table[i];
  
              /* Count contiguous empty buckets, and jump to other
               * locations if they reach 'count' (with a minimum of 5). */
              if (he == NULL) {
                  emptylen++;
                  if (emptylen >= 5 && emptylen > count) {
                      i = random() & maxsizemask;
                      emptylen = 0;
                  }
              } else {
                  emptylen = 0;
                  while (he) {
                      /* Collect all the elements of the buckets found non
                       * empty while iterating. */
                      *des = he;
                      des++;
                      he = he->next;
                      stored++;
                      if (stored == count) return stored;
                  }
              }
          }
          i = (i+1) & maxsizemask;
      }
      return stored;
  }
  ```
  
  注意这个函数返回的(即存放在des中的)节点可能会**重复**，而且可能返回的节点数*小于*count(当while循环执行了maxsteps次后就会返回)。所以这个函数并不适用希望返回的元素良好分布的场景。
  
  
  
* dictGetFairRandomKey   -- 随机获取一个节点（较dictGetRandomKey更加良好的分布）
  
  ```c
  #define GETFAIR_NUM_ENTRIES 15
  dictEntry *dictGetFairRandomKey(dict *d) {
      dictEntry *entries[GETFAIR_NUM_ENTRIES];
      unsigned int count = dictGetSomeKeys(d,entries,GETFAIR_NUM_ENTRIES);
      /* Note that dictGetSomeKeys() may return zero elements in an unlucky
       * run() even if there are actually elements inside the hash table. So
       * when we get zero, we call the true dictGetRandomKey() that will always
       * yeld the element if the hash table has at least one. */
      if (count == 0) return dictGetRandomKey(d);
      unsigned int idx = rand() % count;
      return entries[idx];
  }
  ```
  
  为什么说dictRandomKey不够公平呢？因为dictRandomKey首先选出一个桶节点，再从桶上“挂”着的链表中随机选取一个元素。*这样就导致处于不同链表(长度不同)的节点被选取的概率不同！*较长的链表中的元素被选取的概率更小。每个节点被选取的概率为 [1/(bucketNums) * 1/(chainLen)]。
  
  为了能够更加公平的选取节点，dictGetFairRandomKey通过调用dictGetSomeKeys获得多个桶节点上链表上**所有**的节点(最多GETFAIR_NUM_ENTRIES个)，并存放在数组里，再从数组中随机选取一个节点返回。这样就消除了不同链长导致概率不同的问题。



* dictScan   -- 迭代字典所有节点

  ```c
  unsigned long dictScan(dict *d,
                         unsigned long v,
                         dictScanFunction *fn,
                         dictScanBucketFunction* bucketfn,
                         void *privdata)
  {
      dictht *t0, *t1;
      const dictEntry *de, *next;
      unsigned long m0, m1;
  
      if (dictSize(d) == 0) return 0;
  
      /* Having a safe iterator means no rehashing can happen, see _dictRehashStep.
       * This is needed in case the scan callback tries to do dictFind or alike. */
      d->iterators++;
  
      if (!dictIsRehashing(d)) {
          t0 = &(d->ht[0]);
          m0 = t0->sizemask;
  
          /* Emit entries at cursor */
          if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
          de = t0->table[v & m0];
          while (de) {
              next = de->next;
              fn(privdata, de);
              de = next;
          }
  
          /* Set unmasked bits so incrementing the reversed cursor
           * operates on the masked bits */
          v |= ~m0;
  
          /* Increment the reverse cursor */
          v = rev(v);
          v++;
          v = rev(v);
  
      } else {
          t0 = &d->ht[0];
          t1 = &d->ht[1];
  
          /* Make sure t0 is the smaller and t1 is the bigger table */
          if (t0->size > t1->size) {
              t0 = &d->ht[1];
              t1 = &d->ht[0];
          }
  
          m0 = t0->sizemask;
          m1 = t1->sizemask;
  
          /* Emit entries at cursor */
          if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
          de = t0->table[v & m0];
          while (de) {
              next = de->next;
              fn(privdata, de);
              de = next;
          }
  
          /* Iterate over indices in larger table that are the expansion
           * of the index pointed to by the cursor in the smaller table */
          do {
              /* Emit entries at cursor */
              if (bucketfn) bucketfn(privdata, &t1->table[v & m1]);
              de = t1->table[v & m1];
              while (de) {
                  next = de->next;
                  fn(privdata, de);
                  de = next;
              }
  
              /* Increment the reverse cursor not covered by the smaller mask.*/
              v |= ~m1;
              v = rev(v);
              v++;
              v = rev(v);
  
              /* Continue while bits covered by mask difference is non-zero */
          } while (v & (m0 ^ m1));
      }
  
      /* undo the ++ at the top */
      d->iterators--;
  
      return v;
  }
  ```

  刚开始函数的输入参数**v**为0，每次只迭代一个桶(及其链表节点)，并返回下次调用该函数需要输入的**v**。如果返回的v值为0代表迭代结束。**这里v的值并不是普通的自增1，而是先位反转(即二进制位逆序)后加1，再位反转回来。**这样做的好处是即使哈希表的大小改变了，v值不用重新计算，因为我们首先改变的是v的高二进制位。
  
  这里使用的位反转算法为(来自http://graphics.stanford.edu/~seander/bithacks.html#ReverseParallel) :
  
  ```c
  static unsigned long rev(unsigned long v) {
      unsigned long s = CHAR_BIT * sizeof(v); // bit size; must be power of 2
      unsigned long mask = ~0UL;
      while ((s >>= 1) > 0) {
          mask ^= (mask << s);
          v = ((v >> s) & mask) | ((v << s) & ~mask);
      }
      return v;
  }
  
  ```
  
  以32位的unsigned long类型数0x12345678为例，大致流程为：先将高16位与低16位反转，即
  
  0x12345678 -> 0x56781234；在将两部分的高8位与低8位反转，即 0x56781234 -> 0x78563412
  
  ... 反转的粒度不断减小，从16，8，4，2，1，最终这个数的二进制位就反转了！
  
  

