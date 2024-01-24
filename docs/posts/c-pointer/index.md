# 你所不知道的 C 语言: 指针篇


{{&lt; admonition quote &gt;}}
*「指针」 扮演 「记忆体」 和 「物件」 之间的桥梁*
{{&lt; /admonition &gt;}}

&lt;!--more--&gt;

&gt; {{&lt; link href=&#34;https://hackmd.io/@sysprog/c-pointer&#34; content=&#34;原文地址&#34; external-icon=true &gt;}}   
&gt; {{&lt; link href=&#34;https://youtu.be/G7vERppua9o&#34; content=&#34;直播录影(上)&#34; external-icon=true &gt;}}   
&gt; {{&lt; link href=&#34;https://youtu.be/Owxols1RTAg&#34; content=&#34;直播录影(下)&#34; external-icon=true &gt;}}   

## 前言杂谈

[Let’s learn programming by inventing it](https://www.youtube.com/watch?v=l5Mp_DEn4bs) [CppCon 2018] :white_check_mark:

{{&lt; admonition open=false &gt;}}
在 K&amp;R 一书中，直到 93 页才开始谈论 pointer，而全书总计 185 页，所以大概是在全书 $50.27\\%$ 的位置才开始讲 pointer。所以即使不学 pointer，你还是能够掌握 $~50\\%$ 的 C 语言的内容，但是 C 语言的核心正是 pointer，所以 Good Luck :rofl:
{{&lt; /admonition &gt;}}

[godbolt](http://gcc.godbolt.org/) 可以直接在网页上看到，源代码由各类 compiler 生成的 Assembly Code


[How to read this prototype?]() [Stack Overflow] :white_check_mark:

{{&lt; admonition open=false &gt;}}
这个问题是关于 `signal` 系统调用的函数原型解读，里面的回答页给出了很多对于指针，特别是 *函数指针* 的说明，下面节选一些特别有意思的回答：

&gt; The whole thing declares a function called `signal`:
&gt;
&gt; - `signal` takes an int and a function pointer
&gt;   - this function pointer takes an `int` and returns `void`
&gt; - `signal` returns a function pointer
&gt;   - `this function pointer takes an `int` and returns a `void`
&gt;
&gt; That&#39;s where the last int comes in.
&gt;
&gt; ---
&gt;
&gt; You can use [the spiral rule](https://c-faq.com/decl/spiral.anderson.html) to make sense of such declarations, or the program `cdecl(1)`.

这里面提到了 [the spiral rule](https://c-faq.com/decl/spiral.anderson.html) 这是一个用于解析 C 语言中声明 (declaration) 的方法；另外还提到了 `cdecl` 这一程序，它也有类似的作用，可以使用英文进行声明或者解释。

---

&gt; Find the leftmost identifier and work your way out, remembering that `[]` and `()` bind before `*`; IOW, `*a[]` is an array of pointers, `(*a)[]` is a pointer to an array, `*f()` is a function returning a pointer, and `(*f)()` is a pointer to a function. Thus,
&gt; 
&gt; ```c
&gt; void ( *signal(int sig, void (*handler)(int)) ) (int);
&gt; ```
&gt; 
&gt; breaks down as
&gt; 
&gt; ```c
&gt;         signal                                          -- signal
&gt;         signal(                               )         -- is a function
&gt;         signal(    sig                        )         -- with a parameter named sig
&gt;         signal(int sig,                       )         --   of type int
&gt;         signal(int sig,        handler        )         -- and a parameter named handler
&gt;         signal(int sig,       *handler        )         --   which is a pointer
&gt;         signal(int sig,      (*handler)(   )) )         --   to a function
&gt;         signal(int sig,      (*handler)(int)) )         --   taking an int parameter
&gt;         signal(int sig, void (*handler)(int)) )         --   and returning void
&gt;        *signal(int sig, void (*handler)(int)) )         -- returning a pointer
&gt;      ( *signal(int sig, void (*handler)(int)) )(   )    -- to a function
&gt;      ( *signal(int sig, void (*handler)(int)) )(int)    --   taking an int parameter
&gt; void ( *signal(int sig, void (*handler)(int)) )(int);   --   and returning void
&gt; ```

这一回答强调了 `*` 和 `[]`、`()` 优先级的关系，这在判断数组指针、函数指针时是个非常好用的技巧。

{{&lt; /admonition &gt;}}

Rob Pike 于 2009/10/30 的 [Golang Talk](https://talks.golang.org/2009/go_talk-20091030.pdf) [PDF]

David Brailsford 教授解说影片 [Essentials: Pointer Power! - Computerphile](https://www.youtube.com/watch?v=t5NszbIerYc) [YouTube]

## 阅读 C 语言规格书

一手资料的重要性毋庸置疑，对于 C 语言中的核心概念 ***指针***，借助官方规格书清晰概念是非常重要的。

C99 [6.2.5] ***Types***

&gt; An array type of unknown size is an incomplete type. It is completed, for an identifier of that type, by specifying the size in a later declaration (with internal or external linkage). A structure or union type of unknown content is an incomplete type. It is completed, for all declarations of that type, by declaring the same structure or union tag with its defining content later in the same scope.

*incomplete type* 和 *linkage* 配合可以进行 forward declaration，如果搭配 pointer 则可以进一步，在无需知道 object 内部细节即可进行程序开发。

&gt; Array, function, and pointer types are collectively called derived declarator types. A declarator type derivation from a type T is the construction of a derived declarator type from T by the application of an array-type, a function-type, or a pointer-type derivation to T.


{{&lt; admonition &gt;}}
*derived declarator types*  表示衍生的声明类型，因为 array, function, pointer 本质都是地址，而它们的类型都是由其它类型衍生而来的，所以可以使用这些所谓的 *derived declarator types* 来提前声明 object，表示在某个地址会存储一个 object，这也是为什么这些类型被规格书定义为 *derived declarator types*。
{{&lt; /admonition &gt;}}

- **lvalue**: Locator value

{{&lt; admonition danger &gt;}}
C 语言里只有 ***call by value***
{{&lt; /admonition &gt;}}

## void &amp; void *

C89 之前，函数如果没有标注返回类型，则默认返回类型 `int`，返回值 0。但由于这样既可以表示返回值不重要，也可以表示返回值为 0，这会造成歧义，所以引进了 `void`。

`void *` 只能表示地址，而不能对所指向的地址区域的内容进行操作。因为通过 `void *` 无法知道所指向区域的 size，所以无法对区域的内容进行操作，必须对 `void *` 进行 ***显示转换*** 才能操作指向的内容。（除此之外，对于指针本身的操作，`void *` 与 `char *` 是等价的，即对于 `&#43;/- 1` 这类的操作，二者的偏移量是一致的）

### Alignment

这部分原文描述不是很清晰，`2-byte aligned` 图示如下：

![](/images/c/2-byte-aligned.svg)

如果是 2-byte aligned 且是 little-endian 的处理器，对于左边，可以直接使用 `*(uint16_t *) ptr`，但对于右边就无法这样（不符合 alignment）：

```c
/* may receive wrong value if ptr is not 2-byte aligned */
uint16_t value = *(uint16_t *) ptr;
/* portable way of reading a little-endian value */
uint16_t value = *(uint8_t *) ptr | ((*(uint8_t *) (ptr &#43; 1)) &lt;&lt; 8);
```

因为内存寻址的最小粒度是 Byte，所以使用 `(uint_8 *)` 不需要担心 alignment 的问题。原文并没有给出 32-bit aligned 的 portable way，我们来写一下：

```c
/* may receive wrong value if ptr is not 2-byte aligned */
uint32_t value = *(uint32_t *) ptr;
/* portable way of reading a little-endian value */
uint32_t value = *(uint8_t *) ptr 
                 | ((*(uint8_t *) (ptr &#43; 1)) &lt;&lt; 8) 
                 | ((*(uint8_t *) (ptr &#43; 2)) &lt;&lt; 16) 
                 | ((*(uint8_t *) (ptr &#43; 3)) &lt;&lt; 24);
```

{{&lt; admonition info &gt;}}
- [ ] [The Lost Art of Structure Packing](http://www.catb.org/esr/structure-packing/)
{{&lt; /admonition &gt;}}

### 规格书中的 Pointer

C99 [6.3.2.3] ***Pointers***

&gt; A pointer to a function of one type may be converted to a pointer to a function of another
type and back again; the result shall compare equal to the original pointer. Ifaconverted
pointer is used to call a function whose type is not compatible with the pointed-to type,
the behavior is undefined.

C11 [6.3.2.3] ***Pointers***

&gt;  A pointer to a function of one type may be converted to a pointer to a function of another
type and back again; the result shall compare equal to the original pointer. If a converted
pointer is used to call a function whose type is not compatible with the referenced type,
the behavior is undefined.

**C99 和 C11 都不保证 pointers (whose type is not compatible with the *pointed-to / referenced* type) 之间的转换是正确的。**

## Pointers vs. Arrays

C99 6.3.2.1
&gt; Except when it is the operand of the sizeof operator or the unary &amp; operator, or is a string literal used to initialize an array, an expression that has type ‘‘array of type’’ is converted to an expression with type ‘‘pointer to type’’ that points to the initial element of the array object and is not an lvalue.

Array 只有在表示其自身为数组时才不会被 converted to Pointer，例如

```c
// case 1: extern declaration of array
extern char a[];
// case 2: defintion of array
char a[10];
// case 3: size of array
sizeof(a);
// case 4: address of array
&amp;a
```

在其他情况则会倍 converted to Pointer，这时 Array 可以和 Pointer 互换进行表示或操作，例如

```c
// case 1: function parameter
void func(char a[]);
void func(char *a);

// case 2: operation in expression
char c = a[2];
char c = *(a &#43; 2);
```

这也是为什么对于一个 Array `a`，`&amp;a` 和 `&amp;a[0]` 值虽然相同，但 `&amp;a &#43; 1` 和 `&amp;a[0] &#43; 1` 的结果大部分时候是大不相同的，这件事乍一看是非常惊人的，但其实不然，在了解 Array 和 Pointer 之后，也就那么一回事 :rofl:

### GDB 实作

```c
char a[10];
int main() {
    return 0;
};
```

我们以上面这个例子，通过 GDB 来对 Array 和 Pointer 进行深入研究：

```bash
(gdb) print &amp;a
$1 = (char (*)[10]) 0x555555558018 &lt;a&gt;
(gdb) print &amp;a[0]
$2 = 0x555555558018 &lt;a&gt; &#34;&#34;
```

符合预期，`&amp;a` 和 `&amp;a[0]` 得到的值是相同的，虽然类型看起来不同，但是现在先放到一边。

```bash
(gdb) print &amp;a &#43; 1
$3 = (char (*)[10]) 0x555555558022
(gdb) print &amp;a[0] &#43; 1
$4 = 0x555555558019 &lt;a&#43;1&gt; &#34;&#34;
(gdb) print a &#43; 1
$5 = 0x555555558019 &lt;a&#43;1&gt; &#34;&#34;
```

Oh! 正如我们之前所说的 `&amp;a &#43; 1` 与 `&amp;a[0] &#43; 1` 结果并不相同（而 `&amp;a[0] &#43; 1` 和 `a &#43; 1` 结果相同正是我们所提到的 Array 退化为 Pointer），虽然如此，GDB 所给的信息提示我们可能是二者 Pointer 类型不相同导致的。

```bash
(gdb) whatis &amp;a
type = char (*)[10]
(gdb) whatis &amp;a[0]
type = char *
```

Great! 果然是 Pointer 类型不同导致的，我们可以看到 `&amp;a` 的类型是 `char (*)[10]` 一个指向 Array 的指针，`&amp;a[0]` 则是 `char *`。所以这两个 Pointer 在进行 `&#43;/-` 运算时的偏移量是不同的，`&amp;a[0]` 的偏移量为 `sizeof(a[0])` 即一个 `char` 的宽度 ($0x18 &#43; 1 = 0x19$)，而 `&amp;a` 的偏移量为 `sizeof(a)` 即 10 个 `char` 的宽度 ($0x18 &#43; 10 = 0x22$)。

{{&lt; admonition warning &gt;}}
在 GDB 中使用 `memcpy` 后直接打印可能会出现以下错误：

```bash
(gdb) p memcpy(calendar, b, sizeof(b[0]))
&#39;memcpy&#39; has unknown return type; cast the call to its declared return type
```

只需加入 `void *` 进行类型转换即可解决该问题：

```bash
(gdb) p (void *) memcpy(calendar, b, sizeof(b[0]))
...
```

{{&lt; /admonition &gt;}}

### malloc

{{&lt; admonition tip &gt;}}
遇到陌生的函数，可以使用 `man` 来快速查阅手册，例如 `man strcpy`, `man strcat`，手册可以让我们快速查询函数的一些信息，从而进入实作。
{{&lt; /admonition &gt;}}

### Runtime Environment

根据 [Zero size arrays in C ](https://news.ycombinator.com/item?id=11674374)，原文中的 `char (*argv)[0]` 在函数参数传递时会被转换成 `char **argv`。而为什么在查看地址 `((char **) argv)[0]` 开始的连续 4 个 `char *` 内容时，会打印出 `envp` 中的内容，可以参考以下的进入 `main` 函数时的栈布局：

![](/images/c/argv.png)

`argv` 和 `envp` 所指的字符串区域是相连的，所以在越过 `argv` 字符串区域的边界后，会继续打印 `envp` 区域的字符串。这也是为什么打印出的字符串之间地址增长于其长度相匹配。所以从地址 `(char **) argv` 开始的区域只是一个 `char *` 数组，使用 `x/4s` 对这部分进行字符串格式打印显然是看不懂的。

{{&lt; admonition &gt;}}
`argv` 和 `envp` 都是在 shell 进行 `exec` 系统调用之前进行传递（事实上是以 arguments 的形式传递给 `exec`）

man 2 execve
```c
int execve(const char *pathname, char *const argv[],
           char *const envp[]);
```

`execve` 实际上在内部调用了 `fork`，所以 `argv` 和 `envp` 的传递是在 `fork` 之前。（设想如果是在 `fork` 之后传递，可能会出现 `fork` 后 child process 先执行，这种情况 child process 显然无法获得这些被传递的信息）

注意到 `execve` 只传递了 `argv` 而没有传递 `argc`，这也很容易理解，`argc` 是 `argv` 的计数，只需 `argv` 即可推导出 `argc`。

{{&lt; /admonition &gt;}}

## Function Pointer

{{&lt; admonition danger&gt;}}
与 Array 类似，Function 只有在表示自身时不会被 converted to Function Pointer (即除 `sizeof` 和 `&amp;` 运算之外)，其它情况、运算时都会被 convert to Function Pointer

理解 C 语言中的 Function 以及 Function Pointer 的核心在于理解 ***Function Designator*** 这个概念，函数名字必然是 Function Designator，其它的 designator 则是根据以下两条规则进行推导得来。
{{&lt; /admonition &gt;}}

C99 [ 6.3.2.1 ] 
&gt; A function designator is an expression that has function type. **Except** when it is the operand of the **sizeof** operator or the unary **&amp;** operator, a **function designator** with type ‘‘function returning type’’ **is converted to** an expression that has type ‘‘**pointer to function returning type**’’.

C99 [6.5.3.2-4] 
&gt; The unary * operator denotes indirection. **If the operand points to a function, the result is a function designator.**

## 指针的修饰符

指针 `p` 自身不能变更，既不能改变 `p` 自身所存储的地址。const 在 * 之后：

```c
char * const p;
```

指针 `p` 所指向的内容不能变更，即不能通过 `p` 来更改所指向的内容。const 在 * 之前：

```c
const char * p;
char const * p;
```
指针 `p` 自身于所指向的内容都不能变更：

```c
const char * const p;
char const * const p;
```

## 字符串



---

> 作者: [ccrysisa](https://github.com/ccrysisa)  
> URL: https://ccrysisa.github.io/posts/c-pointer/  
