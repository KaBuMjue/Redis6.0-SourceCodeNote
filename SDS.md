Redis使用SDS(Simple Dynamic String)来实现字符串对象，相关文件为sdsalloc.h、sds.h、sds.c。

##sdsalloc.h
只是"重命名"，仅有三行代码：
```
#define s_malloc zmalloc
#define s_realloc zrealloc
#define s_free zfree
```

##sds.h重要代码
sds.h中定义了5种sdshdr(SDS头部)类型，sdshdr5、sdshdr8、
sdshdr16、sdshdr32、sdshdr64。其中sdshdr5并不使用，其它类型适用不同长度的字符串，其定义如下：
```
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
```
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
```
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
```
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