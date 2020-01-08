리눅스 커널의 여러 데이터 구조
================================================================================

기수(Radix) 트리
--------------------------------------------------------------------------------

이미 알고 있듯이 리눅스 커널은 다양한 데이터 구조와 알고리즘을 구현하는 다양한 라이브러리와 함수를 제공합니다.  리눅스 커널에서 제공하는 자료구조 중 하나 인 [기수(Radix) 트리](https://en.wikipedia.org/wiki/Radix_tree) 에 대해 다뤄 볼 것입니다. 

* [include/linux/radix-tree.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/radix-tree.h)
* [lib/radix-tree.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/lib/radix-tree.c)

위에 두 파일은 리눅스 커널에서 `기수(Radix) 트리`의 구현과 API에 관련된 파일입니다.

`기수(Radix) 트리`에 대해서 이야기해봅시다. 기수 트리는 `압축 트라이(compressed trie)`입니다. [트라이(trie)](https://en.wikipedia.org/wiki/Trie)는 연관 배열의 인터페이스를 구현하고 값을 `키-값`으로 저장할 수있는 데이터 구조입니다. 키는 일반적으로 문자열이지만 모든 데이터 타입을 키로 사용할 수 있습니다. 노드(node)때문에 `트라이(trie)`는 `n-트리` 다릅니다. `트라이(trie)`의 노드는 키를 저장하지 않습니다. 대신 `트라이(trie)`의 노드는 단일 문자 레이블을 저장합니다. 주어진 노드와 관련된 키는 트리의 루트에서이 노드로 순회하여 파생됩니다. 예를 들면 다음과 같습니다



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
위의 `트라이(tire)`에서 `go`와 `cat` `키(key)`인 것을 알 수 있습니다. `압축 트라이(compressed trie)` 또는 `기수(Radix) 트리`는 자식이 하나만있는 모든 중간 노드가 제거된다는 점에서 `trie`와 다릅니다.

리눅스 커널에서 `기수(Radix) 트리`는 `값(value)`을 `정수 키(integer key)`에 매핑하는 자료구조이다. 아래 파일에서 다음과 같은 구조로 구현됩니다.
[include/linux/radix-tree.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/radix-tree.h):
```C
struct radix_tree_root {
         unsigned int            height;
         gfp_t                   gfp_mask;
         struct radix_tree_node  __rcu *rnode;
};
```
`radix_tree_root` 구조체는 아래 세가지 필드를 포함하고 있습니다.
* `height`   - 트리의 높이;
* `gfp_mask` - 메모리의 할당과 수행 방법에 대해 알려줍니다; (gfp_mask는 페이지 할당자에게 할당 할 수있는 페이지, 할당자가 더 많은 메모리가 해제 될 때까지 대기 할 수 있는지 등을 알리는 데 사용됩니다.)
* `rnode`    - 자식노드에 대한 포인터.


첫 번째 이야기할 것은 `gfp_mask`이다.

낮은 수준의 커널 메모리 할당 기능은 플래그를  `gfp_mask`으로 지정합니다. `gfp_mask`는  할당을 수행하는 방법을 설명합니다. 할당 프로세스를 제어하는 `GFP_` 플래그는 다음과 같은 값을 가질 수 있습니다 :  `GF_NOIO`, `__GFP_HIGHMEM`, `GFP_ATOMIC`.

| flag | mean|
|-----|------|
|`GF_NOIO`| 메모리에 대한 절전 및 대기|
|`__GFP_HIGHMEM`|높은 메모리를 사용 할 수 있음|
|`GFP_ATOMIC` |할당 프로세스 우선순위가 높아서 절전 할 수 없다.|


다음은  필드는 `rnode` 이다


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
이 구조는 부모의 오프셋과 하단의 높이, 하위 노드의 수와 노드에 접근하고 노드를 해제하기 위한 필드를 포함하고 있다. 이 필드는 아래에 설명되어 있다.

* `path` - 부모에서 오프셋 및 아래에서 높이
* `count` - 하위 노드의 수
* `parent` - 상위 노드에 대한 포인터
* `private_data` - 트리 사용자가 사용함
* `rcu_head` - 노드를 해제하는 데 사용됨
* `private_list` - 트리 사용자가 사용함


라딕스 트리니드(radix_tree_node)-태그(tags)-슬롯(slots)의 마지막 두 분야가 중요하고 흥미롭다. 모든 노드는 데이터에 대한 포인터를 저장하는 슬롯 세트를 포함할 수 있다. 리눅스 커널 라딕스 트리 구현의 빈 슬롯은 NULL을 저장한다. 리눅스 커널의 라딕스 트리도 라딕스_트리_노드 구조의 태그와 연관된 태그를 지원한다. 태그는 라딕스 트리에 저장된 레코드에 개별 비트를 설정할 수 있다.


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
