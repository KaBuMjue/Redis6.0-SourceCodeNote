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



Redis还定义了链表迭代器listIter，即可正向迭代，也可反向迭代。

```c
typedef struct listIter {
    listNode *next;	//该迭代器的下一个节点
    int direction;	//迭代方向， 0为从头开始正向，1为从尾开始反向
} listIter;
```



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



* listInsertNode  -- 插入节点到指定节点前/后

  ```c
  list *listInsertNode(list *list, listNode *old_node, void *value, int after) {
      listNode *node;
  
      if ((node = zmalloc(sizeof(*node))) == NULL)
          return NULL;
      node->value = value;
      if (after) {	//插入到old_node后
          node->prev = old_node;
          node->next = old_node->next;
          if (list->tail == old_node) {	//注意插入末尾的情况
              list->tail = node;
          }
      } else {		//插入到old_node前
          node->next = old_node;
          node->prev = old_node->prev;
          if (list->head == old_node) {	//注意插入头部的情况
              list->head = node;
          }
      }
      if (node->prev != NULL) {
          node->prev->next = node;
      }
      if (node->next != NULL) {
          node->next->prev = node;
      }
      list->len++;
      return list;
  }
  ```



* listGetIterator   -- 获取链表的迭代器

  ```c
  listIter *listGetIterator(list *list, int direction)
  {
      listIter *iter;
  
      if ((iter = zmalloc(sizeof(*iter))) == NULL) return NULL;
      if (direction == AL_START_HEAD)		//如果为direction为AL_START_HEAD(0)，从链表头开始
          iter->next = list->head;		//下一个节点为链表头
      else
          iter->next = list->tail;		//否则从链表尾开始
      iter->direction = direction;		//下一个节点为链表尾
      return iter;
  }
  ```

   

* listNext   -- 获取链表迭代器的下一个节点

```c
listNode *listNext(listIter *iter)
{
    listNode *current = iter->next;

    if (current != NULL) {
        if (iter->direction == AL_START_HEAD)
            iter->next = current->next;		//正向迭代
        else
            iter->next = current->prev;		//反向迭代
    }
    return current;
}
```

这个函数的常见使用方式为：

```c
iter = listGetIterator(list,<direction>);
while ((node = listNext(iter)) != NULL) {
	doSomethingWith(listNodeValue(node));
}
```



* listDup   -- 复制链表

  ```c
  list *listDup(list *orig)
  {
      list *copy;
      listIter iter;
      listNode *node;
  
      if ((copy = listCreate()) == NULL)
          return NULL;
      copy->dup = orig->dup;
      copy->free = orig->free;
      copy->match = orig->match;
      listRewind(orig, &iter);	//将iter设为指向orig头部的迭代器
      while((node = listNext(&iter)) != NULL) {	//获取orig下一个节点
          void *value;
  
          if (copy->dup) {
              value = copy->dup(node->value);
              if (value == NULL) {
                  listRelease(copy);	//如果失败，释放已分配的新链表
                  return NULL;
              }
          } else
              value = node->value;
          if (listAddNodeTail(copy, value) == NULL) {	//添加节点至新链表尾部
              listRelease(copy);
              return NULL;
          }
      }
      return copy;
  }
  ```

  

* listSearchKey   -- 查找key对应的节点是否在链表中

```c
listNode *listSearchKey(list *list, void *key)
{
    listIter iter;
    listNode *node;

    listRewind(list, &iter);	//将iter设为指向orig头部的迭代器
    while((node = listNext(&iter)) != NULL) {
        if (list->match) {	//如果设置了match函数，使用match来对比节点值和key
            if (list->match(node->value, key)) {
                return node;
            }
        } else {		//没有设置match函数，key直接与节点值相比
            if (key == node->value) {
                return node;	
            }
        }
    }
    return NULL;	//没找到，返回NULL
}
```



* listJoin   -- 将O链表中的元素全部**"移动"**到L链表中

  ```c
  void listJoin(list *l, list *o) {
      if (o->head)
          o->head->prev = l->tail;
  
      if (l->tail)
          l->tail->next = o->head;
      else	//如果L链表为空
          l->head = o->head;
  
      if (o->tail) l->tail = o->tail;
      l->len += o->len;
  
      /* Setup other as an empty list. */
      o->head = o->tail = NULL;
      o->len = 0;
  }
  
  ```



## 总结

Redis实现的链表是**双向无环**的，list数据结构中包含头节点指针与尾节点指针，还有长度属性，所以只需要O(1)时间去获取它们，list还具有**多态性**，能够根据不同类型的节点值设置不同的dup、free、match节点值相关函数。