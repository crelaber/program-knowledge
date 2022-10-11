Redis是一个键值对数据库（key-value DB），下面是一个简单的Redis的命令：

```shell
> SET msg "hello wolrd"
```

该命令将键“msg”、值“hello wolrd”这两个字符串保存到Redis数据库中。 
本章分析Redis如何在内存中保存这些字符串。

## redisObject

Redis中的数据对象server.h/redisObject是Redis对内部存储的数据定义的抽象类型，在深入分析Redis数据类型前，我们先了解redisObject，它的定义如下：

```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS;
    int refcount;
    void *ptr;
} robj;
```

- type：数据类型。
- encoding：编码格式，即存储数据使用的数据结构。同一个类型的数据，Redis会根据数据量、占用内存等情况使用不同的编码，最大限度地节省内存。
- refcount，引用计数，为了节省内存，Redis会在多处引用同一个redisObject。
- ptr：指向实际的数据结构，如sds，真正的数据存储在该数据结构中。
- lru：24位，LRU时间戳或LFU计数。

redisObject负责装载Redis中的所有键和值。redisObject.ptr指向真正存储数据的数据结构，redisObject .refcount、redisObject.lru等属性则用于管理数据（数据共享、数据过期等）。

> 提示：type、encoding、lru使用了C语言中的位段定义，这3个属性使用同一个unsigned int的不同bit位。这样可以最大限度地节省内存。

Redis定义了以下数据类型和编码，如表1-1所示。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Of81vjDNtAzfe3Zvj112Zj7YUXefTQXusrZZHfdjvF0E0Y147qhEjHFldfssM7vCUW64jue4GD1C0V3JmxrMMA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

本书第1部分会对表1-1中前五种数据类型进行分析，最后两种数据类型会在第5部分进行分析。如果读者现在对表1-1中内容感到疑惑，则可以先带着疑问继续阅读本书。

## sds

我们知道，C语言中将空字符结尾的字符数组作为字符串，而Redis对此做了扩展，定义了字符串类型sds（Simple Dynamic String）。 
Redis键都是字符串类型，Redis中最简单的值类型也是字符串类型，  
字符串类型的Redis值可用于很多场景，如缓存HTML片段、记录用户登录信息等。

### 定义

> 提示：本节代码如无特殊说明，均在sds.h/sds.c中。
> 对于不同长度的字符串，Redis定义了不同的sds结构体：

```c
typedef char *sds;

struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; 
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; 
    uint8_t alloc;
    unsigned char flags;
    char buf[];
};
...
```

Redis还定义了sdshdr16、sdshdr32、sdshdr64结构体。为了版面整洁，这里不展示sdshdr16、sdshdr32、sdshdr64 结构体的代码，它们与sdshdr8结构体基本相同，只是len、alloc属性使用了 uint16_t、uint32、uint64_t类型。Redis定义不同sdshdr结构体是为了针对不同长度的字符串，使用合适的len、alloc属性类型，最大限度地节省内存。

- len：已使用字节长度，即字符串长度。sdshdr5可存放的字符串长度小于32（25），sdshdr8可存放的字符串长度小于256（28），以此类推。由于该属性记录了字符串长度，所以sds可以在常数时间内获取字符串长度。Redis限制了字符串的最大长度不能超过512MB。
- alloc：已申请字节长度，即sds总长度。alloc-len为sds中的可用（空闲）空间。
- flag：低3位代表sdshdr的类型，高5位只在sdshdr5中使用，表示字符串的长度，所以sdshdr5中没有len属性。另外，由于Redis对sdshdr5的定义是常量字符串，不支持扩容，所以不存在alloc属性。
- buf：字符串内容，sds遵循C语言字符串的规范，保存一个空字符作为buf的结尾，并且不计入len、alloc属性。这样可以直接使用C语言strcmp、strcpy等函数直接操作sds。

> 提示：sdshdr结构体中的buf数组并没有指定数组长度，它是C99规范定义的柔性数组—结构体中最后一个属性可以被定义为一个大小可变的数组（该属性前必须有其他属性）。使用sizeof函数计算包含柔性数组的结构体大小，返回结果不包括柔性数组占用的内存。 
> 另外，**attribute__((__packed**))关键字可以取消结构体内的字节对齐以节省内存。

