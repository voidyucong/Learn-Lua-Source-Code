# Table
## Table常用操作

> 插入

```
local t = {}
t.x = 1
```
> 查找

```
local t = {}
print(t.x)
```
> 获取长度

```
local t = {1, 2, 3}
print(#t)
```
> 迭代

```
local t = {1, 2, 3}
for k, v in pairs(t) do
	print(k, v)
end
```

## Table的基本结构
在 Lua 实现中，Table 实际上由两部分构成：

- Array
- Hash Table

首先看一下源码中的结构（`lobject.h`）：

```
typedef union TKey {
  struct {
    TValuefields;
    int next;  /* for chaining (offset for next node) */
  } nk; /* next key */
  TValue tvk; /* key */
} TKey;

/*
** hash table里的节点
 */
typedef struct Node {
  TValue i_val;
  TKey i_key;
} Node;


typedef struct Table {
  CommonHeader;
  lu_byte flags;  /* 1<<p means tagmethod(p) is not present */
  lu_byte lsizenode;
  unsigned int sizearray;
  TValue *array;
  Node *node;
  Node *lastfree;  /* any free position is before this position */
  struct Table *metatable;
  GCObject *gclist;
} Table;
```
捡几个主要的字段说：
> nk

Hash Table 中用到闭散列，也就是发生冲突的节点存储在表的另一个槽内，由 nk 指向下一个节点;其中的 next 是根据当前节点的偏移，可正可负;
> sizearray,lsizenode

Array 的大小和 Hash Table 的大小，由于哈希表的大小一定为 2 的整数次幂，所以这里的 lsizenode 表示的是幂次，而不是实际大小;
> array,node

Array 和 Hash Table;
> lastfree

用于指向 Hash Table 中的空闲位置;

## Hash Table 的存储方式

## 插入
获取 key 的 value 并设置；key 不存在则创建新 key。

插入操作主要是通过 `settableProtected`（lvm.c） 宏完成的操作。

```
/* same for 'luaV_settable' */
#define settableProtected(L,t,k,v) { const TValue *slot; \
  if (!luaV_fastset(L,t,k,slot,luaH_get,v)) \
    Protect(luaV_finishset(L,t,k,v,slot)); }
```
`luaV_fastset` 方法会从 table 中获取 key 是否存在，存在则这是 value 的值；否则，执行 `luaV_finishset`，如果 t 是 table 类型，则会创建一个新 key 并设置 value；如果 t 不是 table 类型，那就从 t 中获取元方法 `__newindex` 并执行对应的操作；

这种情况下，会判断给定的 key 是 string 还是 number，string后面会讲到，单说number。

插入过程：

首先，判断 key 在不在 array 的区间中，如果在，则直接对 array 操作；

否则，从 Hash Table 中获取 key 对应的 value 信息，如果 value ~= nil，说明 Hash Table 存在此 key，则直接对 Hash Table 操作；

key 既不在 array 中也不在 Hash Table 中，那么就需要执行 `luaH_newKey` 创建一个新 key。

```
/*
** inserts a new key into a hash table; first, check whether key's main
** position is free. If not, check whether colliding node is in its main
** position or not: if it is not, move colliding node to an empty place and
** put new key in its main position; otherwise (colliding node is in its main
** position), new key goes to an empty position.
*/
TValue *luaH_newkey (lua_State *L, Table *t, const TValue *key) {
  Node *mp;
  TValue aux;
  if (ttisnil(key)) luaG_runerror(L, "table index is nil");
  else if (ttisfloat(key)) {
    lua_Integer k;
    if (luaV_tointeger(key, &k, 0)) {  /* index is int? */
      setivalue(&aux, k);
      key = &aux;  /* insert it as an integer */
    }
    else if (luai_numisnan(fltvalue(key)))
      luaG_runerror(L, "table index is NaN");
  }
  mp = mainposition(t, key); // 获取key的mainposition
  if (!ttisnil(gval(mp)) || isdummy(mp)) {  /* main position is taken? */
    Node *othern;
    Node *f = getfreepos(t);  /* get a free place */
    if (f == NULL) {  /* cannot find a free place? */
      rehash(L, t, key);  /* grow table */
      /* whatever called 'newkey' takes care of TM cache */
      return luaH_set(L, t, key);  /* insert key into grown table */
    }
    lua_assert(!isdummy(f));
    othern = mainposition(t, gkey(mp));
    if (othern != mp) {  /* is colliding node out of its main position? */
      /* yes; move colliding node into free position */
      while (othern + gnext(othern) != mp)  /* find previous */
        othern += gnext(othern);
      gnext(othern) = cast_int(f - othern);  /* rechain to point to 'f' */
      *f = *mp;  /* copy colliding node into free pos. (mp->next also goes) */
      if (gnext(mp) != 0) { //
        gnext(f) += cast_int(mp - f);  /* correct 'next' */
        gnext(mp) = 0;  /* now 'mp' is free */
      }
      setnilvalue(gval(mp));
    }
    else {  /* colliding node is in its own main position */
      /* new node will go into free position */
      if (gnext(mp) != 0)
        gnext(f) = cast_int((mp + gnext(mp)) - f);  /* chain new position */
      else lua_assert(gnext(f) == 0);
      gnext(mp) = cast_int(f - mp);
      mp = f;
    }
  }
  setnodekey(L, &mp->i_key, key);
  luaC_barrierback(L, t, key);
  lua_assert(ttisnil(gval(mp)));
  return gval(mp);
}
```
`luaH_newkey`是对 Hash Table 添加 一个新 key 的操作并返回对应的 value。Table 使用闭散列表，发生冲突时会写入空闲的位置并链到冲突链表的最后面。

