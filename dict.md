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
  
  *Redis默认使用的哈希函数是SipHash，在siphash.c定义。*
  
  接下来就先看看SipHash的实现原理。
  
  
  
  
  
  

