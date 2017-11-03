# Lua 函数调用中的栈情况

这个拿一个特例，变参函数来查看函数调用时候栈内的分布情况。

创建一个test.lua

```
function f( n1,n2,n3,n4,n5,n6,n7,n8,n9 )
	print(1)
end
f(1,2,...) 
```



### 执行 ll_require:

这里压入栈中的 func 代表执行 `test.lua`创建的闭包，类似匿名函数，并压入了两个参数。

| func   | "test" | "./test.lua" | L->top |
| ------ | ------ | ------------ | ------ |
| 0x06b0 | 0x06c0 | 0x06d0       | 0x06e0 |

数据准备完毕，调用 `lua_call(L, 2, 1)` 执行，对 lua 函数而言，`lua_call` 调用 `luaD_precall` 和 `luaV_execute`。

### 执行 lua_precall:

如果 func 是普通函数，则`base = func + 1`；如果 func 是变参函数，则 `base = L->top`。

 L->top 需要向后移动 func 需要的寄存器数量 `L->top = L->top + p->maxstacksize`。

从现在开始，函数 func 的寄存器位置是`(base, L->top - 1)`。

| func   | "test" | "./test.lua" | base   |      |      |      | L->top |
| ------ | ------ | ------------ | ------ | ---- | ---- | ---- | ------ |
| 0x06b0 | 0x06c0 | 0x06d0       | 0x06e0 | -    | -    | -    | 0x0720 |

`luaD_precall`结束后，进入 `luaV_execute `执行字节码。本例中涉及到的字节码可通过`luac -l test.lua`获取。

### 执行 OP_CLOSURE:

指令创建 closure “f”，并存储在第一个寄存器，也就是 base 的位置。

| func   | "test" | "./test.lua" | base (f closure) |      |      |      | L->top |
| ------ | ------ | ------------ | ---------------- | ---- | ---- | ---- | ------ |
| 0x06b0 | 0x06c0 | 0x06d0       | 0x06e0           | -    | -    | -    | 0x0720 |

### 执行 OP_SETTABUP:

将 base 寄存器中 closure 放入上值 Upvalue["f"] 中:

```
不变
```



### 执行 OP_GETTABUP:

从上值中取出放入 base 位置的寄存器中:

```
不变
```



### 执行 OP_LOADK:

将常量 1 载入寄存器 0x06f0：


| func   | "test" | "./test.lua" | base (f closure) | 1      |      |      | L->top |
| ------ | ------ | ------------ | ---------------- | ------ | ---- | ---- | ------ |
| 0x06b0 | 0x06c0 | 0x06d0       | 0x06e0           | 0x06f0 | -    | -    | 0x0720 |


### 执行 OP_LOADK:

将常量 2 载入寄存器 0x0700

| func   | "test" | "./test.lua" | base (f closure) | 1      | 2      |      | L->top |
| ------ | ------ | ------------ | ---------------- | ------ | ------ | ---- | ------ |
| 0x06b0 | 0x06c0 | 0x06d0       | 0x06e0           | 0x06f0 | 0x0700 | -    | 0x0720 |



### 执行 OP_VARARG:

这个指令专门用于处理 `…` 变参的参数。

将 func 与 base 之间的参数拷贝到 剩余的寄存器中，如果不足，以 nil 填充。调整 `L->top` 以空余出寄存器。

| func   | "test" | "./test.lua" | base (f closure) | 1      | 2      | "test" | "./test.lua" | L->top |
| ------ | ------ | ------------ | ---------------- | ------ | ------ | ------ | ------------ | ------ |
| 0x06b0 | 0x06c0 | 0x06d0       | 0x06e0           | 0x06f0 | 0x0700 | 0x0710 | 0x0720       | 0x0730 |

### OP_CALL

这个指令就是真正运行函数 `f`的位置。

进入lua_precall，获取 closure ，参数数量 `L->top - base = 4`，固定需要参数位于 Proto 中的 numparams 字段， 9，需要寄存器数量 11，将不足的参数位置置 `nil`，也就是`(L->top+4, L->top + 9)`，然后进入 `luaV_execute` 执行相关的字节码。

| func   | "test" | "./test.lua" | base (f closure) | base2 (1) | 2      | "test" | "./test.lua" | nil             | 寄存器    | 寄存器    | L->top |
| ------ | ------ | ------------ | ---------------- | --------- | ------ | ------ | ------------ | --------------- | ------ | ------ | ------ |
| 0x06b0 | 0x06c0 | 0x06d0       | 0x06e0           | 0x06f0    | 0x0700 | 0x0710 | 0x0720       | 0x0730 - 0x0770 | 0x0780 | 0x0790 | 0x07a0 |



### 执行 `f`内的字节码加载寄存器数值后：

| func   | "test" | "./test.lua" | base (f closure) | base2 (1) | 2      | "test" | "./test.lua" | 5个nil           | print  | 1      | L->top |
| ------ | ------ | ------------ | ---------------- | --------- | ------ | ------ | ------------ | --------------- | ------ | ------ | ------ |
| 0x06b0 | 0x06c0 | 0x06d0       | 0x06e0           | 0x06f0    | 0x0700 | 0x0710 | 0x0720       | 0x0730 - 0x0770 | 0x0780 | 0x0790 | 0x07a0 |

### OP_CALL:

这个指令又进入到 `print`函数的执行过程中。



## 结论

`CallInfo`调用栈在数据栈内的结构是这样的：

![Lua Table New Key](https://raw.githubusercontent.com/voidyucong/Learn-Lua-Source-Code/master/pic/Lua%20CallInfo%20stack.png)

如果`ci->func`是变参函数，则 `ci->func` 与 `ci->base` 之间会间隔变参的数量。

`ci->base` 的作用其实是用来寻址寄存器的，用 `ci->base + 寄存器索引`就是寄存器的地址。
