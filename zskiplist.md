*跳跃表(skiplist)是一种有序数据结构，可看成多级链表。跳跃表的查找操作平均复杂度为O(logn)，最坏情况为O(n)，大多数情况下跳跃表的效率可与平衡树媲美，而且跳跃表的实现更加简单，是代替平衡树的一个很好的选择。Redis跳跃表在*server.h中定义



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

  