### 前言

在上一篇文章[《Redis列表实现原理之ziplist结构》](https://mp.weixin.qq.com/s?__biz=MzI2MDQzMTU2MA==&mid=2247484130&idx=1&sn=f9677e06b5fccfd0696a68c431b3f013&chksm=ea688945dd1f00533ef41bfc97041704a7d2a4318ffa4387a03f4b52e9d959ed005ab930599f&token=1987875319&lang=zh_CN&scene=21#wechat_redirect)，我们分析了ziplist结构如何使用一块完整的内存存储列表数据。
同时也提出了一个问题：如果链表很长，ziplist中每次插入或删除节点时都需要进行大量的内存拷贝，这个性能是无法接受的。
本文分析：quicklist结构如何解决这个问题并实现Redis的列表类型。

quicklist的设计思想很简单，将一个长ziplist拆分为多个短ziplist，避免插入或删除元素时导致大量的内存拷贝。 
ziplist存储数据的形式更类似于数组，而quicklist是真正意义上的链表结构，它由quicklistNode节点链接而成，在quicklistNode中使用ziplist存储数据。

> 提示：本文以下代码如无特殊说明，均位于quicklist.h/quicklist.c中。
> 本文以下说的“节点”，如无特殊说明，都指quicklistNode节点，而不是ziplist中的节点。

### 定义

quicklistNode的定义如下：

```c
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;
    unsigned int sz;             
    unsigned int count : 16;  
    unsigned int encoding : 2; 
    unsigned int container : 2; 
    unsigned int recompress : 1;
    unsigned int attempted_compress : 1;
    unsigned int extra : 10; 
} quicklistNode;
```

- prev、next：指向前驱节点，后驱节点。
- zl：ziplist，负责存储数据。
- sz：ziplist占用的字节数。
- count：ziplist的元素数量。
- encoding：2代表节点已压缩，1代表没有压缩。
- container：目前固定为2，代表使用ziplist存储数据。
- recompress：1代表暂时解压（用于读取数据等），后续需要时再将其压缩。
- extra：预留属性，暂未使用。

当链表很长时，中间节点数据访问频率较低。这时Redis会将中间节点数据进行压缩，进一步节省内存空间。Redis采用是无损压缩算法—LZF算法。 
压缩后的节点定义如下：

```c
typedef struct quicklistLZF {
    unsigned int sz;
    char compressed[];
} quicklistLZF;
```

- sz：压缩后的ziplist大小。
- compressed：存放压缩后的ziplist字节数组。

quicklist的定义如下：

```c
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        
    unsigned long len;          
    int fill : QL_FILL_BITS;              
    unsigned int compress : QL_COMP_BITS;
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;
```

- head、tail：指向头节点、尾节点。
- count：所有节点的ziplist的元素数量总和。
- len：节点数量。
- fill：16bit，用于判断节点ziplist是否已满。
- compress：16bit，存放节点压缩配置。

quicklist的结构如图2-5所示。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Of81vjDNtAwaw1XQ5uUeSJw4kWtd8lYGacKZT5L2j2qkPxEt2YoxfPAoaDYu8wOfwnQxWDWrmc2PeEK5J008ZQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)图2-5

### 操作分析

#### 插入元素到quicklist头部

```c
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_head = quicklist->head;
    // [1]
    if (likely(
            _quicklistNodeAllowInsert(quicklist->head, quicklist->fill, sz))) {
        // [2]
        quicklist->head->zl =
            ziplistPush(quicklist->head->zl, value, sz, ZIPLIST_HEAD);
        // [3]
        quicklistNodeUpdateSz(quicklist->head);
    } else {
        // [4]
        quicklistNode *node = quicklistCreateNode();
        node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);

        quicklistNodeUpdateSz(node);
        _quicklistInsertNodeBefore(quicklist, quicklist->head, node);
    }
    quicklist->count++;
    quicklist->head->count++;
    return (orig_head != quicklist->head);
}
```

##### 参数说明：

- value、sz：插入元素的内容与大小。 

##### 操作过程

【1】判断head节点ziplist是否已满，_quicklistNodeAllowInsert函数中根据quicklist.fill属性判断节点是否已满。 
【2】head节点未满，直接调用ziplistPush函数，插入元素到ziplist中。 
【3】更新quicklistNode.sz属性。 
【4】head节点已满，创建一个新节点，将元素插入新节点的ziplist中，再将该节点头插入quicklist中。

#### 在quicklist的指定位置插入元素：

```c
REDIS_STATIC void _quicklistInsert(quicklist *quicklist, quicklistEntry *entry,
                                   void *value, const size_t sz, int after) {
    int full = 0, at_tail = 0, at_head = 0, full_next = 0, full_prev = 0;
    int fill = quicklist->fill;
    quicklistNode *node = entry->node;
    quicklistNode *new_node = NULL;
    ...
    // [1]
    if (!_quicklistNodeAllowInsert(node, fill, sz)) {
        full = 1;
    }

    if (after && (entry->offset == node->count)) {
        at_tail = 1;
        if (!_quicklistNodeAllowInsert(node->next, fill, sz)) {
            full_next = 1;
        }
    }

    if (!after && (entry->offset == 0)) {
        at_head = 1;
        if (!_quicklistNodeAllowInsert(node->prev, fill, sz)) {
            full_prev = 1;
        }
    }
    // [2]
    ...
}
```

##### 参数说明：