### sds构建函数

接下来看一下sds构建函数：

```c
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    // [1]
    char type = sdsReqType(initlen);
    // [2]
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    // [3]
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */

    sh = s_malloc(hdrlen+initlen+1);
    ...
    // [4]
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
        ...
    }
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    // [5]
    return s;
}
```

#### 参数说明：

- init、initlen：字符串内容、长度。 

#### 过程

【1】根据字符串长度，判断对应的sdshdr类型。 
【2】长度为0的字符串后续通常需要扩容，不应该使用sdshdr5，所以这里转换为sdshdr8。
【3】sdsHdrSize函数负责查询sdshdr结构体的长度，s_malloc函数负责申请内存空间，申请的内存空间长度为hdrlen+initlen+1，其中hdrlen为sdshdr结构体长度（不包含buf属性），initlen为字符串内容长度，最后一个字节用于存放空字符“\0”。s_malloc与C语言的malloc函数的作用相同，负责分配指定大小的内存空间。 
【4】给sdshdr属性赋值。 
SDS_HDR_VAR是一个宏，负责将sh指针转化为对应的sdshdr结构体指针。 
【5】注意，sds实际上就是char*的别名，这里返回的s指针指向sdshdr.buf属性，即字符串内容。Redis通过该指针可以直接读/写字符串数据。 

构建一个内容为“hello wolrd”的sds，其结构如图1-1所示。 

