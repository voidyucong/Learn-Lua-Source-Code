# String
Lua 中字符串的保存形式有两种，短字符串和长字符串。

```
#define LUA_TSHRSTR	(LUA_TSTRING | (0 << 4))  /* short strings */
#define LUA_TLNGSTR	(LUA_TSTRING | (1 << 4))  /* long strings */
```
区分长短字符串的接线被定义在 luaconf.h 的 LUAI_MAXSHORTLEN 内

```
#define LUAI_MAXSHORTLEN	40
```
TString 结构定义在`lobject.h`中

```
/*
** Header for string value; string bytes follow the end of this structure
** (aligned according to 'UTString'; see next).
*/
typedef struct TString {
  CommonHeader;
  lu_byte extra;
  lu_byte shrlen;  /* length for short strings */
  unsigned int hash;
  union {
    size_t lnglen;  /* length for long strings */
    struct TString *hnext;  /* linked list for hash table */
  } u;
} TString;
```
- extra: 对于短字符串用来判断是否是系统保留串（例如function，return...），以加快分析器; 对于长字符串用于惰性hash
- shrlen: 短字符串的长度
- hash: 长字符串的 hash 值，用于存放在 table 中的位置
- lnglen: 长字符串的长度
- hnext: 开散列的冲突键；

所有短字符串都会只保留一份在全局表 strt 中。

字符串的数据内容并没有被分配独立一块内存来保存，而是直接最加在 TString 结构的后面。用 getstr 这个宏就可以取到实际的 C 字符串指针。

```lua
#define getstr(ts)  \
  check_exp(sizeof((ts)->extra), cast(char *, (ts)) + sizeof(UTString))
```

## 比较
长短两种类型是需要不同的比较方法。

```
/*
** equality for long strings
*/
int luaS_eqlngstr (TString *a, TString *b) {
  size_t len = a->u.lnglen;
  lua_assert(a->tt == LUA_TLNGSTR && b->tt == LUA_TLNGSTR);
  return (a == b) ||  /* same instance or... */
    ((len == b->u.lnglen) &&  /* equal length and ... */
     (memcmp(getstr(a), getstr(b), len) == 0));  /* equal contents */
}
```
长字符串比较，先比较长度，然后逐个字符比较就可以。

```
/*
** equality for short strings, which are always internalized
*/
#define eqshrstr(a,b)	check_exp((a)->tt == LUA_TSHRSTR, (a) == (b))
```
短字符串比较就更简单的写成了宏，因为短字符串只会有一份，所以比较地址就可以。

## 短字符串的实现
上面说过，所有的短字符串保存在全局表 strt 中, strt 是一个 stringtable 类型。

```
typedef struct stringtable {
  TString **hash;
  int nuse;  /* number of elements */
  int size;
} stringtable;
```
- hash: 开散列形式的哈希表
- nuse: 表中元素数量
- size: 表的总大小

```
static TString *internshrstr (lua_State *L, const char *str, size_t l) {
  TString *ts;
  global_State *g = G(L);
  unsigned int h = luaS_hash(str, l, g->seed); // 将 str => hash
  TString **list = &g->strt.hash[lmod(h, g->strt.size)]; // 将strhash映射到hash表中，取出链表
  lua_assert(str != NULL);
  for (ts = *list; ts != NULL; ts = ts->u.hnext) {
    if (l == ts->shrlen &&
        (memcmp(str, getstr(ts), l * sizeof(char)) == 0)) {
      /* found! */
      if (isdead(g, ts))  /* dead (but not collected yet)? */
        changewhite(ts);  /* resurrect it */
      return ts;
    }
  }
  // 当stringtable的元素数量大于table的最大数量，则resize
  if (g->strt.nuse >= g->strt.size && g->strt.size <= MAX_INT/2) {
    luaS_resize(L, g->strt.size * 2);
    list = &g->strt.hash[lmod(h, g->strt.size)];  /* recompute with new size */
  }
  ts = createstrobj(L, l, LUA_TSHRSTR, h);
  memcpy(getstr(ts), str, l * sizeof(char)); // 将str保存在结构体的后面
  ts->shrlen = cast_byte(l);
  ts->u.hnext = *list;
  *list = ts;
  g->strt.nuse++;
  return ts;
}
```
字符串放入哈希表时，查看表中是否有相同的字符串，有则把存在的字符串直接返回；没有则创建一个 TString。其中还涉及到了 gc 对它的处理和全局表的扩容。

# UserData
UserData 在存储形式上类似字符串，大概就是个有元表的字符串。

```
/*
** Header for userdata; memory area follows the end of this structure
** (aligned according to 'UUdata'; see next).
*/
typedef struct Udata {
  CommonHeader;
  lu_byte ttuv_;  /* user value's tag */
  struct Table *metatable;
  size_t len;  /* number of bytes */
  union Value user_;  /* user value */
} Udata;
```
- ttuv_: 类型
- metatable: 元表
- len: 长度，包括结构和指针的大小



