
# 1\.概览：


谈到raft协议实现就绕不开网上流行的mit6\.824，但其为go语言，官方没有lab的答案，框架也很晦涩难懂，且全网没有一个博客对其有清晰的解释，有的只是甩一堆名词然后直接贴没有任何注释的代码，非常不适合学习。
但是github上面的[cornerstone](https://github.com)是纯c\+\+实现的一个非常优雅且精简的raft协议，码风优美，代码易懂，接口清晰，对c\+\+党非常友好，也很适合初学raft的人来学习。
鉴于cornerstone这么优秀的代码还没人对其有过源码级解析，我决定记录自己学习其源码过程并对其源码进行详细解析。
那我们得从哪里开始分析cornerstone呢？开始的切入点应该越小越好，同时具有很强的通用性，在很多环节又用得到。
而[buffer](https://github.com)是cornerstone中的一个非常重要的概念，从rpc发送请求，接受response，到log记录等等方面都采用buffer来存储信息。
因此让我们先从buffer开始我们对cornerstone源码的解析。


# 2\.buffer的总体架构


![](https://img2024.cnblogs.com/blog/3534914/202410/3534914-20241012230026611-1645016322.png)
**如图，总体buffer可分为3部分：**
`size`：记录`data`块的字节个数，从0开始编号（size \= 0代表data块为空而不是buffer为空），不包括前面的`size`与`pos`（`size` \+ `pos` 统称**header**）。
根据size的大小（也就是data块的字节数）可将buffer分为大块与小块，其中size \>\= 0x8000 为大块，否则为小块。**（这里有个问题：大小块size都是一个范围，我们要怎么快速来确定buffer是大块还是小块呢？这个问题答案我们放到后面再细说）**
`pos`：记录`data`块的读写指针，也是从0开始编号（这里`pos`既记录读又记录写操作的位置，本身是1个指针，**存不了两条信息**，所以我们需要自己手动调整`pos`）
`data`：存储buffer里面的实际数据，可以是`int，ulong，str`等多种类型


# 3\.buffer的内存分配


我们先上源码：



```
bufptr buffer::alloc(const size_t size) {
    if (size >= 0x80000000) {
        throw std::out_of_range("size exceed the max size that cornrestone::buffer could support");
    }

    if (size >= 0x8000) {
        bufptr buf(reinterpret_cast(new char[size + sizeof(uint) * 2]), &free_buffer);
        any_ptr ptr = reinterpret_cast(buf.get());
        __init_b_block(ptr, size);
        return buf;
    }

    bufptr buf(reinterpret_cast(new char[size + sizeof(ushort) * 2]), &free_buffer);
    any_ptr ptr = reinterpret_cast(buf.get());
    __init_s_block(ptr, size);
    return buf;
}

```

我们这里只分析大块的分配，小块的代码同理。
**(1\)**首先判断要求分配size的大小，如果`size >= 0x80000000`，直接抛出异常


**(2\)**当`size >= 0x8000`（也就是INT\_MAX \= 32768）的时候意味着要分配大块。通过`new char[size + sizeof(uint) * 2]`分配了要求的size \+ header的字节数。这里bufptr的定义在`buffer.hxx`里面`using bufptr = uptr;`。根据bufptr的定义可以知道这是一个指向buffer类的unique\_ptr,第二个参数`void(*)(buffer*)`是一个函数指针， 返回值为void,参数是buffer\*,对应着源码里面的`&free_buffer`,是一个自定义的释放bufptr指向内容的函数。


**(3\)**把bufptr展开来就是`unique_ptr buf(reinterpret_cast(new char[size + sizeof(uint) * 2]), &free_buffer)`, 这里reinterpret\_cast是用于无关类型的相互转换。new char\[]返回的是char \*指针，但是根据`unique_ptr A(xxx)`的语法括号里面的xxx是指向T类型的指针，所以我们需要用reinterpret\_cast将char \*指针转换为buffer \*


**(4\)**完成了内存的分配然后到`any_ptr ptr = reinterpret_cast(buf.get());`这里的any\_ptr在`basic_types.hxx`里面的定义是`typedef void* any_ptr;`。而`buf.get()`是unique\_ptr的一个成员函数，用于获取其原始指针，那么`any_ptr ptr = reinterpret_cast(buf.get());`这一行实现的便是将原始指针提取出来并转换为`void*`类型




---


**(5\)**接着是`__init_b_block(ptr, size);`这个宏定义



```
#define __init_block(p, s, t) ((t*)(p))[0] = (t)s;\
    ((t*)(p))[1] = 0
#define __init_s_block(p, s) __init_block(p, s, ushort)
#define __init_b_block(p, s) __init_block(p, s, uint);\
    *((uint*)(p)) |= 0x80000000

```

b\_block表示大块，s\_block表示小块
**(5\.1\)**不管大块还是小块都通过`__init_block(p, s, t)`来初始化，t表示类型(`ushort`或`uint`),p就是指向buffer的指针,s是buffer的`size`参数。


**(5\.2\)**前面**buffer的总体架构**里面我们说过buffer分为三个部分，那么这里的p\[0],p\[1]很明显就是对应的`size` 与 `pos` 参数。


**(5\.3\)**为什么初始化`size` 与 `pos`参数不直接用`p[0] = size, p[1] = pos`呢？这里的`((t*)(p))[0]，((t*)(p))[1]`又是什么？
由于我们规定`size >= 0x8000`(**USHORT\_MAX \= 32768**), 说明我们p\[0]存的size在大块的时候就不能用ushort来表示了，必须得用uint类型，所以我们将p指针强转为uint\*类型，这样uint\*意义下的p\[0]便表示以p开始往后数uint个字节来存储我们的size。pos也是同理，因为pos是描述data块的读写指针，所以pos ∈ \[0, size),也需要考虑是用uint类型还是ushort类型。


**(5\.4\)**在初始化大块的时候为什么要`*((uint*)(p)) |= 0x80000000`？
这里就是我们前面说的如何确定buffer是大块还是小块问题的关键。
0x80000000转换为10进制是231，由于大块的`size`、`pos`是uint，所以有32位且无符号位，231刚好占据的是uint的最高位。
让p强转为uint\*类型后又用\*取内容得到uint类型的值（实际上就是uint类型的p\[0]），接着将其\|\= 0x80000000使得最高位为1。


那具体这个最高位为1是怎么用于判断大小块的呢？



```
#define __is_big_block(p) (0x80000000 & *((uint*)(p)))

```

我们将p\[0]转为uint类型，接着与0x80000000进行相与。


* 如果是大块，由于我们`__init_b_block`的时候将p\[0]最高位置1，与0x80000000相与的结果就是1，对应的`__is_big_block`返回值是1
* 如果是小块，由于\&操作是按位进行，所以最高位为0，而后面的位由于0x80000000全为0得到的也是0，对应的`__is_big_block`返回值是0
我们便通过位运算而不是进行`size >= 0x80000000`的判断从而快速确定buffer是大块还是小块。




---


(6\)了解完`__init_b_block`的宏定义后，我们还有一个问题没有解决，那就是为什么要将`bufptr`取原始指针后再转化为`any_ptr`？
首先我们得知道智能指针中的unique\_ptr有**独占所有权**的概念，而uint\*与ushort\*都是没有所有权管理的普通指针，所以不能进行转换。
但是unique\_ptr给我们提供了get()成员函数，允许我们**不转移所有权**的使用原始指针，而原始指针是可以转换成我们需要的uint\*或者ushort\*的。
因此我们需要先调用`bufptr.get()`取出原始指针，然后转换为void\*类型的any\_ptr，再根据需要转换为uint\*或者ushort\*。


# 4\.buffer数据的写入


## 4\.1 byte数据的写入



```
void buffer::put(byte b) {
    if (size() - pos() < sz_byte) {
        throw std::overflow_error("insufficient buffer to store byte");
    }

    byte* d = data();
    *d = b;
    __mv_fw_block(this, sz_byte);
}

```

再具体解释怎么写入之前，我们先把代码里面的陌生函数解释一遍。




---


(1\)`size()`函数



```
size_t buffer::size() const {
    return (size_t)(__size_of_block(this));
}

```

`__size_of_block`的宏定义是



```
#define __size_of_block(p) (__is_big_block(p)) ? (*((uint*)(p)) ^ 0x80000000) : *((ushort*)(p))

```

* 如果是大块，就讲转成int类型的p\[0]异或上0x80000000。前面我们说过大块的p\[0]需要 \|\= 0x80000000将最高位置1达到快速判断大小块的目的。在获取大块真实的p\[0]表示的size数据时，我们需要反过来取异或将1消掉得到真实的size。
* 如果是小块，则直接取ushort类型的p\[0]


(2\)`pos()`函数



```
size_t buffer::pos() const {
    return (size_t)(__pos_of_block(this));
}

```

对应的宏定义是



```
#define __pos_of_s_block(p) ((ushort*)(p))[1]
#define __pos_of_b_block(p) ((uint*)(p))[1]

```

根据大块还是小块选择uint或者ushort类型的p\[1]


(3\)`data()`函数



```
byte* buffer::data() const {
    return __data_of_block(this);
}

```

对应的宏定义：



```
#define __data_of_block(p) (__is_big_block(p)) ? (byte*) (((byte*)(((uint*)(p)) + 2)) + __pos_of_b_block(p)) : (byte*) (((byte*)(((ushort*)p) + 2)) + __pos_of_s_block(p))

```

这里的\_\_data\_of\_block有点复杂，我们以大块为例逐步分解来看，小块同理。


* 首先通过`(uint*)(p)`将p转成uint\*类型,然后再此基础上 \+ 2(2个uint的字节)。根据前面buffer的总体架构我们知道，buffer前两个区域是`size`与`pos`。大块的size 与 pos均为uint类型，将p转成uint\*然后再 \+ 2便可以实现跳转到data块的开始处。
* 但是读写buffer的过程中读写指针会变化，比如我们写入了1，然后又想写入0，如果还是从data开始处写的话会直接覆盖开始的1，只有从1的末尾继续写0才合理，换句话说我们要实现追加(append)模式。因此`((byte*)(((uint*)(p)) + 2)) + __pos_of_b_block(p))` 通过 `+__pos_of_b_block(p)` 跳转到当前读写指针的位置。
小块也是同理，只不过从uint\*换成ushort\*。


通过这两步我们便可以得到当前读写指针所在位置的指针。


(4\)`__mv_fw_block(this, sz_byte);`
这是一个宏定义



```
#define __mv_fw_block(p, d) if(__is_big_block(p)){\
    ((uint*)(p))[1] += (d);\
    }\
    else{\
    ((ushort*)(p))[1] += (ushort)(d);\
    }

```

每次读或者写buffer的时候，我们都要更新p\[1]代表的`pos`，实现流的读入或者流的写入。
比如说要读入12345，我们读了1，让pos \+\= 1,这样再读就可以读到2。如果一次读了123， 就让pos \+\= 3,下次再读就可以从4开始。写也是同理。




---


介绍完这几个函数后，我们再回到byte数据的写入。


* 首先判断写入的数据是否超过buffer的大小，如果是，抛出overflow异常
* 否则取到当前读写指针所在位置的`byte *d = data();`
* 通过`*d = b`写入字节型数据b
* `__mv_fw_block(this, sz_byte);`更新读写指针


## 4\.2 int32类型数据的写入


与简单的byte数据直接写入不同，多字节型数据写入有着顺序上的讲究。
我们先来看源码：



```
void buffer::put(int32 val) {
    if (size() - pos() < sz_int) {
        throw std::overflow_error("insufficient buffer to store int32");
    }

    byte* d = data();
    for (size_t i = 0; i < sz_int; ++i) {
        *(d + i) = (byte)(val >> (i * 8));
    }

    __mv_fw_block(this, sz_int);
}

```

前面都与byte数据的写入大同小异，重点是后面的for循环



```
    for (size_t i = 0; i < sz_int; ++i) {
        *(d + i) = (byte)(val >> (i * 8));
    }

```

* `sz_int`是指**int的字节个数**，for循环遍历了要写入的`int val`的每一个字节。
* `*（d + i）`是给从d开始数的第i个字节的位置赋值。
* `(byte)(val >> (i * 8));` 是将val右移了`i * 8`位，然后通过byte转换取到右移后的低8位。


合起来就是`从d开始数的第i个字节的位置赋值成val右移了i * 8位的低8位`。
我们具体举例来说明：
比如要放入1010001111111110010（10进制 \= 327658）。


* 第一个i \= 0， 放入11110010 到buffer的d\[0]。
* 第二次i \= 1， 放入01111111 到buffer的d\[1]。
* 第三次i \= 2， 放入00010100 到buffer的d\[2]。
* 第四次i \= 3， 放入00000000 到buffer的d\[3]。


**很明显可以看到这是将val按字节进行了逆序存储，为什么不直接按原顺序正向存储呢？**
答案就是因为难，我们进行字节的转换是通过（byte）来转的，而这种转换是截取**低8位**而非**高8位**，所以我们for循环遍历val的每个字节时只能做到逆序存储val。


## 4\.3 ulong类型数据的写入



```
void buffer::put(ulong val) {
    if (size() - pos() < sz_ulong) {
        throw std::overflow_error("insufficient buffer to store int32");
    }

    byte* d = data();
    for (size_t i = 0; i < sz_ulong; ++i) {
        *(d + i) = (byte)(val >> (i * 8));
    }

    __mv_fw_block(this, sz_ulong);
}

```

解析同4\.2，for循环遍历val的每个字节，并且逆序存储。


## 4\.4 string类型数据的写入



```
void buffer::put(const std::string& str) {
    if (size() - pos() < (str.length() + 1)) {
        throw std::overflow_error("insufficient buffer to store a string");
    }

    byte* d = data();
    for (size_t i = 0; i < str.length(); ++i) {
        *(d + i) = (byte)str[i];
    }

    *(d + str.length()) = (byte)0;
    __mv_fw_block(this, str.length() + 1);
}

```

* 由于**string类型每一个元素都是char类型**（byte其实就是unsigned char）,所以即使string也是多字节数据，但是其不需要像int32或者ulong逆序存储。
* `*(d + str.length()) = (byte)0;`将最后一个字节置为0，表示字符串的终止，与“hello World!”等c\-string在末尾自动添加'\\0'同理。


## 4\.5 buffer类型数据的写入



```
void buffer::put(const buffer& buf) {
    size_t sz = size();
    size_t p = pos();
    size_t src_sz = buf.size();
    size_t src_p = buf.pos();
    if ((sz - p) < (src_sz - src_p)) {
        throw std::overflow_error("insufficient buffer to hold the other buffer");
    }

    byte* d = data();
    byte* src = buf.data();
    ::memcpy(d, src, src_sz - src_p);
    __mv_fw_block(this, src_sz - src_p);
}

```

通过memcpy实现deep copy。


# 5\.buffer数据的读取


## 5\.1 byte数据的读取



```
byte buffer::get_byte() {
    size_t avail = size() - pos();
    if (avail < sz_byte) {
        throw std::overflow_error("insufficient buffer available for a byte");
    }

    byte val = *data();
    __mv_fw_block(this, sz_byte);
    return val;
}

```

* 先判断buffer是否有足够空间，没有就抛出overflow异常
* 直接调用data()，获取读写指针所在位置，并用\*符号取其内容，得到所需val
* `__mv_fw_block(this, sz_byte);`更新读写指针


## 5\.2 int32数据的读取



```
int32 buffer::get_int() {
    size_t avail = size() - pos();
    if (avail < sz_int) {
        throw std::overflow_error("insufficient buffer available for an int32 value");
    }

    byte* d = data();
    int32 val = 0;
    for (size_t i = 0; i < sz_int; ++i) {
        int32 byte_val = (int32)*(d + i);
        val += (byte_val << (i * 8));
    }

    __mv_fw_block(this, sz_int);
    return val;
}

```

重点关注for循环，从d开始遍历一个int32的所有字节。


* 对每个字节取出`byte_val`
* 将`byte_val`左移 `i * 8`位后累加到`val`
* 返回`val`


在前面**4\.2 int32数据的写入**中我们已经说过多字节数据是逆序存储的(除string类型）


我们再以1010001111111110010（10进制 \= 327658）来举例说明
由于逆序存储buffer中数据应该是11110010\|01111111\|00010100 \|00000000


* i \= 0, 取出byte\_val \= 11110010， 这个byte\_val应该在原始数据里面占据0\-7位，因此我们左移 0 \* 8 \= 0位并累加到val
* i \= 1, 取出byte\_val \= 01111111， 这个byte\_val应该在原始数据里面占据8\-15位，因此我们左移 1 \* 8 \= 8位并累加到val
* i \= 2, 取出byte\_val \= 00010100， 这个byte\_val应该在原始数据里面占据16\-23位，因此我们左移 2 \* 8 \= 16位并累加到val
* i \= 3, 取出byte\_val \= 00000000， 这个byte\_val应该在原始数据里面占据24\-31位，因此我们左移 3 \* 8 \= 24位并累加到val


最后我们便得到正常顺序的数据val


## 5\.3 ulong数据的读取



```
ulong buffer::get_ulong() {
    size_t avail = size() - pos();
    if (avail < sz_ulong) {
        throw std::overflow_error("insufficient buffer available for an ulong value");
    }

    byte* d = data();
    ulong val = 0L;
    for (size_t i = 0; i < sz_ulong; ++i) {
        ulong byte_val = (ulong)*(d + i);
        val += (byte_val << (i * 8));
    }

    __mv_fw_block(this, sz_ulong);
    return val;
}

```

分析同5\.2，只是遍历的字节数从sz\_int 变为sz\_ulong


## 5\.4 string类型的读取



```
const char* buffer::get_str() {
    size_t p = pos();
    size_t s = size();
    size_t i = 0;
    byte* d = data();
    while ((p + i) < s && *(d + i)) ++i;
    if (p + i >= s || i == 0) {
        return nilptr;
    }

    __mv_fw_block(this, i + 1);
    return reinterpret_cast<const char*>(d);
}

```

string类型数据的读取有点不一样。



```
while ((p + i) < s && *(d + i)) ++i;
    if (p + i >= s || i == 0) {
        return nilptr;
    }

```

* 首先我们不知道string类型数据的长度，我们只知道其以0结尾。所以要用while来一直取
* while里面判断0直接用 `while(*(d + i)) ++i`就行了，为什么还要加一个`(p + i) < s`呢？
因为只有`*（d + i）`我们并不清楚自己是否会越界，可能由于某些原因即使遍历完buffer依旧没找到0，
这个时候就需要我们再加一个`(p + i) < s`保证不会越界
* 如果`p + i >= s`说明越界， `i == 0`说明根本没数据,都应该返回nilptr



```
    __mv_fw_block(this, i + 1);
    return reinterpret_cast<const char*>(d);

```

最后调整读写指针的`__mv_fw_block(this, i + 1);`是`i + 1`个字节而不是`i`的原因:
假设string \= "abc"（以'\\0'标记结尾）,i \= 2的时候while条件依旧满足，i\+\+→i \= 3
不妨设没读入string前的pos \= 0， 这时候`__mv_fw_block`将pos \+\= (i \+ 1\)→pos \= 4
表示下次读写操作都要从pos \= 4开始。即跳过了"abc"与'\\0'后的第一个位置。
如果`__mv_fw_block(this, i + 1);`是`i`个字节，则表示下次读写操作要从'\\0'开始，会改变'\\0'导致无法该字符串无法被识别。


## 5\.5 读取数据并存到指定的buffer里面



```
void buffer::get(bufptr& dst) {
    size_t sz = dst->size() - dst->pos();
    ::memcpy(dst->data(), data(), sz);
    __mv_fw_block(this, sz);
}

```

* 先计算要copy的字节数`sz`,`dst->size() - dst->pos();`表示是dst所剩的所有字节
* `::memcpy(dst->data(), data(), sz);` 将data()开始往后数`sz`字节的数据全部拷贝到dst里面
* 调整读写指针 `__mv_fw_block(this, sz);`


# 6\.总结


* 1\.在判断大小块面前，我们巧妙的运用最高位通过位运算来快速判断。
* 2\.buffer最重要的两个操作便是`读(get)` 与 `写(put)`，在面对多字节数据的时候我们可以看到逆序存储的想法。
* 3\.面对智能指针需要转换的情况，我们应该先采取`get()`得到原始指针，然后再通过`reinterpret_cast`进行转换。


  * [1\.概览：](#tid-CEYbrC)
* [2\.buffer的总体架构](#tid-a3Gjjp)
* [3\.buffer的内存分配](#tid-36Wert)
* [4\.buffer数据的写入](#tid-dmHbmD)
* [4\.1 byte数据的写入](#tid-MG4TBS)
* [4\.2 int32类型数据的写入](#tid-BscaMA)
* [4\.3 ulong类型数据的写入](#tid-hw3SBy)
* [4\.4 string类型数据的写入](#tid-GW3ffB)
* [4\.5 buffer类型数据的写入](#tid-frrbrX)
* [5\.buffer数据的读取](#tid-8sC7Ap):[milou加速器](https://xinminxuehui.org)
* [5\.1 byte数据的读取](#tid-kYBAsm)
* [5\.2 int32数据的读取](#tid-e7nPrs)
* [5\.3 ulong数据的读取](#tid-swHcSb)
* [5\.4 string类型的读取](#tid-Ac4tn8)
* [5\.5 读取数据并存到指定的buffer里面](#tid-TJ2i6z)
* [6\.总结](#tid-REDTNr)

   \_\_EOF\_\_

       - **本文作者：** [TomGeller](https://github.com)
 - **本文链接：** [https://github.com/Tomgeller/p/18461473](https://github.com)
 - **关于博主：** 评论和私信会在第一时间回复。或者[直接私信](https://github.com)我。
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
