---
title: Go 1.24 Map 重构：Swiss Table 与代码细节
tags: Go
key: go-map-swiss
---

Go语言在1.24版本重构了其map的数据结构，使用开源的、被Rust 1.36内置的Swiss Table作为替代。官方宣称其实现了：map操作10~60%的性能提升、被测程序整体1.5%的性能提升。本文我们将深入讨论原map的实现与该数据结构的思路，以及使用Go源码进行相应说明。
<!--more-->

# TL;DR 太长不看

- Go语言在1.24版本重构了其 map 数据结构，使用 Google 于 2017年[开源](https://abseil.io/blog/20180927-swisstables)的 [Swiss Table](https://www.youtube.com/watch?v=ncHmEUmJZf4) 作为其底层算法实现。[Go语言官方博客](https://go.dev/blog/swisstable)宣称查询（对于Key较多的情况）、增加与遍历性能都有[10+%的提升](https://github.com/golang/go/issues/54766#issuecomment-2542444404)，用于测试的**应用程序整体运行性能**有1.5%的提升。
- Go语言在1.23版本及之前使用“拉链法”作为 map 的[底层逻辑](https://draven.co/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/#哈希函数)——即碰撞后通过溢出桶存储Key——但逐个比较碰撞的Key速度较慢。
- 与之对比的是[新算法](https://github.com/golang/go/blob/master/src/internal/runtime/maps/map.go) Swiss Table，将64位的 hash 分为两部分 h1（前 57 bits）和 h2（后7bits）：
	- 使用 h1 确定包含 $2^k$ 个桶的、具体的 hash group
	- 然后使用平方探测法（quadratic probing）确定要检索的[桶序列](https://github.com/golang/go/blob/master/src/internal/runtime/maps/group.go)
	- 最后依次在每个桶中优先使用 h1 进行批量初步比对，只有当 h1 匹配的时候才对完整的 Key 进行比对（概率为 $1/2^7 = 1/128$），从而无需遍历整个桶
		- 因为每个桶包含 8 个槽位的 h1，故可以使用`uint64`单次比对8个元素
		- 使用SIMD指令可单次对比更多元素

## Swiss Table 示意图

```
hash: 1111 ... 1111 1222 2222
      |----- h1 ----||- h2 -|
h1 57 bits 用来选择桶
h2  7 bits 用来初步比较：两个hash的h2不一致说明一定不匹配

ctrl word: 用于初步比较，标志位加上h2
empty  : 1 0 0 0 0 0 0 0  (0b10000000)
deleted: 1 1 1 1 1 1 1 0  (0b11111110)
full   : 0 h h h h h h h  (0 加上 H2 的 7 bits)

ctrl group: 每个桶8个槽位的标志位加h2，用于批量比对，例如
Test word	    89	89	89	89	89	89	89	89
Comparison	    ==	==	==	==	==	==	==	==
Control word	23	89	50	-	-	-	-	-
Result	        0	1	0	0	0	0	0	0
```
# 回顾：Go 1.23 及之前的 Map 实现（拉链法）

在 Go 1.24 之前，Go 的 `map` 底层采用的是经典的**拉链法哈希表 (Chaining Hashing)**。其核心思想是将键值对存储在一系列“桶”（Buckets）中，并通过链表解决哈希冲突。

## 核心数据结构 bmap 的说明

### bmap (Bucket)：每个桶的实际结构

```go
const (
	// Maximum number of key/elem pairs a bucket can hold.
	bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits // bucketCnt == 8
) // 即，每个桶最多8个元素

// A bucket for a Go map.
type bmap struct {
	// 存储每个 Key hash 的头 8 bits 用于快速对比查询的Key
	tophash [bucketCnt]uint8 

	// ... 存储空间与溢出桶的指针
}

// 返回溢出桶的指针
func (b *bmap) overflow(t *maptype) *bmap {
	// Access the overflow pointer stored at the end of the bucket struct.
	return *(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-sys.PtrSize))
}
        ```

## 哈希与寻址

简而言之，使用哈希函数与map实例的随机种子计算 Key 的 hash 值；其中
* hash 值的低 B 位用于选择主桶
```go
hash := t.hasher(key, uintptr(h.hash0))
m := bucketMask(h.B) // bucketMask(B) == (1 << B) - 1
b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize))) 
```
* hash值的高 8 位用于 `tophash` 值便于快速对比
```go
func tophash(hash uintptr) uint8 {
	top := uint8(hash >> (sys.PtrSize*8 - 8))
	if top < minTopHash { // Avoid reserved values
		top += minTopHash
	}
	return top
}
--- 
top := tophash(hash)  
bucketloop:  
    for ; b != nil; b = b.overflow(t) {  
       for i := uintptr(0); i < abi.MapBucketCount; i++ {  
          if b.tophash[i] != top {  
             if b.tophash[i] == emptyRest {  
                break bucketloop  
             }  
             continue  
          }
```
## 冲突处理：拉链法

* 当一个桶的 8 个槽都满了，会通过 `h.newoverflow(t, b)` 创建一个新的 `bmap` 作为溢出桶，并将其链接到前一个桶的末尾
* 查找、插入、删除操作在检查完主桶后，会沿着 `b.overflow(t)` 指针遍历溢出桶链

## 小结

可以从上方简介看出，1.23及之前版本的 map 逻辑清晰，但因为使用拉链法会导致潜在的溢出桶的逐桶遍历——这也为 1.24 版本的重构提供了可优化的空间。另外，拉链法因为使用指针指向下一个桶，故逐溢出桶遍历的时候并不具有缓存友好的特性。

# 重构的直观想法

我想请读者先想一想：现在我们了解到了原有 map 可优化的点，那么我们可以怎么进行优化？

## 批量比对

留意到，原有 map 每个桶留有8个字节的 `tophash`，用于**初步判断**查找的 Key 是否存在于桶中（如果`tophash`一致，那么 Key 可能一致；否则 Key 一定不一致）。我们可以将这 64 字节看做一个 int64，然后直接每字节与 Key 的 `tophash` 进行位操作，用于初步判断。
比如，这8个`tophash`一起构成了`0x01020304050607`这个 int64，我们要查询的 Key 的 `tophash` 是 `0x00`，按位与我们就知道这个桶不包含该 Key。当然，详细的位运算操作我们后面会讲到。
但是直接使用这 8 个 `tophash` 会有问题，我们无法直接区分该桶是否已满，或对应 Key 是否已删除——我们将引入其他的流程或数据来作为该槽位是否空余的标志。

## 更快的批量比对

假设我们能把同一系列桶（即 hash 值低 B 位相同的 Key 所指向的桶）所有 `tophash` 集中到一起，那我们就可以通过 SIMD 一次性进行至少 16 个 `tophash` 的初步判断。

我们可以进一步优化，这一系列桶一定包含共 $2^k\ (k \in \mathbb{N^*})$ 个桶，这样便于 SIMD 操作。
如果加大了桶数量（而不是逐个溢出），那我们就需要考虑调整根据 hash 选择桶的策略，让更多的 Key 放到这一系列的桶中，从而提高内存的利用率。

看到这里，恭喜你的直观想法和 Swiss Table 的理念不谋而合了！

# 重构：Go 1.24 Swiss Table 实现详解

Go 1.24 转向了基于 **Swiss Table** 的**开放寻址法哈希表**。其核心优势在于利用 CPU 特性（特别是位运算和 SIMD）加速探测过程。

我们先简单讲一下 map 从下到上究竟包含哪些东西：
- `Slot` **槽位**：存储单独的 Key/Value 对；每个 Key 对应于两部分 hash：
	- `H1`：头 57 bit
	- `H2`：后 7 bit，用于快速比对
- `Group` **桶**（为方便对比，我们还是沿用原有同概念的中文名），包含：
	- 8个槽位（后续可能扩展到更少/更多槽位——32位系统可以使用4槽位，为了便于使用SIMD指令，可以拓展为例如16槽位）
	- `Control word`（简称为`ctrl`），每个槽位对应8 bit的控制位，所有槽位的控制位组成`crtl`，用于标记槽位是否占用/删除以及快速比对（类似于`tophash`）
		- 该 8 位控制位包含 1 位标记位和 7 位的 `h2`
- `Table` **表**，即一个完整的 Swiss Table，包含多个使用开放寻址法的桶以及元信息
- `Directory` **目录**，包含多个表，便于快速 map 局部快速扩容

## 核心结构：Group 和 Control Word

底层数据存储在一个逻辑上划分为多个 **Group** 的数组中。每个 Group 包含 8 个键值对槽（Slots）和关键的 **Control Word**。

### Control Word

每个槽位的 8 位控制位共同组成 `ctrl` word，这 8 位描述如下：
```go
// states: empty, deleted, and full. 
// They have the following bit patterns:
//  empty  : 1 0 0 0 0 0 0 0  (0b10000000)
//  deleted: 1 1 1 1 1 1 1 0  (0b11111110) - Tombstone 墓碑
//  full   : 0 h h h h h h h  - H2 的 7 bits
type ctrl uint8

const (
	ctrlEmpty   ctrl = 0b10000000
	ctrlDeleted ctrl = 0b11111110
)

// ctrlGroup 即 control word，目前被存储为 uint64，便于整体操作
type ctrlGroup uint64

```

### Group 桶

目前每个桶是固定的 8 槽位，且包含 ctrl：
```go 
type group struct {
 ctrls ctrlGroup
 slots [abi.SwissMapGroupSlots]slot
}

type slot struct {
 key  typ.Key
 elem typ.Elem
}
```

## 哈希值的使用

每个 Key 的 64 位哈希值被拆分为 `h1` (高 57 位) 和 `h2` (低 7 位)。
其中，`h1`用于确定初始的 Group 和后续的探测序列。

```go
// Extracts the H1 portion of a hash: the 57 upper bits.
func h1(h uintptr) uintptr {
	return h >> 7
}

// Extracts the H2 portion of a hash: the 7 bits not used for h1.
// These are used as an occupied control byte.
func h2(h uintptr) uintptr {
	return h & 0x7f
}
```

### 快速比对

给定 Key 的 64 位 hash 和 桶的 `ctrl`，如何确定元素是否可能在桶中？我们来看看 Go 的源码：
```go
const ctrlEmpty   ctrl = 0b10000000  // 0x80
const ctrlDeleted ctrl = 0b11111110  // 0xFE

const bitsetLSB     = 0x0101010101010101  
const bitsetMSB     = 0x8080808080808080

func ctrlGroupMatchH2(g ctrlGroup, h uintptr) bitset {
	// XOR each byte with the target h2
	v := uint64(g) ^ (bitsetLSB * uint64(h)) 
	// Check which bytes became zero (match) and also ensure the high bit is 0 (full slot)
	return bitset(((v - bitsetLSB) &^ v) & bitsetMSB)
}

// bitset 表示最终结果，每 8 bit 有两种情况：
// 0x80 -> h2 match; 0x00 -> not match
// 为什么如此设计？后文会提到
type bitset uint64
```

我们拆解一下源码，然后给出例子方便理解：
```go
// 为方便分析，我们假设桶仅包含2个槽位，现在仅一个元素占据该桶，其 h2 为 0x10
// 相应地，bitsetLSB = 0x0101，bitsetMSB = 0x8080
// 现在我们要查找的 Key h2 为 0x10

/* v := uint64(g) ^ (bitsetLSB * uint64(h)) */
g := 0x8010 // 0x80 -> empty; 0b 0 001 0000 -> flag: 1 full, h2: 001 0000
a := bitsetLSB * h // 0x0101 * 0x10 = 0x1010
v := g ^ a // ^: XOR
// g: 0b 1000 0000 0001 0000
// a: 0b 0001 0000 0001 0000
// v: 0b 1001 0000 0000 0000

/* ((v - bitsetLSB) &^ v) & bitsetMSB */
m := v - bitsetLSB
// v        : 0b 1001 0000 0000 0000
// bitsetLSB: 0b 0000 0001 0000 0001
// m        : 0b 1000 1110 1111 1111

c := m &^ v // &^ bit clear (AND NOT)
// m        : 0b 1001 0000 0000 0000
// v        : 0b 0000 0001 0000 0001
// c        : 0b 1000 1110 1111 1111

ret := c & bitsetMSB
// c        : 0b 0000 1110 1111 1111
// bitsetMSB: 0b 1000 0000 1000 0000
// ret      : 0b 0000 0000 1000 0000
// meaning  :   | UNMATCH |  MATCH  |
```

备注：源码中说明了`ctrlGroupMatchH2`可能给出false-positive（实质不match但返回match）的情况，即两个元素的 h2 依次为 `2^N+1` 和 `2^N`的时候，例如 `0x0302`，如果查询的 h2 为`0x02`，那么会返回`0x8080`——意味着两个槽位都匹配。
#### 获取匹配的槽位

假设 `match := ctrlGroupMatchH2(g, h)`且不为0（表示有槽位 match 到了给定输入 h2），那么怎么依次获取是第几个槽位？

```go
func bitsetFirst(b bitset) uintptr {  
    return uintptr(sys.TrailingZeros64(uint64(b))) >> 3  
}

// TrailingZeros64 返回x中结尾0的个数；实际上 AMD64 机器用底层指令实现了该函数
func TrailingZeros64(x uint64) int {
	// 位运算具体可见(Knuth, volume 4, section 7.3.1)
}
```

那么`0x0080`的末尾共有 7 个 0，故得到`0`；如果是`0x8000`，末尾共 15 个 0，故得到`1`。

## 探测序列 (Probe Sequence)

如果当前 Group 未命中，则使用平方探测法查找下一个 Group。
```go
// probeSeq 维持当前探测到哪个桶的状态
// p(i) := (i^2 + i)/2 + hash (mod mask+1)
type probeSeq struct {
	mask   uint64 // 当前桶数量减一
	offset uint64 // 当前探测的桶的索引
	index  uint64 // 桶步数
}

func makeProbeSeq(hash uintptr, mask uint64) probeSeq { // hash for h1
	return probeSeq{
		mask:   mask,
		offset: uint64(hash) & mask, // 根据 h1 来选择桶
		index:  0,
	}
}

func (s probeSeq) next() probeSeq {
	s.index++
	s.offset = (s.offset + s.index) & s.mask
	// 每次探测都会使得步数加 1
	// index:  0 -> 1 -> 2 -> 3 -> 4  -> 5
	// offset: 0 -> 1 -> 3 -> 6 -> 10 -> 15
	// 即平方探测法
	return s
}
```

最终，当我们遍历抵达了一个拥有空槽位的桶时，我们认为 Key 没有匹配。
因此，当删除元素时，我们不能直接将该槽位清空，而可能需要将该槽位标记为已删除状态。
## 删除 (Tombstone Logic)

删除时，检查当前 Group 是否有空槽：
- 如果有，则意味着该桶为探测序列的末尾，可以直接将删除的槽标记为 `ctrlEmpty`；
- 如果没有空槽（即 Group 已满），则必须标记为 `ctrlDeleted`，以保证后续的探测能正确越过这个位置。

## 增量式增长

Swiss Table 在扩容时，需要整个表进行扩容（通常意味着数组大小翻倍）；显然，如果一个表占用了例如 `1GB` 的内存，让延时敏感的 Go 语言去扩容这部分内存到 `2GB` 是不可接受的。

因此，Go 语言使用类似于原有的逻辑，使用可扩展 hash 的方式，将整个 map 分割为多个目录，每个 table 不超过 1024 个槽位；这样，每次扩容只需要扩容至多 2048 个槽位的内存。

具体来说，Go 使用 hash 值的最头部的 `X` bit 来映射到目录。比如有两个 table 的时候，最头部的 1 bit，如果为 `0` 则映射到第 0 个 table，为 `1` 则映射到第 1 个 table。
现在，假设 table 1 要扩容，则我们将得其分割为 table 1 和 table 2。因此，我们引入“目录 Directory”的概念进行管理：

```go
//  directory (globalDepth=2)  
//  +----+  
//  | 00 | --\  
//  +----+    +--> table (localDepth=1)  
//  | 01 | --/  
//  +----+  
//  | 10 | ------> table (localDepth=2)  
//  +----+  
//  | 11 | ------> table (localDepth=2)  
//  +----+
```

# 性能与展望

根据[测试](https://github.com/golang/go/issues/54766#issuecomment-2542444404)，[Go语言官方博客](https://go.dev/blog/swisstable)宣称查询（对于Key较多的情况）、增加与遍历性能都有[10~60%的提升](https://github.com/golang/go/issues/54766#issuecomment-2542444404)，用于测试的**应用程序整体运行性能**有1.5%的提升。

很显然，我们在“重构的直观想法”一节画的饼还没有都实现。比如，SIMD还没引入，现在只是用 64 位寄存器的操作；Swiss Table 的 C++ 实现，还包括把table内所有桶的 control word 集中起来，更缓存友好，且可以用SIMD指令集实现更大规模的初步比对。

# 总结

请看文首的“太长不看”内容。