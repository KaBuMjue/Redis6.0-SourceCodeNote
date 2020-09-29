*跳跃表(skiplist)是一种有序数据结构，可看成多级链表。跳跃表的查找操作平均复杂度为O(logn)，最坏情况为O(n)，大多数情况下跳跃表的效率可与平衡树媲美，而且跳跃表的实现更加简单，是代替平衡树的一个很好的选择。Redis跳跃表在*server.h中定义



* zskipklistNode  -- 跳跃表节点

  ```c
  typedef struct zskiplistNode {
      sds ele;
      double score;
      struct zskiplistNode *backward;
      struct zskiplistLevel {
          struct zskiplistNode *forward;
          unsigned long span;
      } level[];
  } zskiplistNode;
  ```

  