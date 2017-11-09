# Lua 栈

## 数据栈

Lua 与 其他语言进行通讯的手段就是数据栈，宿主语言将数据压入栈中，Lua 从栈中就能获取到信息。

数据栈的初始化在函数 `stack_init` 中，分配栈内存空间，设置栈底、栈头等数据。

```c
static void stack_init (lua_State *L1, lua_State *L) {
  int i; CallInfo *ci;
  /* initialize stack array */
  L1->stack = luaM_newvector(L, BASIC_STACK_SIZE, TValue);
  L1->stacksize = BASIC_STACK_SIZE;
  for (i = 0; i < BASIC_STACK_SIZE; i++)
    setnilvalue(L1->stack + i);  /* erase new stack */
  L1->top = L1->stack;
  L1->stack_last = L1->stack + L1->stacksize - EXTRA_STACK; // space available
  /* initialize first ci */
  ci = &L1->base_ci;
  ci->next = ci->previous = NULL;
  ci->callstatus = 0;
  ci->func = L1->top;
  setnilvalue(L1->top++);  /* 'function' entry for this 'ci' */
  ci->top = L1->top + LUA_MINSTACK;
  L1->ci = ci;
}
```
数据栈的初始默认大小是 `BASIC_STACK_SIZE`。

```c
/* minimum Lua stack available to a C function */
#define LUA_MINSTACK	20

#define BASIC_STACK_SIZE        (2*LUA_MINSTACK)
```
当栈空间不足时，调用 `luaD_reallocstack` 扩充大小。

```c
void luaD_reallocstack (lua_State *L, int newsize) {
  TValue *oldstack = L->stack;
  int lim = L->stacksize;
  lua_assert(newsize <= LUAI_MAXSTACK || newsize == ERRORSTACKSIZE);
  lua_assert(L->stack_last - L->stack == L->stacksize - EXTRA_STACK);
  luaM_reallocvector(L, L->stack, L->stacksize, newsize, TValue);
  for (; lim < newsize; lim++)
    setnilvalue(L->stack + lim); /* erase new segment */
  L->stacksize = newsize;
  L->stack_last = L->stack + newsize - EXTRA_STACK;
  correctstack(L, oldstack);
}
```
Lua 中有很多对数据栈的引用，当大小变化后，需要修正这些指针，例如 `L->top`、调用帧。

```c
static void correctstack (lua_State *L, TValue *oldstack) {
  CallInfo *ci;
  UpVal *up;
  L->top = (L->top - oldstack) + L->stack;
  for (up = L->openupval; up != NULL; up = up->u.open.next)
    up->v = (up->v - oldstack) + L->stack;
  for (ci = L->ci; ci != NULL; ci = ci->previous) {
    ci->top = (ci->top - oldstack) + L->stack;
    ci->func = (ci->func - oldstack) + L->stack;
    if (isLua(ci))
      ci->u.l.base = (ci->u.l.base - oldstack) + L->stack;
  }
}
```

## 调用栈

调用栈是一个双向链表，存放着 `CallInfo` 结构，每一个结构中包含指向前一个节点和后一个节点的指针。

### CallInfo 字段说明

```c
typedef struct CallInfo {
  StkId func;  /* function index in the stack */
  StkId	top;  /* top for this function */
  struct CallInfo *previous, *next;  /* dynamic call link */
  union {
    struct {  /* only for Lua functions */
      StkId base;  /* base for this function */
      const Instruction *savedpc;
    } l;
    struct {  /* only for C functions */
      lua_KFunction k;  /* continuation in case of yields */
      ptrdiff_t old_errfunc;
      lua_KContext ctx;  /* context info. in case of yields */
    } c;
  } u;
  ptrdiff_t extra;
  short nresults;  /* expected number of results from this function */
  lu_byte callstatus;
} CallInfo;
```
- StkId func: 指向数据栈中函数本体的指针；

- StkId	top: 这次调用所用到的数据栈最大位置；

