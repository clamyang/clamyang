---
title: Go 大对象分配探索
date: 2022-07-31
comments: true
---

在上一篇文章中，对 Go 语言中小对象的内存分配过程有了一个清晰的认识，同时为了更好的理解，我们忽略了大对象的内存分配过程。

我觉得，大对象的分配过程并不难理解，难的点在于你需要了解一些背景，理论基础。在这些资料的辅助下，才能更好的理解整个大对象的分配过程。我推荐你先读一下优化 [[page allocator](https://go.googlesource.com/proposal/+/master/design/35112-scaling-the-page-allocator.md)] 的 Proposal。

<!--more-->

读完之后就知道总说的 radix tree 到底是个什么形态，树中的节点代表什么含义，以及为什么要分为 5 层等。

本文使用的 Golang 版本为：`go1.16`

> 文中忽略了垃圾回收的相关内容，只针对大对象的分配进行了分析。

## Page Allocator

大对象（>32 KB）的分配直接走的是 page allocator

```go
type p struct {
	id          int32
	status      uint32 // one of pidle/prunning/...
	mcache      *mcache
    
	pcache      pageCache // 

    // ...
}
```



这里需要声明的一点是**大对象的分配也是有缓存的**，当大对象在 32 KB - 512(64*8)KB 时，会先尝试从当前的 P 进行分配，只有当对象 > 512 KB 时，才直接使用堆上的 page allocator。



当 cache 表示的内存大小被用光了，即 `cache == 0` ，这里需要将 cache 重填。

```go
func (p *pageAlloc) allocToCache() pageCache {}
```



大于 512 KB 时，主要分配逻辑如下：

![](https://s2.loli.net/2022/08/08/Jfq1k5TAtYQCPwo.png)

省略了在 summary 中具体的查找过程，其实把那个 proposal 读上个三四遍自然就明白整个查找过程是什么样了。



疑问：

- 为什么大对象的分配中会有很多地方都判断了 `npages == 1` ？

  提出这个问题的原因在于，大对象都是 > 32KB 的也就是说至少占用了 4 个 page，所以这个条件肯定总是假。但是，除了大对象的分配要走这里，**small对象进行堆区增长的时候，这个条件就为真了。**
  
- 指针地址为什么通过 `levelShift` 偏移，就能计算出这个指针在 summary[level] 的索引位置

  其实这个移位操作等同于加法，但是移位操作的效率更好，我们拿到一个指针地址后，对他进行移位就能计算出他在当前level的位置。这个操作就是，拿到一个地址，当前层按照每块 16GiB，那么使用这个地址除块大小即可。

  主要还是理解移位运算与除法运算之间的关系。

  4 / 2 就是将 0100 中的 1 向右移动一位。