当 Hash Table 没有空余位置时会出发 rehash 操作。rehash 会检测 array 和 Hash Table 中的有效元素数量是否超过最大值的一半，如果 array 中的有效元素过少，则会相应调整 array 的尺寸，并把多余的元素放到 Hash Table 中，已达到较高的空间利用率。

过程如下：

![Lua Table New Key](https://raw.githubusercontent.com/voidyucong/Learn-Lua-Source-Code/master/pic/Lua%20Table%20New%20Key.png)


## 获取长度

### 思考

```
t = {1, nil, nil, 4, 5}
print(#t)

t = {1, nil, 2, 3, 4}
print(#t)

t = {1, 2, 3, nil, nil}
print(#t)

t = {1, 2, nil, 4, nil}
print(#t)

t = {1, 2, 3, 4}
t[10] = 10
print(#t)

> t = {1, nil, 2, 3, 4}
> t[10]=10
> t.x=1
> print(#t)
```

获取长度是通过 luaH_getn 方法实现的（ltable.c）：

```
/*
** Try to find a boundary in table 't'. A 'boundary' is an integer index
** such that t[i] is non-nil and t[i+1] is nil (and 0 if t[1] is nil).
*/
int luaH_getn (Table *t) {
  unsigned int j = t->sizearray;
  if (j > 0 && ttisnil(&t->array[j - 1])) {
    /* there is a boundary in the array part: (binary) search for it */
    unsigned int i = 0;
    while (j - i > 1) {
      unsigned int m = (i+j)/2;
      if (ttisnil(&t->array[m - 1])) j = m;
      else i = m;
    }
    return i;
  }
  /* else must find a boundary in hash part */
  else if (isdummy(t->node))  /* hash part is empty? */
    return j;  /* that is easy... */
  else return unbound_search(t, j);
}
```
此方法有三个分支判断：

- 1 . 如果 sizearray 大于0，并且 array 的最后一个元素是 nil，则从[0 - sizearray]二分查找值是 nil 并且前一个值不是 nil 的位置，直接返回
- 例如对 `t = {1, 2, nil, 4, nil}` 取长度，过程如下：

> - 左边界 `i = 0`，右边界 `j = 5`，`array[(i + j) / 2] != nil`，所以令 `i = (i + j) / 2`;


> - 左边界 `i = 2`，右边界 `j = 5`，`array[(i + j) / 2] == nil`，所以令 `j = (i + j) / 2`;


> - 左边界 `i = 2`，右边界 `j = 3`，i 和 j 已相邻，返回 i 的值2;

- 2 . 如果 sizearry 小于0 或者 最后一个元素不是nil，则判断 Hash Table 是否是假的数据，如果是，则直接返回 sizearray。

- 3 . 前两个条件都不满足，则执行 `unbound_search`。在 `unbound_search` ，会从 Hash Table 中从 sizearray + 1 开始查找空元素，每次步进为2次幂；找到后进行第一步二分查找边界值，并返回。
- 例如对 `t = {1, nil, 3, 4, 5}; t.x = 1;` 取长度，过程如下：

> - 暂定左边界 `i = sizearray = 4`，`j = sizearray + 1 = 5`，`node[5] ~= nil`，所以令 `j *= 2`;


> - `node[10] == nil`，所以确定右边界 `j = 10`;


> - 二分查找，最后确定 `i = 5`并返回;

下面给出上方的答案：

```
> t = {1, nil, nil, 4, 5}
> print(#t)
5

> t = {1, nil, 2, 3, 4}
> print(#t)
5

> t = {1, 2, 3, nil, nil}
> print(#t)
3

> t = {1, 2, nil, 4, nil}
> print(#t)
2

> t = {1, nil, 2, 3, 4}
> t[10]=10
> t.x=1
> print(#t)
10
```
结论：在不确定的情况下最好用 `table.count` 或者设置 table 的元方法 `__len`。


## 查找
## 迭代