- struct CallInfo *previous, *next: 构成双向链表的两个指针；

- StkId base: 表示当前函数的寄存器开始位置，只适用于 Lua 函数；

- const Instruction *savedpc: 字节码列表，只适用于 Lua 函数；

- lua_KFunction k: 中断恢复函数指针，只适用于 c 函数；

- ptrdiff_t old_errfunc:

- lua_KContext ctx:

- ptrdiff_t extra: 

- short nresults: 期望的函数返回值数量；

- lu_byte callstatus: 当前调用栈状态，所有的状态定义在文件 `lstate.h` 中：

	```c
	/*
	** Bits in CallInfo status
	*/
	#define CIST_OAH	(1<<0)	/* original value of 'allowhook' */
	#define CIST_LUA	(1<<1)	/* call is running a Lua function */
	#define CIST_HOOKED	(1<<2)	/* call is running a debug hook */
	#define CIST_FRESH	(1<<3)	/* call is running on a fresh invocation
	                                   of luaV_execute */
	#define CIST_YPCALL	(1<<4)	/* call is a yieldable protected call */
	#define CIST_TAIL	(1<<5)	/* call was tail called */
	#define CIST_HOOKYIELD	(1<<6)	/* last hook called yielded */
	#define CIST_LEQ	(1<<7)  /* using __lt for __le */
	```
	可以看到有一个标志位来表示函数是不是一个 lua 函数，所以 Lua 提供了宏 `#define isLua(ci) ((ci)->callstatus & CIST_LUA)` 来判断。

### 调用栈的出生

当进入一个新层次的函数调用时，通过 `luaE_extendCI` 返回一个调用栈的栈帧 `CallInfo`，值得注意的是，并不是每次都创建一个新的栈帧，而是会判断当前栈帧后面有没有可以复用的，没有再创建新栈帧。

宏 `#define next_ci(L) (L->ci = (L->ci->next ? L->ci->next : luaE_extendCI(L)))` 做了这些事情。

```c
CallInfo *luaE_extendCI (lua_State *L) {
  CallInfo *ci = luaM_new(L, CallInfo);
  lua_assert(L->ci->next == NULL);
  L->ci->next = ci;
  ci->previous = L->ci;
  ci->next = NULL;
  L->nci++;
  return ci;
}
```
*那栈帧就永远存在吗？*

当 GC 或者 函数抛出异常时，会对数据栈和调用栈进行一系列的收缩操作。

- 用函数 `stackinuse` 获得当前数据栈的实际使用大小，并根据算法计算出最适合当前的数据栈空间；

- 判断当前栈实际空间如果大于最大空间 `LUAI_MAXSTACK`，则释放当前栈帧后的所有栈帧，否则，从当前栈帧开始释放一半的栈帧；

- 根据第一步计算出来的最佳空间重新设置数据栈的大小；

`luaD_shrinkstack` 的实现：

```c
void luaD_shrinkstack (lua_State *L) {
  int inuse = stackinuse(L);
  int goodsize = inuse + (inuse / 8) + 2*EXTRA_STACK;
  if (goodsize > LUAI_MAXSTACK) goodsize = LUAI_MAXSTACK;
  if (L->stacksize > LUAI_MAXSTACK)  /* was handling stack overflow? */
    luaE_freeCI(L);  /* free all CIs (list grew because of an error) */
  else
    luaE_shrinkCI(L);  /* shrink list */
  if (inuse <= LUAI_MAXSTACK &&  /* not handling stack overflow? */
      goodsize < L->stacksize)  /* trying to shrink? */
    luaD_reallocstack(L, goodsize);  /* shrink it */
  else
    condmovestack(L,,);  /* don't change stack (change only for debugging) */
}
```

## 数据栈与调用栈的数据图

![Lua Table New Key](https://raw.githubusercontent.com/voidyucong/Learn-Lua-Source-Code/master/pic/Lua%20CallInfo%20stack2.png)





