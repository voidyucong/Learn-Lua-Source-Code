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
> flags

位图，每一位代表的是一个元方法类型，可快速判断当前 table 是否有某个元方法。


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

**向 Hash Table 中插入及冲突处理过程如下：**

![Lua Table New Key](https://raw.githubusercontent.com/voidyucong/Learn-Lua-Source-Code/master/pic/Lua%20Table%20New%20Key.png)

当 Hash Table 没有空余位置时会出发 rehash 操作。rehash 的主要工作是统计当前 table 中到底有多少有效键值对，以及决定数组部分需要开辟多少空间。 其原则是最终数组部分的利用率需要超过 50%。如果 array 中的有效元素过少或者但是需要的空间较大（例如在下标 1、2、100 处存在三个元素，则没必要开辟 100 的空间存放），则会相应调整 array 的尺寸，并把多余的元素放到 Hash Table 中，已达到较高的空间利用率。

例如：

```
local t = {1, 2, nil, nil, 5} -- 1
t.x = 1  -- 2
```
在table t 创建时，会把所有元素放置到 array 中，并设置 sizearray = 5，lsizenode = 0；

接下来对 t 创建了一个新的key x，所以会对 Hash Table 进行 rehash 以容纳所有的 key-value 类型；

rehash 过程中也会对 array 进行 resize，根据现有的大小会调整为8，之后发现有效元素只有3个（1、2、5），没超过 50% 使用率，所以会设置 sizearray = 4 并且将超出的元素 5 放到 Hash Table中；同理，收缩后发现有效元素（1、2）还是没超过 50% 利用率，设置 sizearray = 2；

最终，sizearray = 2，array 中存放1、2，lsizenode = 1，Hash Table 中存放 5、x；


## 获取长度

### 思考

```
t = {1, nil, nil, 4, 5}
print(#t)

t = {1, 2, 3, nil, nil}
print(#t)

t = {1, 2, nil, 4, nil}
print(#t)

t = {1, nil, 2, 3, 4}
t[100]=100
t.x=1
print(#t)
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

- **1** . 如果 sizearray 大于0，并且 array 的最后一个元素是 nil，则从[0 - sizearray]**二分查找 值不是 nil 但是后一个值是 nil 的位置**，直接返回。
  例如对 `t = {1, 2, nil, 4, nil}` 取长度，过程如下：

> - 左边界 `i = 0`，右边界 `j = 5`，`array[(i + j) / 2] != nil`，所以令 `i = (i + j) / 2`;


> - 左边界 `i = 2`，右边界 `j = 5`，`array[(i + j) / 2] == nil`，所以令 `j = (i + j) / 2`;


> - 左边界 `i = 2`，右边界 `j = 3`，i 和 j 已相邻，返回 i 的值2;

- **2** . 如果 sizearry 小于0 或者 最后一个元素不是nil，则判断 Hash Table 是否是假的数据，如果是，则直接返回 sizearray。

- **3** . 前两个条件都不满足，则执行 `unbound_search`。在 `unbound_search` ，会从 Hash Table 中从 sizearray + 1 开始查找空元素，每次步进为2次幂；找到后进行第一步二分查找边界值，并返回。
- 例如对 `t = {1, nil, 3, 4, 5}; t.x = 1;` 取长度，过程如下：

> - 暂定左边界 `i = sizearray = 4`，`j = sizearray + 1 = 5`，`node[j] ~= nil`，所以令 `j *= 2`;


> - `node[10] == nil`，所以确定右边界 `j = 10`;


> - 二分查找，最后确定 `i = 5`并返回;

下面给出上方的答案：

```
> t = {1, nil, nil, 4, 5}
> print(#t)
5

> t = {1, 2, 3, nil, nil}
> print(#t)
3

> t = {1, 2, nil, 4, nil}
> print(#t)
2

> t = {1, nil, 2, 3, 4}
> t[100]=100
> t.x=1
> print(#t)
5
```
结论：在不确定的情况下最好用 `table.count` 或者自定义 `__len` 方法。

## 迭代
### pairs
Lua 提供了一个 next 方法，传入上一个键，返回下一个键值对，而 pairs 也是用的这个方法，也就是luaH_next（ltable.c）。

```
int luaH_next (lua_State *L, Table *t, StkId key) {
  unsigned int i = findindex(L, t, key);  /* find original element */
  for (; i < t->sizearray; i++) {  /* try first array part */
    if (!ttisnil(&t->array[i])) {  /* a non-nil value? */
      setivalue(key, i + 1);
      setobj2s(L, key+1, &t->array[i]);
      return 1;
    }
  }
  for (i -= t->sizearray; cast_int(i) < sizenode(t); i++) {  /* hash part */
    if (!ttisnil(gval(gnode(t, i)))) {  /* a non-nil value? */
      setobj2s(L, key, gkey(gnode(t, i)));
      setobj2s(L, key+1, gval(gnode(t, i)));
      return 1;
    }
  }
  return 0;  /* no more elements */
}
```
它尝试在 array 中获取下一个值，当超出 array 部分后，则检索 Hash Table 中的位置。

而对键的预先操作是在 findinex 中完成的，如果给定的键是nil，返回0，否则，会在 array 和 Hash Table 中查找并返回相应的值。

```
/*
** returns the index of a 'key' for table traversals. First goes all
** elements in the array part, then elements in the hash part. The
** beginning of a traversal is signaled by 0.
*/
static unsigned int findindex (lua_State *L, Table *t, StkId key) {
  unsigned int i;
  if (ttisnil(key)) return 0;  /* first iteration */
  i = arrayindex(key);
  if (i != 0 && i <= t->sizearray)  /* is 'key' inside array part? */
    return i;  /* yes; that's the index */
  else {
    int nx;
    Node *n = mainposition(t, key);
    for (;;) {  /* check whether 'key' is somewhere in the chain */
      /* key may be dead already, but it is ok to use it in 'next' */
      if (luaV_rawequalobj(gkey(n), key) ||
            (ttisdeadkey(gkey(n)) && iscollectable(key) &&
             deadvalue(gkey(n)) == gcvalue(key))) {
        i = cast_int(n - gnode(t, 0));  /* key index in hash table */
        /* hash elements are numbered after array ones */
        return (i + 1) + t->sizearray;
      }
      nx = gnext(n);
      if (nx == 0)
        luaG_runerror(L, "invalid key to 'next'");  /* key not found */
      else n += nx;
    }
  }
}
```
### ipairs
ipairs 是通过 lua_geti 方法根据给定的 int 类型键查找，当遇到 nil 后停止，控制循环的代码在 lvm.c 中，后面的章节会讲到。

```
LUA_API int lua_geti (lua_State *L, int idx, lua_Integer n) {
  StkId t;
  const TValue *slot;
  lua_lock(L);
  t = index2addr(L, idx);
  if (luaV_fastget(L, t, n, slot, luaH_getint)) {
    setobj2s(L, L->top, slot);
    api_incr_top(L);
  }
  else {
    setivalue(L->top, n);
    api_incr_top(L);
    luaV_finishget(L, t, L->top - 1, L->top - 1, slot);
  }
  lua_unlock(L);
  return ttnov(L->top - 1);
}
```

## 元表
由于很多操作都需要对元表的元方法进行访问，并非所有元表都提供了需要的元方法，所以查询一个元方法存不存在就需要特别的优化。

每个 Table 结构中都有一个 flags 字段，它用位的方式记录当前有哪些元方法。

下面是从 table 中获取指定 key 的值。

```
void luaV_finishget (lua_State *L, const TValue *t, TValue *key, StkId val,
                      const TValue *slot) {
  int loop;  /* counter to avoid infinite loops */
  const TValue *tm;  /* metamethod */
  for (loop = 0; loop < MAXTAGLOOP; loop++) { /* 处理MAXTAGLOOP(2000)层的元表 */
    if (slot == NULL) {  /* 't' is not a table? */
      lua_assert(!ttistable(t));
      tm = luaT_gettmbyobj(L, t, TM_INDEX);
      if (ttisnil(tm))
        luaG_typeerror(L, t, "index");  /* no metamethod */
      /* else will try the metamethod */
    }
    else {  /* 't' is a table */
      lua_assert(ttisnil(slot));
      tm = fasttm(L, hvalue(t)->metatable, TM_INDEX);  /* table's metamethod */
      if (tm == NULL) {  /* no metamethod? */
        setnilvalue(val);  /* result is nil */
        return;
      }
      /* else will try the metamethod */
    }
    if (ttisfunction(tm)) {  /* is metamethod a function? */
      luaT_callTM(L, tm, t, key, val, 1);  /* call it */
      return;
    }
    t = tm;  /* else try to access 'tm[key]' */
    if (luaV_fastget(L,t,key,slot,luaH_get)) {  /* fast track? */
      setobj2s(L, val, slot);  /* done */
      return;
    }
    /* else repeat (tail call 'luaV_finishget') */
  }
  luaG_runerror(L, "'__index' chain too long; possible loop");
}
```
luaV_finishget 一般是在 table 中直接获取 key 不存在时的后续步骤。如果 table 中没有，则从 metatable 中查找__index元方法，如果__index存在并且是 function，则调用 luaT_callTM 执行 __index 方法。否则将 metatable 按照同样的方法执行。注意，层次深度定义为宏 **MAXTAGLOOP(1000)**。

```
void luaT_callTM (lua_State *L, const TValue *f, const TValue *p1,
                  const TValue *p2, TValue *p3, int hasres) {
  ptrdiff_t result = savestack(L, p3);
  StkId func = L->top;
  setobj2s(L, func, f);  /* push function (assume EXTRA_STACK) */
  setobj2s(L, func + 1, p1);  /* 1st argument */
  setobj2s(L, func + 2, p2);  /* 2nd argument */
  L->top += 3;
  if (!hasres)  /* no result? 'p3' is third argument */
    setobj2s(L, L->top++, p3);  /* 3rd argument */
  /* metamethod may yield only when called from Lua code */
  if (isLua(L->ci))
    luaD_call(L, func, hasres);
  else
    luaD_callnoyield(L, func, hasres);
  if (hasres) {  /* if has result, move it to its place */
    p3 = restorestack(L, result);
    setobjs2s(L, p3, --L->top);
  }
}
```
所有元方法的参数都是 table、key，并把结果返回。