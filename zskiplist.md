*跳跃表(skiplist)是一种有序数据结构，可看成多级链表。跳跃表的查找操作平均复杂度为O(logn)，最坏情况为O(n)，大多数情况下跳跃表的效率可与平衡树媲美，而且跳跃表的实现更加简单，是代替平衡树的一个很好的选择。Redis跳跃表的设计理念基于William Pugh的https://www.epaperpress.com/sortsearch/download/skiplist.pdf不了解跳跃表的务必看看这篇论文，不然源码还是有些难以理解的。Redis跳跃表数据结构在server.h中定义，相关函数在t_zset.c中实现。*



## 数据结构

* zskipklistNode  -- 跳跃表节点

  ```c
  typedef struct zskiplistNode {
      sds ele;		// 数据
      double score;		// 分值，用来排序
      struct zskiplistNode *backward;		// 指向前一节点
      struct zskiplistLevel {
          struct zskiplistNode *forward;	//指向表尾方向的下一节点
          unsigned long span;		// forward指向的节点与当前节点的距离，即跨度
      } level[];
  } zskiplistNode;
  ```




* zskiplist   -- 跳跃表

  ```c
  typedef struct zskiplist {
      struct zskiplistNode *header, *tail;	//表头、表尾节点
      unsigned long length;	//跳跃表所含节点数
      int level;				//跳跃表中层数最大的节点的层数
  } zskiplist;
  ```





## 函数实现


* zslCreateNode   -- 创建跳跃表节点

  ```c
  zskiplistNode *zslCreateNode(int level, double score, sds ele) {
      zskiplistNode *zn =
          zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
      zn->score = score;
      zn->ele = ele;
      return zn;
  }
  ```

  创建一个给定层数(level)、分值(score)、数据(ele)的节点。
  
  

* zslCreate   -- 创建一个新跳跃表

  ```c
  zskiplist *zslCreate(void) {
      int j;
      zskiplist *zsl;
  
      zsl = zmalloc(sizeof(*zsl));
      zsl->level = 1;
      zsl->length = 0;
      zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
      for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
          zsl->header->level[j].forward = NULL;
          zsl->header->level[j].span = 0;
      }
      zsl->header->backward = NULL;
      zsl->tail = NULL;
      return zsl;
  }
  ```

  ZSKIPLIST_MAXLEVEL宏定义为**32**，表明Redis的跳跃表支持的最大层次为32。注意该函数将头节点的层次设为最大，并且每层的前向指针为NULL，这与William Pugh的论文实现一致。



* zslFreeNode   -- 释放跳跃表节点

  ```c
  void zslFreeNode(zskiplistNode *node) {
      sdsfree(node->ele);	//注意还要释放ele
      zfree(node);
  }
  ```

  

* zslFree   -- 释放跳跃表

  ```c
  void zslFree(zskiplist *zsl) {
      zskiplistNode *node = zsl->header->level[0].forward, *next;
  
      zfree(zsl->header);
      while(node) {	//类似单链表的释放
          next = node->level[0].forward;
          zslFreeNode(node);
          node = next;
      }
      zfree(zsl);
  }
  ```



* zslRandomLevel   -- 获得随机层次

  ```c
  int zslRandomLevel(void) {
      int level = 1;
      while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
          level += 1;
      return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
  }
  ```

  ZSKIPLIST_P宏定义为0.25，所以level选取到高层次的概率很小(论文里选取的是0.5).