![图片](https://mmbiz.qpic.cn/mmbiz_png/Of81vjDNtAzfe3Zvj112Zj7YUXefTQXuZslKAA3U5QdqhSX3aYWGfaRH0DHdMwp5T2c4pp7qnl3JT4OmXGhFvw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



### sds扩容机制

sds的扩容机制是一个很重要的功能。

```c
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    // [1]
    size_t avail = sdsavail(s);
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;

    if (avail >= addlen) return s;
    // [2]
    len = sdslen(s);

    sh = (char*)s-sdsHdrSize(oldtype);
    newlen = (len+addlen);
    // [3]
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;

    // [4]    
    type = sdsReqType(newlen);
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;
    // [5]
    hdrlen = sdsHdrSize(type);
    if (oldtype==type) {
        newsh = s_realloc(sh, hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        newsh = s_malloc(hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    // [6]
    sdssetalloc(s, newlen);
    return s;
}
```

#### 参数说明： 
addlen：要求扩容后可用长度（alloc-len）大于该参数。 

#### 过程

【1】获取当前可用空间长度。如果当前可用空间长度满足要求，则直接返回。 
【2】sdslen负责获取字符串长度，由于sds.len中记录了字符串长度，该操作复杂度为O(1)。这里len变量为原sds字符串长度，newlen变量为新sds长度。sh指向原sds的sdshdr结构体。 
【3】预分配比参数要求多的内存空间，避免每次扩容都要进行内存拷贝操作。新sds长度如果小于SDS_MAX_PREALLOC（默认为1024×1024，单位为字节），则新sds长度自动扩容为2倍。否则，新sds长度自动增加SDS_MAX_PREALLOC。 
【4】sdsReqType(newlen)负责计算新的sdshdr类型。注意，扩容后的类型不使用sdshdr5，该类型不支持扩容操作。 
【5】如果扩容后sds还是同一类型，则使用s_realloc函数申请内存。否则，由于sds结构已经变动，必须移动整个sds，直接分配新的内存空间，并将原来的字符串内容复制到新的内存空间。s_realloc与C语言realloc函数的作用相同，负责为给定指针重新分配给定大小的内存空间。它会尝试在给定指针原地址空间上重新分配，如原地址空间无法满足要求，则分配新内存空间并复制内容。 
【6】更新sdshdr.alloc属性。

对上面“hello wolrd”的sds调用sdsMakeRoomFor(sds,64)，则生成的sds如图所示。 

![图片](https://mmbiz.qpic.cn/mmbiz_png/Of81vjDNtAzfe3Zvj112Zj7YUXefTQXuB1sbuR8icEic5AH3wPCuicyMqbkX7Pk5owPf5W4TUvkzFnrNx3IkfjPnw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

从图中可以看到，使用len记录字符串长度后，字符串中可以存放空字符。Redis字符串支持二进制安全，可以将用户的输入存储为没有任何特定格式意义的原始数据流，因此Redis字符串可以存储任何数据，比如图片数据流或序列化对象。C语言字符串将空字符作为字符串结尾的特定标记字符，它不是二进制安全的。
sds常用函数如表1-2所示。

| 函数                                  | 作用                                                  |
| :------------------------------------ | :---------------------------------------------------- |
| sdsnew，sdsempty                      | 创建sds                                               |
| sdsfree，sdsclear，sdsRemoveFreeSpace | 释放sds，清空sds中的字符串内容，移除sds剩余的可用空间 |
| sdslen                                | 获取sds字符串长度                                     |
| sdsdup                                | 将给定字符串复制到sds中，覆盖原字符串                 |
| sdscat                                | 将给定字符串拼接到sds字符串内容后                     |
| sdscmp                                | 对比两个sds字符串是否相同                             |
| sdsrange                              | 获取子字符串，不在指定范围内的字符串将被清除          |

### 编码

字符串类型一共有3种编码：

- OBJ_ENCODING_EMBSTR：长度小于或等于OBJ_ENCODING_EMBSTR_SIZE_LIMIT（44字节）的字符串。 
  在该编码中，redisObject、sds结构存放在一块连续内存块中，如图1-3所示。

  ![图片](https://mmbiz.qpic.cn/mmbiz_png/Of81vjDNtAzfe3Zvj112Zj7YUXefTQXuDcGGGNRZFLPot4dpmLNVsicGgc6nOUcrGmz6AeaCs0F5XXCEQKtLHmw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1) 

OBJ_ENCODING_EMBSTR编码是Redis针对短字符串的优化，有如下优点： 
  (1)内存申请和释放都只需要调用一次内存操作函数。 
  (2)redisObject、sdshdr结构保存在一块连续的内存中，减少了内存碎片。

- OBJ_ENCODING_RAW：长度大于OBJ_ENCODING_EMBSTR_SIZE_LIMIT的字符串，在该编码中，redisObject、sds结构存放在两个不连续的内存块中。
- OBJ_ENCODING_INT：将数值型字符串转换为整型，可以大幅降低数据占用的内存空间，如字符串“123456789012”需要占用12字节，在Redis中，会将它转化为long long类型，只占用8字节。

#### 创建编码函数

我们向Redis发送一个请求后，Redis会解析请求报文，并将命令、参数转化为redisObjec。
object.c/createStringObject函数负责完成该操作：

```c
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
```

可以看到，这里根据字符串长度，将encoding转化为OBJ_ENCODING_RAW或OBJ_ENCODING_EMBSTR的redisObject。

将参数转换为redisObject后，Redis再将redisObject存入数据库，例如：

```
> SET Introduction "Redis is an open source (BSD licensed), in-memory data structure store, used as a database, cache and message broker. "
```

Redis会将键“Introduction”、值“Redis…”转换为两个redisObject，再将redisObject存入数据库，结果如图1-4所示。 

![图片](https://mmbiz.qpic.cn/mmbiz_png/Of81vjDNtAzfe3Zvj112Zj7YUXefTQXurwDU0aTPor1ooDCtJ0s6dIG4zC5etGYEF5Dg4qGaMQDdq5KwSfUscw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

Redis中的键都是字符串类型，并使用OBJ_ENCODING_RAW、OBJ_ENCODING_ EMBSTR编码，而Redis还会尝试将字符串类型的值转换为OBJ_ENCODING_INT 编码。object.c/tryObjectEncoding函数完成该操作：

```c
robj *tryObjectEncoding(robj *o) {
    long value;
    sds s = o->ptr;
    size_t len;
    ...
    // [1]
     if (o->refcount > 1) return o;

    len = sdslen(s);
    // [2]
    if (len <= 20 && string2l(s,len,&value)) {
        // [3]
        if ((server.maxmemory == 0 ||
            !(server.maxmemory_policy & MAXMEMORY_FLAG_NO_SHARED_INTEGERS)) &&
            value >= 0 &&
            value < OBJ_SHARED_INTEGERS)
        {
            decrRefCount(o);
            incrRefCount(shared.integers[value]);
            return shared.integers[value];
        } else {
            // [4]
            if (o->encoding == OBJ_ENCODING_RAW) {
                sdsfree(o->ptr);
                o->encoding = OBJ_ENCODING_INT;
                o->ptr = (void*) value;
                return o;
            } else if (o->encoding == OBJ_ENCODING_EMBSTR) {
                // [5]
                decrRefCount(o);
                return createStringObjectFromLongLongForValue(value);
            }
        }
    }

    // [6]
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT) {
        robj *emb;

        if (o->encoding == OBJ_ENCODING_EMBSTR) return o;
        emb = createEmbeddedStringObject(s,sdslen(s));
        decrRefCount(o);
        return emb;
    }

    // [7]
    trimStringObjectIfNeeded(o);

    return o;
}
```

【1】该数据对象被多处引用，不能再进行编码操作，否则会影响其他地方的正常运行。 
【2】如果字符串长度小于或等于20，则调用string2l函数尝试将其转换为long long类型，如果成功则返回1。 
在C语言中，long long占用8字节，取值范围是-9223372036854775808～9223372036854775807，因此最多能保存长度为19的字符串转换后的数值，加上负数的符号位，一共20位。 
下面是字符串可以转换为OBJ_ENCODING_INT 编码的处理步骤。 
【3】首先尝试使用shared.integers中的共享数据，避免重复创建相同数据对象而浪费内存。shared是Redis启动时创建的共享数据集，存放了Redis中常用的共享数据。shared.integers是一个整数数组，存放了小数字0～9999，共享于各个使用场景。 
注意：如果配置了server.maxmemory，并使用了不支持共享数据的淘汰算法（LRU、LFU），那么这里不能使用共享数据，因为这时每个数据中都必须存在一个redisObjec.lru属性，这些算法才可以正常工作。 
【4】如果不能使用共享数据并且原编码格式为OBJ_ENCODING_RAW，则将redisObject.ptr原来的sds类型替换为字符串转换后的数值。 
【5】如果不能使用共享数据并且原编码格式为OBJ_ENCODING_EMBSTR，由于redisObject、sds存放在同一个内存块中，无法直接替换redisObject.ptr，所以调用createString- ObjectFromLongLongForValue函数创建一个新的redisObject，编码为OBJ_ENCODING_INT，redisObject.ptr指向long long类型或long类型。 
【6】到这里，说明字符串不能转换为OBJ_ENCODING_INT 编码，尝试将其转换为OBJ_ENCODING_EMBSTR编码。 
【7】到这里，说明字符串只能使用OBJ_ENCODING_RAW编码，尝试释放sds中剩余的可用空间。 
字符串类型的实现代码在t_string.c中，读者可以查看源码了解更多实现细节。

> 提示：server.c/redisCommandTable定义了每个Redis命令与对应的处理函数，读者可以从这里查找感兴趣的命令的处理函数。

```c
struct redisCommand redisCommandTable[] = {
    ...
    {"get",getCommand,2,
     "read-only fast @string",
     0,NULL,1,1,1,0,0,0},

    {"set",setCommand,-3,
     "write use-memory @string",
     0,NULL,1,1,1,0,0,0},
     ...
}
```

GET命令的处理函数为getCommand，SET命令的处理函数为setCommand，以此类推。

另外，我们可以通过TYPE命令查看数据对象类型，通过OBJECT ENCODING命令查看编码：

```shell
> SET msg "hello world"
OK
> TYPE msg
string
> OBJECT ENCODING  msg
"embstr"
> SET Introduction "Redis is an open source (BSD licensed), in-memory data structure store, used as a database, cache and message broker. "
OK
> TYPE Introduction
string
> OBJECT ENCODING  info
"raw"
> SET page 1
OK
> TYPE page
string
> OBJECT ENCODING  page
"int"
```

**总结：**

- Redis中的所有键和值都是redisObject变量。
- sds是Redis定义的字符串类型，支持二进制安全、扩容。
- sds可以在常数时间内获取字符串长度，并使用预分配内存机制减少内存拷贝次数。
- Redis对数据编码的主要目的是最大限度地节省内存。字符串类型可以使用OBJ_ENCODING_ RAW、OBJ_ENCODING_EMBSTR、OBJ_ENCODING_INT编码格式。