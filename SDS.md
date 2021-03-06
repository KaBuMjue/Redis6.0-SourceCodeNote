Redis使用SDS(Simple Dynamic String)来实现字符串对象，相关文件为*sdsalloc.h、sds.h、sds.c*。

## sdsalloc.h
只是"重命名"，仅有三行代码：

```c
#define s_malloc zmalloc
#define s_realloc zrealloc
#define s_free zfree
```

## sds.h重要代码
sds.h中定义了5种sdshdr(SDS头部)类型，sdshdr5、sdshdr8、
sdshdr16、sdshdr32、sdshdr64。其中sdshdr5并不使用，其它类型适用不同长度的字符串，其定义如下：

```c
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```
struct定义中的__attribute__((__packed__))是用来禁止字节对齐的，让struct的数据成员能够紧密排列在一起，具体原因稍后会讲。
sdshdr(除了sdshdr5)的数据成员有**len**(表示该SDS已使用的buf字节数，范围由具体的sdshdr指定，见定义)、**alloc**（表示该SDS分配的buf字节数，范围由具体的sdshdr指定）、**flags**（无符号字符类型，低三位表示该SDS的类型，高五位未使用）、**buf**（字符数组，存放内容）。
* sds.h中定义了一些实用的方法:
    sdslen用来获得字符串长度，即sdshdr的len属性。
