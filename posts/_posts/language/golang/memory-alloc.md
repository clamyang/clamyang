---
title: Go 内存分配器
date: 2022-06-18
comments: true
---

![](https://s2.loli.net/2022/07/24/SYqz9rCngZNVdAp.png)

梳理 Golang 内存分配器，以前通过代码调试过，有了一个初步的认识，这次把一些细节性的东西都梳理出来了，发现自己对内存分配的过程有了更清晰的认识，很多内容靠脑子想还是太费事了，就咱这个“处理器”差一点就烧坏了.. 把图画出来能够有效的缓解大脑负载..

<!--more-->

## Unmanaged memory（堆外内存）

在看 Go 内存源码部分的时候，涉及到管理内存的相关数据结构时，总能看到这样一行 comment， `go:notinheap` 就感觉挺奇怪的，内存地址空间除了堆就是栈，那要存储在哪里呢？



GC 扫描的时候是不需要对堆外内存进行扫描的，官方给出了堆外内存存在的两个必要性：

- 当这块内存是 memory manager 本身时。
- 在调用者没有 P 的情况下分配，调度器初始化之前是不存在 P 的。

同时，官方也给出了声明 unmanaged memory 的三种方式

1. sysAlloc
2. persistentalloc
3. fixalloc

### go:notinheap

类型声明的时候使用，表示这个对象禁止在 gc 扫描的堆上或者栈上分配。尤其是，指向这个对象的指针在 `runtime.inheap` 检查时总是返回假。

> go:notinheap 真正的意义：runtime 在某些底层数据结构上使用，避免了内存屏障，提高了性能。

## basic concept

Go 语言中的内存管理同 tcmalloc 类似，将从操作系统那里得到的内存按照大小不同的块进行管理，形如`segregated list`，`mspan` 是 Go 语言中最基本的内存管理单元。

```go
//go:notinheap
type mspan struct {
    next *mspan     // next span in list, or nil if none
    prev *mspan     // previous span in list, or nil if none
    // ... 省略了大量字段
    // 指向了实际的 chunk
    tiny       uintptr
    // 表明这个 chunk 用了多少
    tinyoffset uintptr
    // 表明 tiny allocation 执行的次数
    tinyAllocs uintptr
    alloc [numSpanClasses]*mspan // spans to allocate from, indexed by spanClass
    startAddr uintptr // address of first byte of span aka s.base()
    npages    uintptr // number of pages in span

    // 每一次分配，都会从 freeindex 开始扫描 allocbits
    // 直到遇到 0，说明这是一个空闲块。
    // 当 freeindex == nelemes 时，说明该 mspan 满了
    // 当 n >= freeindex and allocBits[n/8] & (1<<(n%8)) is 0
    // 说明 object n 是空闲的
    freeindex uintptr

    nelems uintptr // number of object in the span.

    // allocBits 的补码
    allocCache uint64

    allocBits  *gcBits
    gcmarkBits *gcBits

    // 已经分配的对象数量
    allocCount  uint16
    spanclass   spanClass     // size class and noscan (uint8)

    elemsize    uintptr       // computed from sizeclass or from npages
}
```

这里把该数据结构中有关内存分配相关的字段截取出来，不难看出，该结构体通过 `next` `prev` 构成了一个双向链表的数据结构。在整个内存分配过程中，是没有用到这两个字段的，他们的具体作用还有待考究。



`mspan` 中包含了很多个 chunk，更像是一个数组，正如数组的定义一样，内部的元素是 size 相同的 chunk，如下图：

![](https://s2.loli.net/2022/07/23/IGgUj3sm2NnwpKb.png)

Go 中的 pages 是 OS page 的整数倍，通常每个页面为 8KB。图中的 `mspan` 由 5 个 page 组成，5 * 8 = 40 KB，我们申请一个 10KB 的切片 `slice := make([]byte, 10240)` 

![](https://s2.loli.net/2022/07/23/DRVmvIzYd9jJGOL.png)

详细的步骤我们一步步展开，你可能已经注意到了，图中标识了当前 `mspan` 的 `spanclass = 109`，我这里的 go 版本为：`1.16` 有 136 种 spanclass，更进一步，spanclass 分为了：

- scan class

  需要扫描的类别，那什么情况下需要使用 scan class 呢？当 chunk 中有指针时，就需要使用 scan class。

- noscan class

  通常来讲标量类型都是不需要扫描的。
  
  

spanclass 其实就是 uint8 的一个别名，通过最后一个 bit 进行判断：

```go
func (sc spanClass) noscan() bool {
    return sc&1 != 0
}
```

通过 `sc&1 != 0` 就可以判断出：

- spanclass为偶数时，需要
- spanclass为奇数时，不需要

### Tiny Small Large

Go 把内存对象分成三类

| 对象  | 范围          |
| ----- | ------------- |
| Tiny  | (0, 16 B)     |
| Small | [16 B, 32 KB] |
| Large | (32 KB, +∞)   |

在上边的例子中，我们创建了一个 small object，那么为什么要根据对象大小进行区分呢？为了 alloc performance。



### Mcache Mcentral Mheap

这里就是它借鉴了 tcmalloc 中的一点，实现了线程缓存，这样在每个线程上分配时就不需要加锁。当本地处理不了时就需要向上一层一层尝试，当 mheap 都分配不出来了，说明内存用光了。

| 名称     | 作用                                                         |
| :------- | ------------------------------------------------------------ |
| mcache   | 线程缓存<br />用于 tiny,small 对象的分配                     |
| mcentral | 中心缓存<br />当 mcache 中没有足够空间时会用到 mcentral      |
| mheap    | 堆上内存<br />当 mcentral 中没有足够空间时会用到 mheap<br /> |

- mcache 中比较重要的字段

  ```go
  //go:notinheap
  type mcache struct {
  	// ... 省略了大量字段
      // 指向了实际的 chunk，负责 tiny 对象的分配
      tiny       uintptr
      // 表明这个 chunk 用了多少
      tinyoffset uintptr
      // 表明 tiny allocation 执行的次数
      tinyAllocs uintptr
  	// 负责 small 对象的分配
      alloc [numSpanClasses]*mspan // spans to allocate from, indexed by spanClass
  }
  ```

- 当我们学习 large 对象分配的时候再深入研究 mcentral mheap 也不迟

## Allocator

有了基本的概念，我们就可以更深入的了解内存分配器是如何工作的，首先我们要探索的是 `tiny allocator` ，如果你写不出合适的单元测试，那么官方的 test 就是你的最佳选择。

```go
func TestTinyAlloc(t *testing.T) {
    const N = 16
    var v [N]unsafe.Pointer
    for i := range v {
        v[i] = unsafe.Pointer(new(byte))
    }

    chunks := make(map[uintptr]bool, N)
    for _, p := range v {
        // a &^ b 的意思就是 清零a中，ab都为1的位
        // 0110 &^ 1011 -> 0100
        // 按照 8 字节进行对齐
        chunks[uintptr(p)&^7] = true
    }

    if len(chunks) == N {
        t.Fatal("no bytes allocated within the same 8-byte chunk")
    }
}
```

`tiny allocator` 在分配前需要根据 size 计算字节对齐，然后移动 tinyoff 的位置，如果 size 是 8 的倍数，就按照 8 字节对齐，4 的倍数就按照 4 字节对齐，2 的倍数就按照 2 字节对齐。

```go
// alignUp rounds n up to a multiple of a. a must be a power of 2.
func alignUp(n, a uintptr) uintptr {
    // &^ bit clear
    return (n + a - 1) &^ (a - 1)
}
```

对于 Allocator 来说，最重要的特性就是**性能**，`mallocgc` 中会通过位图来提升分配器的性能。


需要注意的一点是，我们虽然可以通过 allocCache 加速在 `mspan` 查找空闲内存块的速度，但是**一个 allocCache**最多只能表示 64 个内存块的使用情况，像前面的 tiny  对象，一个 page （8K） 中有 512 个 16 Byte 的对象，这时该怎么办呢？

我们应该去研究具体的查找过程，即 `nextFreeFast nextFree nextFreeIndex`:

- nextFreeFast (fastPath)

  ```go
  // fastPath 
  // 假设 allocCache 为 0001 1111 1....1 （共 64 位） 这是一个已经分配了三次的 allocCache
  func nextFreeFast(s *mspan) gclinkptr {
      // 计算当前 allocCache 第一个未被使用的内存块位置
      // 0 使用，1 未使用
      theBit := sys.Ctz64(s.allocCache)
      // 64 的原因上边有提到，uint64 只能表示 64 个内存块
      // 如果 theBit == 64 说明该位图对应的所有内存块都被使用了
      // 如果 theBit < 64 说明还有内存块未被使用
      if theBit < 64 {
          // result 代表的是空闲内存块的索引
          result := s.freeindex + uintptr(theBit)
          if result < s.nelems {
              // 下一次 alloc 开始查找的位置
              freeidx := result + 1
              // 解释请见下面内容
              if freeidx%64 == 0 && freeidx != s.nelems {
                  return 0
              }
              s.allocCache >>= uint(theBit + 1)
              s.freeindex = freeidx
              s.allocCount++
              return gclinkptr(result*s.elemsize + s.base())
          }
      }
      return 0
  }
  ```

  以 spancalss=5 的 mspan 举例：

  ![](https://s2.loli.net/2022/07/24/jLlNhRDOP6us3Yd.png)

  还是挺清晰的吧，fastPath 就是检查当前这 64 个内存块有没有空闲的，没找到就走 slowPath，找到了则更新allocCache 对应的 bit，将 freeIndex 指向下一个位置，方便下一次分配的查找，更新当前 mspan 已经分配的次数。

  

  `freeidx%64 == 0 && freeidx != s.nelems` 的意思是：当 freeIndex 指向 allocCache 之外且不是最后一个对象时，fastPath 也是走不通的。 这里主要就是理解 freeIndex 的起始编号为 0，表示这 64 个比特位只需要 0-63，如果对 64 取模为 0 说明已经超过当前 allocCache 的管辖范围了，且当前 allocCache 对应的内存块都被用光了。

  

  那我们就需要更新 allocCache 中的内容，使其指向下一个 64 位的 cache bit。以 tinyAllocator 为例，每次分配  1 字节，我们需要分配 1024 次才能达到这个阈值（前提是这个 mspan是空的）。

  ```go
  *runtime.mspan {
      startAddr: 824633810944,
      npages: 1,
      manualFreeList: 0,
      freeindex: 63,
      nelems: 512,
      allocCache: 1,
      allocCount: 63,
      spanclass: tinySpanClass (5),
      elemsize: 16,
      limit: 824633819136}
  ```

  如上 mspan，我们已经使用它分配了 63 个对象（0-62），本次分配的空闲块索引为 63，下一次内存分配的索引为 64，即 `freeidx%64 == 0 && freeidx != s.nelems` 条件成立，只能走 slowPath 了。

- nextFree (slowPath)

  nextFree 中处理 allocCache 的主要逻辑在 `s.nextFreeIndex` 中

  ```go
  func (c *mcache) nextFree(spc spanClass) (v gclinkptr, s *mspan, shouldhelpgc bool) {
      s = c.alloc[spc]
      shouldhelpgc = false
      freeIndex := s.nextFreeIndex()
      if freeIndex == s.nelems {
          // The span is full.
          if uintptr(s.allocCount) != s.nelems {
              println("runtime: s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
              throw("s.allocCount != s.nelems && freeIndex == s.nelems")
          }
          c.refill(spc)
          shouldhelpgc = true
          s = c.alloc[spc]
  
          freeIndex = s.nextFreeIndex()
      }
  
      if freeIndex >= s.nelems {
          throw("freeIndex is not valid")
      }
  
      v = gclinkptr(freeIndex*s.elemsize + s.base())
      s.allocCount++
      if uintptr(s.allocCount) > s.nelems {
          println("s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
          throw("s.allocCount > s.nelems")
      }
      return
  }
  ```

- nextFreeIndex (find freeindex)

  ```go
  func (s *mspan) nextFreeIndex() uintptr {
      // 当前空闲块的位置
      sfreeindex := s.freeindex
      snelems := s.nelems
      // 如果是该 mspan 中最后一个内存块，则直接返回
      if sfreeindex == snelems {
          return sfreeindex
      }
      if sfreeindex > snelems {				
          throw("s.freeindex > s.nelems")
      }
  
      aCache := s.allocCache
  
      bitIndex := sys.Ctz64(aCache)
      for bitIndex == 64 {
          // Move index to start of next cached bits.
          sfreeindex = (sfreeindex + 64) &^ (64 - 1)
          if sfreeindex >= snelems {
              s.freeindex = snelems
              return snelems
          }
          whichByte := sfreeindex / 8
          // Refill s.allocCache with the next 64 alloc bits.
          s.refillAllocCache(whichByte)
          aCache = s.allocCache
          bitIndex = sys.Ctz64(aCache)
          // nothing available in cached bits
          // grab the next 8 bytes and try again.
      }
      result := sfreeindex + uintptr(bitIndex)
      if result >= snelems {
          s.freeindex = snelems
          return snelems
      }
  
      s.allocCache >>= uint(bitIndex + 1)
      sfreeindex = result + 1
  
      if sfreeindex%64 == 0 && sfreeindex != snelems {
          // We just incremented s.freeindex so it isn't 0.
          // As each 1 in s.allocCache was encountered and used for allocation
          // it was shifted away. At this point s.allocCache contains all 0s.
          // Refill s.allocCache so that it corresponds
          // to the bits at s.allocBits starting at s.freeindex.
          whichByte := sfreeindex / 8
          s.refillAllocCache(whichByte)
      }
      s.freeindex = sfreeindex
      return result
  }
  ```



查看上述源码得知，负责重新分配 allocCache 的是 `refilllAllocCache()`

```go
func (s *mspan) refillAllocCache(whichByte uintptr) {
    bytes := (*[8]uint8)(unsafe.Pointer(s.allocBits.bytep(whichByte)))
    aCache := uint64(0)
    aCache |= uint64(bytes[0])
    aCache |= uint64(bytes[1]) << (1 * 8)
    aCache |= uint64(bytes[2]) << (2 * 8)
    aCache |= uint64(bytes[3]) << (3 * 8)
    aCache |= uint64(bytes[4]) << (4 * 8)
    aCache |= uint64(bytes[5]) << (5 * 8)
    aCache |= uint64(bytes[6]) << (6 * 8)
    aCache |= uint64(bytes[7]) << (7 * 8)
    s.allocCache = ^aCache
}
```

从 allocBits 中获取对应的内存块使用信息，最后以补码的方式保存到 allocCache 中，方便 Ctz 函数的使用。



然后这里我观察了一下 allocBits 它是一个 `*gcBits` 类型，底层实际上是一个 uint8 的别名，这里最初也挺让我好奇的，一个 uint8 类型就 8 个 bit 是怎么标记 mspan 中那么多对象的。

然后又一头扎进源码中，发现这是一个指向 allocBits 的指针，实际 allocBits 大小是根据 nelems 计算得出的：

```go
// allocBit 的初始化函数
func newMarkBits(nelems uintptr) *gcBits {
    // 以 64 为步长，计算表达 nelemes 个对象需要多少块
    blocksNeeded := uintptr((nelems + 63) / 64)
    bytesNeeded := blocksNeeded * 8

    // Try directly allocating from the current head arena.
    head := (*gcBitsArena)(atomic.Loadp(unsafe.Pointer(&gcBitsArenas.next)))
    if p := head.tryAlloc(bytesNeeded); p != nil {
        return p
    }

    // There's not enough room in the head arena. We may need to
    // allocate a new arena.
    lock(&gcBitsArenas.lock)
    // Try the head arena again, since it may have changed. Now
    // that we hold the lock, the list head can't change, but its
    // free position still can.
    if p := gcBitsArenas.next.tryAlloc(bytesNeeded); p != nil {
        unlock(&gcBitsArenas.lock)
        return p
    }

    // Allocate a new arena. This may temporarily drop the lock.
    fresh := newArenaMayUnlock()
    // If newArenaMayUnlock dropped the lock, another thread may
    // have put a fresh arena on the "next" list. Try allocating
    // from next again.
    if p := gcBitsArenas.next.tryAlloc(bytesNeeded); p != nil {
        // Put fresh back on the free list.
        // TODO: Mark it "already zeroed"
        fresh.next = gcBitsArenas.free
        gcBitsArenas.free = fresh
        unlock(&gcBitsArenas.lock)
        return p
    }

    // Allocate from the fresh arena. We haven't linked it in yet, so
    // this cannot race and is guaranteed to succeed.
    p := fresh.tryAlloc(bytesNeeded)
    if p == nil {
        throw("markBits overflow")
    }

    // Add the fresh arena to the "next" list.
    fresh.next = gcBitsArenas.next
    atomic.StorepNoWB(unsafe.Pointer(&gcBitsArenas.next), unsafe.Pointer(fresh))

    unlock(&gcBitsArenas.lock)
    return p
}
```



> `(nelems + 63) / 64` 涉及到一个向上取整的通用算法，感兴趣可以自行  Google



`gcBitsArenas` 是用来管理这些 Bitmap 的数据结构：

```go
//go:notinheap
type gcBitsArena struct {
    free uintptr // free is the index into bits of the next free byte; read/write atomically
    next *gcBitsArena
    bits [gcBitsChunkBytes - gcBitsHeaderBytes]gcBits
}

var gcBitsArenas struct {
    lock     mutex
    free     *gcBitsArena
    next     *gcBitsArena // Read atomically. Write atomically under lock.
    current  *gcBitsArena
    previous *gcBitsArena
}
```

allocBit 的初始化过程还是比较简单的，这里不再赘述，我们继续回到 `allocCache`  的替换上，即 `refillAllocCache` 函数。理解这个函数最关键的一点在于如何理解他的参数 whichByte。

```go
// 当 sfreeindex % 64 == 0 时，才会执行这个计算
// 所以 whichByte 肯定总是 8 的倍数
// 用来定位 freeindex 在 allocBits 哪个位置
// 相对于 allocBit 开始位置的偏移量，最初偏移量为 0 然后依次偏移 8 的整数倍，8， 16，24..
// 便宜的是字节数，如果换算成 bit 是 8*8，16*8， 24*8
whichByte := sfreeindex / 8
```

然后将这个偏移量加到 `s.allocBits.bytep(whichByte)` 就获取到了下一个 8字节的起始地址。再依次获取这8字节中的每个字节表示的内存块的占用信息，以补码的形式保存在 allocCache 中即可。



至此，`tiny allocator` 涉及到的内容就结束了。`small allocator` 查找内存块的过程同 tiny allocator 类似，这里就不再继续展开。



额外提一点，你应该能够发现当我们通过 `fastPaht` 或 `nextFree` 找到 tiny 内存块的起始地址后，将这个指针转换成了 `*[2]uint64`，然后执行了两个赋值操作：

```go
(*[2]uint64)(x)[0] = 0
(*[2]uint64)(x)[1] = 0
```

这是在干什么？内存清零。tiny allocator 中，每个内存块都是 16 字节，两个 uint64 真好对应 16字节，然后将其中存储的内容置为0。







拓展：

德布鲁因序列 （Ctz 的实现原理）

http://supertech.csail.mit.edu/papers/debruijn.pdf

https://halfrost.com/go_s2_de_bruijn/#toc-0

