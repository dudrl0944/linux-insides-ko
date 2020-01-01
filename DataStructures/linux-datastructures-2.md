리눅스 커널의 여러 데이터 구조
================================================================================

기수(Radix) 트리
--------------------------------------------------------------------------------

이미 알고 있듯이 리눅스 커널은 다양한 데이터 구조와 알고리즘을 구현하는 다양한 라이브러리와 함수를 제공합니다.  리눅스 커널에서 제공하는 자료구조 중 하나 인 [기수(Radix) 트리](https://en.wikipedia.org/wiki/Radix_tree) 에 대해 다뤄 볼 것입니다. 

* [include/linux/radix-tree.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/radix-tree.h)
* [lib/radix-tree.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/lib/radix-tree.c)

위에 두 파일은 리눅스 커널에서 `기수(Radix) 트리`의 구현과 API에 관련된 파일입니다.

`기수(Radix) 트리`에 대해서 이야기해보자. 기수 트리는 `압축 트라이(compressed trie)`이다. [트라이(trie)](https://en.wikipedia.org/wiki/Trie)는 연관 배열의 인터페이스를 구현하고 값을 `키-값`으로 저장할 수있는 데이터 구조입니다. 키는 일반적으로 문자열이지만 모든 데이터 타입을 키로 사용할 수 있습니다. 노드(node)때문에 `트라이(trie)`는 `n-트리` 다릅니다. `트라이(trie)`의 노드는 키를 저장하지 않습니다. 대신 `트라이(trie)`의 노드는 단일 문자 레이블을 저장합니다. 주어진 노드와 관련된 키는 트리의 루트에서이 노드로 순회하여 파생됩니다. 예를 들면 다음과 같습니다

Lets talk about what a `radix tree` is. Radix tree is a `compressed trie` where a [trie](http://en.wikipedia.org/wiki/Trie) is a data structure which implements an interface of an associative array and allows to store values as `key-value`. The keys are usually strings, but any data type can be used. A trie is different from an `n-tree` because of its nodes. Nodes of a trie do not store keys; instead, a node of a trie stores single character labels. The key which is related to a given node is derived by traversing from the root of the tree to this node. For example:


```
               +-----------+
               |           |
               |    " "    |
               |           |
        +------+-----------+------+
        |                         |
        |                         |
   +----v------+            +-----v-----+
   |           |            |           |
   |    g      |            |     c     |
   |           |            |           |
   +-----------+            +-----------+
        |                         |
        |                         |
   +----v------+            +-----v-----+
   |           |            |           |
   |    o      |            |     a     |
   |           |            |           |
   +-----------+            +-----------+
                                  |
                                  |
                            +-----v-----+
                            |           |
                            |     t     |
                            |           |
                            +-----------+
```
위의 `트라이(tire)`에서 `go`와 `cat` `키(key)`인 것을 알 수 있습니다.
So in this example, we can see the `trie` with keys, `go` and `cat`. The compressed trie or `radix tree` differs from `trie` in that all intermediates nodes which have only one child are removed.

Radix tree in linux kernel is the data structure which maps values to integer keys. It is represented by the following structures from the file [include/linux/radix-tree.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/radix-tree.h):

```C
struct radix_tree_root {
         unsigned int            height;
         gfp_t                   gfp_mask;
         struct radix_tree_node  __rcu *rnode;
};
```

This structure presents the root of a radix tree and contains three fields:

* `height`   - height of the tree;
* `gfp_mask` - tells how memory allocations will be performed;
* `rnode`    - pointer to the child node.

The first field we will discuss is `gfp_mask`:

Low-level kernel memory allocation functions take a set of flags as - `gfp_mask`, which describes how that allocation is to be performed. These `GFP_` flags which control the allocation process can have following values: (`GF_NOIO` flag) means sleep and wait for memory, (`__GFP_HIGHMEM` flag) means high memory can be used, (`GFP_ATOMIC` flag) means the allocation process has high-priority and can't sleep etc.

* `GFP_NOIO` - can sleep and wait for memory;
* `__GFP_HIGHMEM` - high memory can be used;
* `GFP_ATOMIC` - allocation process is high-priority and can't sleep;

etc.

The next field is `rnode`:

```C
struct radix_tree_node {
        unsigned int    path;
        unsigned int    count;
        union {
                struct {
                        struct radix_tree_node *parent;
                        void *private_data;
                };
                struct rcu_head rcu_head;
        };
        /* For tree user */
        struct list_head private_list;
        void __rcu      *slots[RADIX_TREE_MAP_SIZE];
        unsigned long   tags[RADIX_TREE_MAX_TAGS][RADIX_TREE_TAG_LONGS];
};
```

This structure contains information about the offset in a parent and height from the bottom, count of the child nodes and fields for accessing and freeing a node. This fields are described below:

* `path` - offset in parent & height from the bottom;
* `count` - count of the child nodes;
* `parent` - pointer to the parent node;
* `private_data` - used by the user of a tree;
* `rcu_head` - used for freeing a node;
* `private_list` - used by the user of a tree;

The two last fields of the `radix_tree_node` - `tags` and `slots` are important and interesting. Every node can contains a set of slots which are store pointers to the data. Empty slots in the linux kernel radix tree implementation store `NULL`. Radix trees in the linux kernel also supports tags which are associated with the `tags` fields in the `radix_tree_node` structure. Tags allow individual bits to be set on records which are stored in the radix tree.

Now that we know about radix tree structure, it is time to look on its API.

Linux kernel radix tree API
---------------------------------------------------------------------------------

We start from the data structure initialization. There are two ways to initialize a new radix tree. The first is to use `RADIX_TREE` macro:

```C
RADIX_TREE(name, gfp_mask);
````

As you can see we pass the `name` parameter, so with the `RADIX_TREE` macro we can define and initialize radix tree with the given name. Implementation of the `RADIX_TREE` is easy:

```C
#define RADIX_TREE(name, mask) \
         struct radix_tree_root name = RADIX_TREE_INIT(mask)

#define RADIX_TREE_INIT(mask)   { \
        .height = 0,              \
        .gfp_mask = (mask),       \
        .rnode = NULL,            \
}
```

At the beginning of the `RADIX_TREE` macro we define instance of the `radix_tree_root` structure with the given name and call `RADIX_TREE_INIT` macro with the given mask. The `RADIX_TREE_INIT` macro just initializes `radix_tree_root` structure with the default values and the given mask.

The second way is to define `radix_tree_root` structure by hand and pass it with mask to the `INIT_RADIX_TREE` macro:

```C
struct radix_tree_root my_radix_tree;
INIT_RADIX_TREE(my_tree, gfp_mask_for_my_radix_tree);
```

where:

```C
#define INIT_RADIX_TREE(root, mask)  \
do {                                 \
        (root)->height = 0;          \
        (root)->gfp_mask = (mask);   \
        (root)->rnode = NULL;        \
} while (0)
```

makes the same initialization with default values as it does `RADIX_TREE_INIT` macro.

The next are two functions for inserting and deleting records to/from a radix tree:

* `radix_tree_insert`;
* `radix_tree_delete`;

The first `radix_tree_insert` function takes three parameters:

* root of a radix tree;
* index key;
* data to insert;

The `radix_tree_delete` function takes the same set of parameters as the `radix_tree_insert`, but without data.

The search in a radix tree implemented in two ways:

* `radix_tree_lookup`;
* `radix_tree_gang_lookup`;
* `radix_tree_lookup_slot`.

The first `radix_tree_lookup` function takes two parameters:

* root of a radix tree;
* index key;

This function tries to find the given key in the tree and return the record associated with this key. The second `radix_tree_gang_lookup` function have the following signature

```C
unsigned int radix_tree_gang_lookup(struct radix_tree_root *root,
                                    void **results,
                                    unsigned long first_index,
                                    unsigned int max_items);
```

and returns number of records, sorted by the keys, starting from the first index. Number of the returned records will not be greater than `max_items` value.

And the last `radix_tree_lookup_slot` function will return the slot which will contain the data.

Links
---------------------------------------------------------------------------------

* [Radix tree](http://en.wikipedia.org/wiki/Radix_tree)
* [Trie](http://en.wikipedia.org/wiki/Trie)