```c
static inline size_t sdslen(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return SDS_TYPE_5_LEN(flags);
        case SDS_TYPE_8:
            return SDS_HDR(8,s)->len;
        case SDS_TYPE_16:
            return SDS_HDR(16,s)->len;
        case SDS_TYPE_32:
            return SDS_HDR(32,s)->len;
        case SDS_TYPE_64:
            return SDS_HDR(64,s)->len;
    }
    return 0;
}
```
参数s的类型sds其实就是char\*,指向sdshdr的buf, 前文说的**禁止struct字节对齐**在这就派上了用场，
通过s[-1]我们就能获得flags属性，再与SDS_TYPE_MASK相与(值为7）获得flags的低三位，即该SDS类型。
而宏SDS_HDR能够获得sdshdr类型的指针，并以此获得len属性。定义为:
 `#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))`
这里的##是预处理连接符，将其左右两端的符号连接起来。因为sdshdr的内存布局为：
                   | len | alloc | flags | buf |
而sizeof(sdshdrT)的值为len+alloc+flags所占内存大小，**这里注意buf是空数组，而且不会转为指针，所以sizeof(buf)为0！**这样通过s（即buf指针）减去sizeof(sdshdrT）就能获得指向sdshdrT的指针。
还有个与SDS_HDR类似的宏——SDS_HDR_VAR
`#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));`

  sdsavail用来获得可用空间，即alloc - len
```c
static inline size_t sdsavail(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5: {
            return 0;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            return sh->alloc - sh->len;
        }
    }
    return 0;
}
```
sds.h中还定义了sdsalloc、sdssetlen、sdssetalloc等函数用来获得、设置相应的属性，但实现与前面两个函数类似不再赘述。

## sds.c重要代码
* sdsnewlen--创建一个新SDS对象
```c
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen);
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */

    sh = s_malloc(hdrlen+initlen+1);
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    s = (char*)sh+hdrlen;
    fp = ((unsigned char*)s)-1;
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    return s;
}
```
新创建的SDS的数据区(即buf)长度由参数initlen指定。 sdsReqType是根据initlen的大小来确定SDS的类型,因为sdshdr5不使用，所以使用TYPE_5的场景一律使用TYPE_8。 sdsHdrSize用来获得对应sdshdr的大小hdrlen,即sizeof(struct sdshdrT)。  随后用s_malloc（即zmalloc）来分配内存，大小为hdrlen+initlen+1，加1是因为**SDS和C风格字符串一样，以'\0'结尾。**这样有时候也能调用标准库中的字符串函数，不用实现SDS版本。  如果参数init为SDS_NOINIT(*const char *SDS_NOINIT = "SDS_NOINIT"*)，表示不初始化内存并将init设为NULL，否则如果init为NULL就初始化内存为0。  之后就是根据类型，修改sdshdr的数据成员。 如果init不为NULL且initlen大于0，则根据init初始化buf区域。

* sdsnew -- 根据C风格的字符串创建一个SDS字符串
```c
sds sdsnew(const char *init) {
    size_t initlen = (init == NULL) ? 0 : strlen(init);
    return sdsnewlen(init, initlen);
}
```
其实内部就是调用了sdsnewlen函数，但这样做方便了上层调用者的使用。

* sdsfree -- 释放SDS字符串的空间
```c
void sdsfree(sds s) {
    if (s == NULL) return;
    s_free((char*)s-sdsHdrSize(s[-1]));
}
```
* sdsupdatelen -- 更新SDS字符串长度(即sdshdr的len属性)
```c
void sdsupdatelen(sds s) {
    size_t reallen = strlen(s);
    sdssetlen(s, reallen);
}
```
关于这个函数的使用，源码注释中给了一个例子：
```c
s = sdsnew("foobar"); 
s[2] = '\0'; 
printf("%d\n",sdslen(s));
````
按C风格字符串的话，s的长度应该为2，但实际上s是一个SDS风格的字符串，它内部有一个len属性表示当前s的长度，所以实际s的长度并未修改，输出仍为6。而sdsupdatelen就非常适合这个场景，它能够正确更新s的len属性，只需要在上面例子的第二、三句之间加上 `sdsupdatelen(s);`。

* sdsclear -- 清空SDS字符串
```c
void sdsclear(sds s) {
    sdssetlen(s, 0);
    s[0] = '\0';
}
```
这里的清空只是将len属性设为0，**并未释放任何空间**！*C++STLvector的clear方法也是如此*

* sdsMakeRoomFor -- 为新数据扩充空间（如果需要）
```c++
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    size_t avail = sdsavail(s);
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;

    /* Return ASAP if there is enough space left. */
    if (avail >= addlen) return s;

    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    newlen = (len+addlen);
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;

    type = sdsReqType(newlen);

    /* Don't use type 5: the user is appending to the string and type 5 is
     * not able to remember empty space, so sdsMakeRoomFor() must be called
     * at every appending operation. */
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    hdrlen = sdsHdrSize(type);
    if (oldtype==type) {
        newsh = s_realloc(sh, hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        /* Since the header size changes, need to move the string forward,
         * and can't use realloc */
        newsh = s_malloc(hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, newlen);
    return s;
}
```
如果可用空间avail大于addlen，直接返回(*ASAP:as soon as possible*)。如果新长度newlen小于SDS_MAX_PREALLOC(1024*1024,即1MB），将newlen乘2，否则将newlen加上SDS_MAX_PREALLOC。这样做是为了预留一些空间方便下次使用，节省时间。如果扩容后的SDS字符串类型不需要改变，调用realloc在原位置扩容；如果需要改变则需重新分配内存，因为sdshdr的大小改变了。最后更新alloc属性。

* sdsRemoveFreeSpace -- 释放空闲空间
```c++
sds sdsRemoveFreeSpace(sds s) {
    void *sh, *newsh;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen, oldhdrlen = sdsHdrSize(oldtype);
    size_t len = sdslen(s);
    size_t avail = sdsavail(s);
    sh = (char*)s-oldhdrlen;

    /* Return ASAP if there is no space left. */
    if (avail == 0) return s;

    /* Check what would be the minimum SDS header that is just good enough to
     * fit this string. */
    type = sdsReqType(len);
    hdrlen = sdsHdrSize(type);

    /* If the type is the same, or at least a large enough type is still
     * required, we just realloc(), letting the allocator to do the copy
     * only if really needed. Otherwise if the change is huge, we manually
     * reallocate the string to use the different header type. */
    if (oldtype==type || type > SDS_TYPE_8) {
        newsh = s_realloc(sh, oldhdrlen+len+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+oldhdrlen;
    } else {
        newsh = s_malloc(hdrlen+len+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, len);
    return s;
}
```
如果原先的SDS类型oldtype和调整后的类型type一致，或者type大于TYPE_8，就在原有的内存上调用realloc；否则重新开辟一块内存，并释放原有内存。最后更新alloc属性。

* sdscatlen -- 附加字符串到SDS尾
```c
sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s);

    s = sdsMakeRoomFor(s,len);
    if (s == NULL) return NULL;
    memcpy(s+curlen, t, len);
    sdssetlen(s, curlen+len);
    s[curlen+len] = '\0';
    return s;
}
```
这里就用到了sdsMakeRoomFor函数，避免了SDS尾部空余空间不足导致访问越界的情况。

* sdsll2str -- 将number --> string
```c++
int sdsll2str(char *s, long long value) {
    char *p, aux;
    unsigned long long v;
    size_t l;

    /* Generate the string representation, this method produces
     * an reversed string. */
    v = (value < 0) ? -value : value;
    p = s;
    do {
        *p++ = '0'+(v%10);
        v /= 10;
    } while(v);
    if (value < 0) *p++ = '-';

    /* Compute length and add null term. */
    l = p-s;
    *p = '\0';

    /* Reverse the string. */
    p--;
    while(s < p) {
        aux = *s;
        *s = *p;
        *p = aux;
        s++;
        p--;
    }
    return l;
}
```
返回值为转换后s的长度，这个函数是用来将帮助实现将整型数据存放在SDS中，比如下面这个:
```c++
#define SDS_LLSTR_SIZE 21
sds sdsfromlonglong(long long value) {
    char buf[SDS_LLSTR_SIZE];
    int len = sdsll2str(buf,value);

    return sdsnewlen(buf,len);
}
```



## 总结
* Redis没有直接使用C风格的字符串，而是自己定义了一个SDS(Simple Dynamic String)类型，该类型不仅包含字节数组外，还有len(字节数组有效长度), alloc(字节数组总长度)属性，便于实现其它对SDS进行操作的函
* SDS的字节数组是遵循C风格的，以'\0'结尾，这样可以调用一些标准库的函数，如printf。
* SDS还是**二进制安全的**，C风格字符串里不能包含空字符，那会被当作是结尾的标志。但SDS的字节数组是使用len属性来判断是否到达结尾的，SDS的API不会对字符串内容作任何假设、过滤，其内部调用的都是像memcpy,memmove这样直接操纵内存的函数。
* SDS具有**空间预分配**以及**惰性删除**的特点。