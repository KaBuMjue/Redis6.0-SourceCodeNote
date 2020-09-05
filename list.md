因为C标准库没有实现链表这种数据结构，而链表十分常用，所以Redis实现了自己的链表数据结构，相关文件为*adlist.h、adlist.c*.



## adlist.h

Redis实现的链表是**双向**的，所以链表节点除了包含数据指针外，还有指向前一个节点以及后一个节点的指针。具体定义如下：

```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;
```



然后是链表的定义：

```c
typedef struct list {
    listNode *head;		//头节点
    listNode *tail;		//尾节点
    void *(*dup)(void *ptr);	//节点值复制函数
    void (*free)(void *ptr);	//节点值释放函数
    int (*match)(void *ptr, void *key);	//节点值匹配函数
    unsigned long len;	//链表长度
} list;
```

因为链表内部有三个函数指针，所以在adlist.h中宏定义了一些函数用来操纵它们。如：

`#define listSetDupMethod(l,m)  ((l)->dup = (m)) `	设置节点值复制函数

`#define listGetDupMethod(l) ((l)->dup)`   返回正在使用的节点值复制函数



## adlist.c

* listCreate   -- 创建一个(空)链表

  ```C
  list *listCreate(void)
  {
      struct list *list;
  
      if ((list = zmalloc(sizeof(*list))) == NULL)
          return NULL;
      list->head = list->tail = NULL;
      list->len = 0;
      list->dup = NULL;
      list->free = NULL;
      list->match = NULL;
      return list;
  }
  ```

  很简单，分配完链表所需空间后，将数据成员都设为NULL（0）。

  

* listEmpty  -- 清空链表的节点，但不释放链表

  ```c
  void listEmpty(list *list)
  {
      unsigned long len;
      listNode *current, *next;
  
      current = list->head;
      len = list->len;
      while(len--) {
          next = current->next;
          if (list->free) list->free(current->value);
          zfree(current);
          current = next;
      }
      list->head = list->tail = NULL;
      list->len = 0;
  }
  ```

  如果当前使用了节点值释放函数，每次释放节点时就先调用，后再释放节点空间。

  如果要释放整个链表，使用listRelease函数（先调用listEmpty,再释放链表空间）。

  

* listAddNodeHead  -- 添加节点到链表头部

  ```c
  list *listAddNodeHead(list *list, void *value)
  {
      listNode *node;
  
      if ((node = zmalloc(sizeof(*node))) == NULL)
          return NULL;
      node->value = value;	//新建节点值为val
      if (list->len == 0) {	//注意对空链表的特殊处理！
          list->head = list->tail = node;
          node->prev = node->next = NULL;
      } else {
          node->prev = NULL;
          node->next = list->head;
          list->head->prev = node;
          list->head = node;
      }
      list->len++;
      return list;
  }
  ```

  如果该函数返回值为NULL，代表添加节点失败，链表不发生变化。

  listAddNodeTail 添加节点至链表尾部，实现方法类似，不再赘述。

