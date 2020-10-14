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



* zslInsert   -- 插入一个新节点

  ```c
  zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
      zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
      unsigned int rank[ZSKIPLIST_MAXLEVEL];
      int i, level;
  
      serverAssert(!isnan(score));	// 判断score是否为合法数字
      x = zsl->header;
      for (i = zsl->level-1; i >= 0; i--) {
          /* store rank that is crossed to reach the insert position */
          rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
          while (x->level[i].forward &&
                  (x->level[i].forward->score < score ||
                      (x->level[i].forward->score == score &&
                      sdscmp(x->level[i].forward->ele,ele) < 0)))
          {
              rank[i] += x->level[i].span;
              x = x->level[i].forward;
          }
          update[i] = x;
      }
      /* we assume the element is not already inside, since we allow duplicated
       * scores, reinserting the same element should never happen since the
       * caller of zslInsert() should test in the hash table if the element is
       * already inside or not. */
      level = zslRandomLevel();
      if (level > zsl->level) {
          for (i = zsl->level; i < level; i++) {
              rank[i] = 0;
              update[i] = zsl->header;
              update[i]->level[i].span = zsl->length;
          }
          zsl->level = level;
      }
      x = zslCreateNode(level,score,ele);
      for (i = 0; i < level; i++) {
          x->level[i].forward = update[i]->level[i].forward;
          update[i]->level[i].forward = x;
  
          /* update span covered by update[i] as x is inserted here */
          x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
          update[i]->level[i].span = (rank[0] - rank[i]) + 1;
      }
  
      /* increment span for untouched levels */
      for (i = level; i < zsl->level; i++) {
          update[i]->level[i].span++;
      }
  
      x->backward = (update[0] == zsl->header) ? NULL : update[0];
      if (x->level[0].forward)
          x->level[0].forward->backward = x;
      else
          zsl->tail = x;
      zsl->length++;
      return x;
  }
  ```

  插入算法与论文的实现原理基本一致，但允许重复的分值出现，还有对卫星数据的比较:

  `sdscmp(x->level[i].forward->ele,ele) < 0)`



* zslDeleteNode   -- 删除节点（为辅助函数）

  ```c
  void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
      int i;
      for (i = 0; i < zsl->level; i++) {
          if (update[i]->level[i].forward == x) {
              update[i]->level[i].span += x->level[i].span - 1;
              update[i]->level[i].forward = x->level[i].forward;
          } else {
              update[i]->level[i].span -= 1;
          }
      }
      if (x->level[0].forward) {
          x->level[0].forward->backward = x->backward;
      } else {
          zsl->tail = x->backward;
      }
      while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
          zsl->level--;
      zsl->length--;
  }
  ```

  