- entry：quicklistEntry结构，quicklistEntry.node指定元素插入的quicklistNode节点，quicklistEntry.offset指定插入ziplist的索引位置。
- after：是否在quicklistEntry.offset之后插入。

##### 操作过程 

【1】根据参数设置以下标志。

- full：待插入节点ziplist是否已满。
- at_tail：是否ziplist尾插。
- at_head：是否ziplist头插。
- full_next：后驱节点是否已满。
- full_prev：前驱节点是否已满。

> 提示：头插指插入链表头部，尾插指插入链表尾部。

【2】根据上面的标志进行处理，代码较烦琐，这里不再列出。 
这里的执行逻辑如表2-2所示。

| 条件                                                         | 条件说明                                                     | 处理方式                                                     |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| !full && after                                               | 待插入节点未满，ziplist尾插                                  | 再次检查ziplist插入位置是否存在后驱元素，如果不存在则调用ziplistPush函数插入元素（更快），否则调用ziplistInsert插入元素 |
| !full && !after                                              | 待插入节点未满，非ziplist尾插                                | 调用ziplistInsert函数插入元素                                |
| full && at_tail && node -> next && !full_next && after       | 待插入节点已满，尾插，后驱节点未满                           | 将元素插入后驱节点ziplist中                                  |
| full && at_head && node -> prev && !full_prev && !after      | 待插入节点已满，ziplist头插，前驱节点未满                    | 将元素插入前驱节点ziplist中                                  |
| full && ((at_tail && node -> next && full_next && after) \|\|(at_head && node->prev && full_prev && !after)) | 满足以下条件：(1)待插入节点已满 (2)尾插且后驱节点已满，或者头插且前驱节点已满 | 构建一个新节点，将元素插入新节点，并根据after参数将新节点插入quicklist中 |
| full                                                         | 待插入节点已满，并且在节点ziplist中间插入                    | 将插入节点的数据拆分到两个节点中，再插入拆分后的新节点中     |

我们只看最后一种场景的实现：

```c
    // [1]
    quicklistDecompressNodeForUse(node);
    // [2]
    new_node = _quicklistSplitNode(node, entry->offset, after);
    new_node->zl = ziplistPush(new_node->zl, value, sz,
                                after ? ZIPLIST_HEAD : ZIPLIST_TAIL);
    new_node->count++;
    quicklistNodeUpdateSz(new_node);
    // [3]
    __quicklistInsertNode(quicklist, node, new_node, after);
    // [4]
    _quicklistMergeNodes(quicklist, node);
```

【1】如果节点已压缩，则解压节点。 
【2】从插入节点中拆分出一个新节点，并将元素插入新节点中。 
【3】将新节点插入quicklist中。 
【4】尝试合并节点。_quicklistMergeNodes尝试执行以下操作：

- 将node->prev->prev合并到node->prev。
- 将node->next合并到node->next->next。
- 将node->prev合并到node。
- 将node合并到node->next。
  合并条件：如果合并后节点大小仍满足quicklist.fill参数要求，则合并节点。
  这个场景处理与B+树的节点分裂合并有点相似。

### quicklist常用的函数

| 函数                                 | 作用                             |
| :----------------------------------- | :------------------------------- |
| quicklistCreate、quicklistNew        | 创建一个空的quicklist            |
| quicklistPushHead，quicklistPushTail | 在quicklist头部、尾部插入元素    |
| quicklistIndex                       | 查找给定索引的quicklistEntry节点 |
| quicklistDelEntry                    | 删除给定的元素                   |

### **配置说明**

- list-max-ziplist-size：配置server.list_max_ziplist_size属性，该值会赋值给quicklist.fill。取正值，表示quicklist节点的ziplist最多可以存放多少个元素。例如，配置为5，表示每个quicklist节点的ziplist最多包含5个元素。取负值，表示quicklist节点的ziplist最多占用字节数。这时，它只能取-1到-5这五个值（默认值为-2），每个值的含义如下： 
  -5：每个quicklist节点上的ziplist大小不能超过64 KB。 
  -4：每个quicklist节点上的ziplist大小不能超过32 KB。 
  -3：每个quicklist节点上的ziplist大小不能超过16 KB。 
  -2：每个quicklist节点上的ziplist大小不能超过8 KB。 
  -1：每个quicklist节点上的ziplist大小不能超过4 KB。
- list-compress-depth：配置server.list_compress_depth属性，该值会赋值给quicklist.compress。 
  0：表示节点都不压缩，Redis的默认配置。 
  1：表示quicklist两端各有1个节点不压缩，中间的节点压缩。 
  2：表示quicklist两端各有2个节点不压缩，中间的节点压缩。 
  3：表示quicklist两端各有3个节点不压缩，中间的节点压缩。 
  以此类推。

### 编码

ziplist由于结构紧凑，能高效使用内存，所以在Redis中被广泛使用，可用于保存用户列表、散列、有序集合等数据。 
列表类型只有一种编码格式OBJ_ENCODING_QUICKLIST，使用quicklist存储数据（redisObject.ptr指向quicklist结构）。列表类型的实现代码在t_list.c中，读者可以查看源码了解实现更多细节。

### **总结**

- ziplist是一种结构紧凑的数据结构，使用一块完整内存存储链表的所有数据。
- ziplist内的元素支持不同的编码格式，以最大限度地节省内存。
- quicklist通过切分ziplist来提高插入、删除元素等操作的性能。
- 链表的编码格式只有OBJ_ENCODING_QUICKLIST。