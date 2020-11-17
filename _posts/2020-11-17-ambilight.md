---
layout: post
title: "流光"
description: "浅析 ART 堆实现 - 内存管理 #1"
cover_url: https://i.loli.net/2020/11/18/uFWedkcQl7A92Mt.png
cover_meta: illustration by [Nengoro](https://www.pixiv.net/artworks/82213401)
tags: 
  - Android
---

在 Android Runtime 中，Java 层的堆内存主要通过 Space 相关类进行管理。

Space 根据字面含义，在 Android Runtime 中代表存放 Java 对象的内存空间，但从实现讲，理解为管理 Java 对象的容器更为贴切。每一个 Space 对应一块连续或多块不连续的内存空间，内部可保存多个 Java 对象。

## 光源

> C++ 在语言层面不存在接口的概念，面向对象设计中的接口采用抽象类实现
{:.side-note}

Space 相关类中有两个基础的类：`Space` 与 `AllocSpace`。其中 `AllocSpace` 则是一个接口，表示该内存空间可执行分配操作。

通过继承这两个关键类所构造出的 Space 相关类体系如下：

<script src="{{ site.assets }}/assets/dist/mermaid.min.js"></script>
<div class="mermaid">
classDiagram
    Space <|-- ContinuousSpace
    Space <|-- DiscontinuousSpace
    ContinuousSpace <|-- MemMapSpace
    DiscontinuousSpace <|-- LargeObjectSpace
    MemMapSpace <|-- ImageSpace
    MemMapSpace <|-- ContinuousMemMapAllocSpace
    LargeObjectSpace <|-- LargeObjectMapSpace
    LargeObjectSpace <|-- FreeListSpace
    ContinuousMemMapAllocSpace <|-- BumpPointerSpace
    ContinuousMemMapAllocSpace <|-- ZygoteSpace
    ContinuousMemMapAllocSpace <|-- RegionSpace
    ContinuousMemMapAllocSpace <|-- MallocSpace
    MallocSpace <|-- DlMallocSpace
    MallocSpace <|-- ResAllocSpace
    class LargeObjectSpace {
        allocatable(self)
    }
    class ImageSpace {
    }
    class ContinuousMemMapAllocSpace {
        allocatable(self)
    }
    class LargeObjectMapSpace {
        allocatable(parent)
    }
    class FreeListSpace {
        allocatable(parent)
    }
    class BumpPointerSpace {
        allocatable(parent)
    }
    class ZygoteSpace {
        allocatable(parent)
    }
    class RegionSpace {
        allocatable(parent)
    }
    class MallocSpace {
        allocatable(parent)
    }
    class DlMallocSpace {
        allocatable(parent)
    }
    class ResAllocSpace {
        allocatable(parent)
    }
</div>

> allocatable 表明该类实现了 `AllocSpace`，self 与 parent 用于标注 `AllocSpace` 由自身继承还是由超类继承。

除去一些含有具体实现逻辑的实现类，在此简单介绍部分类的作用。

顾名思义，`ContinuousSpace` 表明该 Space 对应的是一块连续的内存空间，而 `DiscontinuousSpace` 对应的则是多块不连续内存空间的组合。

`MemMapSpace` 是通过内存映射提供内存空间的 Space，实现的关键为对内存映射进行封装的 `MemMap` 辅助工具类。

`LargeObjectSpace` 用于管理大小超过阈值的大型对象，阈值的默认值为 3 个内存页大小，即 12 KB。

`ContinuousMemMapAllocSpace` 通过继承的方式整合了 `MemMapSpace` 与 `AllocSpace`，是连续内存空间且可分配 Space 类型的超类，用于在不同子类之间实现数据的拷贝。

`ImageSpace` 用于加载 `.art` 文件，这些文件中包含了 Android Framework 中通用的 Java 对象，由多个进程共享以提高性能。`ImageSpace` 在创建完成后其中的对象便不再发生变动，其内存也不会被释放，因此没有实现 `AllocSpace`。

`ZygoteSpace` 包含 Zygote 进程启动中所创建的对象，这些对象在进程间共享，与 `ImageSpace` 同理，其内存也不会被释放。

## 内存布局

在学习这些 Space 类型对对象的管理方式之前，首先要了解 Space 在 Android Runtime 堆内存中的作用。

Android Runtime 堆内存的 Space 主要组成包括 Image Space，Zygote Space，Allocation Space 与 Large Object Space。Image Space 与 Zygote Space 进程共享，内部对象由虚拟机提供，目的是为了减少对象冗余，Allocation Space 与 Large Object Space 则为进程相互独立，分别用于管理普通对象与大型对象。

在四者中，Image Space、Zygote Space、Large Object Space 分别与源代码中三个 Space 类型（包括子类）一一对应。

但 Allocation Space 在源代码中不存在直接对应类型，而是根据虚拟机采用不同的垃圾回收算法，在 `BumpPointerSpace`、`RegionSpace`、`DlMallocSpace`、`ResMallocSpace` 中进行选择。

``` c++
Heap::Heap(...) : ... {

  ...

  // Create other spaces based on whether or not we have a moving GC.
  if (foreground_collector_type_ == kCollectorTypeCC) {
    CHECK(separate_non_moving_space);
    // Reserve twice the capacity, to allow evacuating every region for explicit GCs.
    MemMap region_space_mem_map =
        space::RegionSpace::CreateMemMap(kRegionSpaceName, capacity_ * 2, request_begin);
    CHECK(region_space_mem_map.IsValid()) << "No region space mem map";
    region_space_ = space::RegionSpace::Create(
        kRegionSpaceName, std::move(region_space_mem_map), use_generational_cc_);
    AddSpace(region_space_);
  } else if (IsMovingGc(foreground_collector_type_)) {
    // Create bump pointer spaces.
    // We only to create the bump pointer if the foreground collector is a compacting GC.
    // TODO: Place bump-pointer spaces somewhere to minimize size of card table.
    bump_pointer_space_ = space::BumpPointerSpace::CreateFromMemMap("Bump pointer space 1",
                                                                    std::move(main_mem_map_1));
    CHECK(bump_pointer_space_ != nullptr) << "Failed to create bump pointer space";
    AddSpace(bump_pointer_space_);
    temp_space_ = space::BumpPointerSpace::CreateFromMemMap("Bump pointer space 2",
                                                            std::move(main_mem_map_2));
    CHECK(temp_space_ != nullptr) << "Failed to create bump pointer space";
    AddSpace(temp_space_);
    CHECK(separate_non_moving_space);
  } else {
    CreateMainMallocSpace(std::move(main_mem_map_1), initial_size, growth_limit_, capacity_);
    CHECK(main_space_ != nullptr);
    AddSpace(main_space_);
    if (!separate_non_moving_space) {
      non_moving_space_ = main_space_;
      CHECK(!non_moving_space_->CanMoveObjects());
    }
    if (main_mem_map_2.IsValid()) {
      const char* name = kUseRosAlloc ? kRosAllocSpaceName[1] : kDlMallocSpaceName[1];
      main_space_backup_.reset(CreateMallocSpaceFromMemMap(std::move(main_mem_map_2),
                                                           initial_size,
                                                           growth_limit_,
                                                           capacity_,
                                                           name,
                                                           /* can_move_objects= */ true));
      CHECK(main_space_backup_.get() != nullptr);
      // Add the space so its accounted for in the heap_begin and heap_end.
      AddSpace(main_space_backup_.get());
    }
  }

  ...

}
```

## 分配接口

首先看 `AllocSpace` 接口的定义，在此主要关注与这篇文字有关的函数接口。

``` c++
// AllocSpace interface.
class AllocSpace {
 public:
  // Number of bytes currently allocated.
  virtual uint64_t GetBytesAllocated() = 0;
  // Number of objects currently allocated.
  virtual uint64_t GetObjectsAllocated() = 0;

  // Allocate num_bytes without allowing growth. If the allocation
  // succeeds, the output parameter bytes_allocated will be set to the
  // actually allocated bytes which is >= num_bytes.
  // Alloc can be called from multiple threads at the same time and must be thread-safe.
  //
  // bytes_tl_bulk_allocated - bytes allocated in bulk ahead of time for a thread local allocation,
  // if applicable. It is
  // 1) equal to bytes_allocated if it's not a thread local allocation,
  // 2) greater than bytes_allocated if it's a thread local
  //    allocation that required a new buffer, or
  // 3) zero if it's a thread local allocation in an existing
  //    buffer.
  // This is what is to be added to Heap::num_bytes_allocated_.
  virtual mirror::Object* Alloc(Thread* self, size_t num_bytes, size_t* bytes_allocated,
                                size_t* usable_size, size_t* bytes_tl_bulk_allocated) = 0;

  ...

  // Returns how many bytes were freed.
  virtual size_t Free(Thread* self, mirror::Object* ptr) = 0;

  ...

};
```

`GetBytesAllocated()` 与 `GetObjectsAllocated()` 分别返回当前 Space 以分配的空间大小与 Java 对象数量。

`Alloc()` 函数用于在 Space 中分配空间作为 Java 对象，其在 Android Runtime 中对应类型为 `mirror::Object`，分配成功时返回该内存的首地址，即 `mirror::Object` 指针。

> 传入参数中，`self` 为函数调用线程，`num_bytes` 是所需分配空间的大小。
>
> 因为内存分配算法的原因，在部分情况下会分配大于 `num_bytes` 的内存空间，而 `bytes_allocated` 与 `usable_size` 作为返回参数，用于返回实际分配空间大小和其中可用的大小。
>
> `bytes_tl_bulk_allocated` 与 TLAB 相关。

`Free()` 函数则用于在 Space 中释放指定的 Java 对象。

## BumpPointerSpace

顾名思义，`BumpPointerSpace` 采用的内存分配方法极其简单：指针碰撞，即本次分配的起始地址为上一次分配的结尾地址。

> 指针碰撞（Bump Pointer）：将确定大小的内存通过一根指针分隔为两个部分，已使用空间位于指针一边，空闲空间位于指针另一边，分配内存仅需将指针向空闲空间方向挪动一段与对象大小相等的距离。

### 创建

先看一看用于创建 `BumpPointerSpace` 的静态函数，`BumpPointerSpace.Create()`。

``` c++
BumpPointerSpace* BumpPointerSpace::Create(const std::string& name, size_t capacity) {
  capacity = RoundUp(capacity, kPageSize);
  std::string error_msg;
  MemMap mem_map = MemMap::MapAnonymous(name.c_str(),
                                        capacity,
                                        PROT_READ | PROT_WRITE,
                                        /*low_4gb=*/ true,
                                        &error_msg);
  if (!mem_map.IsValid()) {
    LOG(ERROR) << "Failed to allocate pages for alloc space (" << name << ") of size "
        << PrettySize(capacity) << " with message " << error_msg;
    return nullptr;
  }
  return new BumpPointerSpace(name, std::move(mem_map));
}
```

`BumpPointerSpace` 的构造函数也很简单，仅对成员变量进行初始化操作。

``` c++
BumpPointerSpace::BumpPointerSpace(const std::string& name, MemMap&& mem_map)
    : ContinuousMemMapAllocSpace(name,
                                 std::move(mem_map),
                                 mem_map.Begin(),
                                 mem_map.Begin(),
                                 mem_map.End(),
                                 kGcRetentionPolicyAlwaysCollect),
      growth_end_(mem_map_.End()),
      objects_allocated_(0), bytes_allocated_(0),
      block_lock_("Block lock", kBumpPointerSpaceBlockLock),
      main_block_size_(0),
      num_blocks_(0) {
}
```

通过 `RoundUp()` 函数对 Space 内存空间大小 `capacity` 按内存页大小对齐，并映射出对应大小的内存空间，在构造函数中初始化为对应的成员变量。

### 分配

> 内联函数位于以 `-inl.h` 为结尾的头文件中，而非以 `.cc` 结尾的源文件。
{:.side-note}

直接看 `Alloc()`。

``` c++
inline mirror::Object* BumpPointerSpace::Alloc(Thread*, size_t num_bytes, size_t* bytes_allocated,
                                               size_t* usable_size,
                                               size_t* bytes_tl_bulk_allocated) {
  num_bytes = RoundUp(num_bytes, kAlignment);
  mirror::Object* ret = AllocNonvirtual(num_bytes);
  if (LIKELY(ret != nullptr)) {
    *bytes_allocated = num_bytes;
    if (usable_size != nullptr) {
      *usable_size = num_bytes;
    }
    *bytes_tl_bulk_allocated = num_bytes;
  }
  return ret;
}
```

可见实现位于 `AllocNonvirtual()` 中。

``` c++
inline mirror::Object* BumpPointerSpace::AllocNonvirtual(size_t num_bytes) {
  mirror::Object* ret = AllocNonvirtualWithoutAccounting(num_bytes);
  if (ret != nullptr) {
    objects_allocated_.fetch_add(1, std::memory_order_relaxed);
    bytes_allocated_.fetch_add(num_bytes, std::memory_order_relaxed);
  }
  return ret;
}
```

实际内存分配位于 `AllocNonvirtualWithoutAccounting()`，`AllocNonvirtual()` 在此基础上封装了对 `objects_allocated_` 与 `bytes_allocated_` 两个计数器进行更新的操作。

``` c++
inline mirror::Object* BumpPointerSpace::AllocNonvirtualWithoutAccounting(size_t num_bytes) {
  DCHECK_ALIGNED(num_bytes, kAlignment);
  uint8_t* old_end;
  uint8_t* new_end;
  do {
    old_end = end_.load(std::memory_order_relaxed);
    new_end = old_end + num_bytes;
    // If there is no more room in the region, we are out of memory.
    if (UNLIKELY(new_end > growth_end_)) {
      return nullptr;
    }
  } while (!end_.CompareAndSetWeakSequentiallyConsistent(old_end, new_end));
  return reinterpret_cast<mirror::Object*>(old_end);
}
```

`end_` 是 `ContinuousSpace` 的成员变量，记录了上次内存分配的结束位置。可见 `AllocNonvirtualWithoutAccounting` 通过 CAS 操作将分配结束位置的指针后移了 `num_bytes`，并返回分配前结束为止的地址，以此实现对象的创建。

> Thread Local Allocation Buffer，简称 TLAB，线程本地分配缓存区，是线程的专有内存资源，不同线程间相互独立。
{:.side-note}

除了 `Alloc()`，`BumpPointerSpace` 还提供了在用于 TLAB 时在线程本地空间分配的一些函数。

`BumpPointerSpace` 提供了 `AllocNewTlab()` 函数来让进程创建自己的 TLAB，原理是采用指针碰撞的算法分配一块称为 Block 的内存空间，此时每个 Block 可以将其视为一个独立的容器，其对象管理方式与 `BumpPointerSpace` 完全相同。

`AllocNewTlab()` 的实现如下。

``` c++
bool BumpPointerSpace::AllocNewTlab(Thread* self, size_t bytes) {
  MutexLock mu(Thread::Current(), block_lock_);
  RevokeThreadLocalBuffersLocked(self);
  uint8_t* start = AllocBlock(bytes);
  if (start == nullptr) {
    return false;
  }
  self->SetTlab(start, start + bytes, start + bytes);
  return true;
}
```

在通过 `RevokeThreadLocalBuffersLocked()` 释放线程 `self` 原有的 TLAB 后，`AllocNewTlab()` 调用了 `AllocBlock()` 分配指定大小的内存空间，并通过 `Thread.SetTlab()` 设置为 `self` 的新 TLAB。

``` c++
// Returns the start of the storage.
uint8_t* BumpPointerSpace::AllocBlock(size_t bytes) {
  bytes = RoundUp(bytes, kAlignment);
  if (!num_blocks_) {
    UpdateMainBlock();
  }
  uint8_t* storage = reinterpret_cast<uint8_t*>(
      AllocNonvirtualWithoutAccounting(bytes + sizeof(BlockHeader)));
  if (LIKELY(storage != nullptr)) {
    BlockHeader* header = reinterpret_cast<BlockHeader*>(storage);
    header->size_ = bytes;  // Write out the block header.
    storage += sizeof(BlockHeader);
    ++num_blocks_;
  }
  return storage;
}
```

`AllocBlock()` 同样采用了 `AllocNonvirtualWithoutAccounting()` 函数来分配内存空间，但区别在于，`AllocBlock()` 额外分配了 `BlockHeader` 大小的空间，位于 Block 的头部，用于记录 Block 的大小。

``` c++
class BumpPointerSpace final : public ContinuousMemMapAllocSpace {

  ...

  private:
    struct BlockHeader {
        size_t size_;  // Size of the block in bytes, does not include the header.
        size_t unused_;  // Ensures alignment of kAlignment.
    };

  ...

}
```

而在 TLAB 中分配对象的函数位于 `Thread` 类中，其内存分配算法与 `BumpPointerSpace` 一致，均为指针碰撞。

``` c++
inline mirror::Object* Thread::AllocTlab(size_t bytes) {
  DCHECK_GE(TlabSize(), bytes);
  ++tlsPtr_.thread_local_objects;
  mirror::Object* ret = reinterpret_cast<mirror::Object*>(tlsPtr_.thread_local_pos);
  tlsPtr_.thread_local_pos += bytes;
  return ret;
}
```

### 释放

由于 `BumpPointerSpace` 的内存分配算法过于简单，因此压根无法实现释放单一对象的功能。

``` c++
// NOPS unless we support free lists.
size_t Free(Thread*, mirror::Object*) override {
  return 0;
}
```

但 `BumpPointerSpace` 可一次性释放内部的所有对象，对应函数为 `Clear()`，实现方式为将部分成员变量进行复原，并通过 `madvise()` 对 Space 对应内存空间进行了清零操作。

``` c++
void BumpPointerSpace::Clear() {
  // Release the pages back to the operating system.
  if (!kMadviseZeroes) {
    memset(Begin(), 0, Limit() - Begin());
  }
  CHECK_NE(madvise(Begin(), Limit() - Begin(), MADV_DONTNEED), -1) << "madvise failed";
  // Reset the end of the space back to the beginning, we move the end forward as we allocate
  // objects.
  SetEnd(Begin());
  objects_allocated_.store(0, std::memory_order_relaxed);
  bytes_allocated_.store(0, std::memory_order_relaxed);
  growth_end_ = Limit();
  {
    MutexLock mu(Thread::Current(), block_lock_);
    num_blocks_ = 0;
    main_block_size_ = 0;
  }
}
```

### 遍历

`BumpPointerSpace` 提供 `Walk()` 函数用于遍历 Space 中的对象，不过在了解该函数的实现之前，必须先了解 Main Block 这一概念。

回看 `AllocBlock()` 的实现，在第一次分配 Block 的时候调用了一个 `UpdateMainBlock()` 的函数。

``` c++
// Returns the start of the storage.
uint8_t* BumpPointerSpace::AllocBlock(size_t bytes) {
  ...
  if (!num_blocks_) {
    UpdateMainBlock();
  }
  ...
}
```

该函数的逻辑非常简单，将成员变量 `main_block_size_` 设置为 `BumpPointerSpace` 当前分配结束地址与起始地址的差值，即 Space 当前已使用的空间大小。

Main Block 的定义，即 Space 起始位置至第一次分配的 Block 起始位置之前。当 `AllocBlock()` 第一次被调用时，此时 `BumpPointerSpace` 内不存在任何 Block，则 `BumpPointerSpace` 会将当前已使用的空间定义为 Main Block，之后在 Main Block 结束地址创建新的 Block。

也就是说，**Main Block 内部不包含任何 Block**。

`Walk()` 的实现可以拆分成三部分。

第一部分是根据成员变量将边界条件初始化为相应变量，`main_end` 设置为 Main Block 的结束位置。当 `AllocBlock()` 不曾被调用，即 `num_blocks_` 为 0 时，需要调用 `UpdateMainBlock()` 以定义 Main Block。

``` c++
template <typename Visitor>
inline void BumpPointerSpace::Walk(Visitor&& visitor) {
  uint8_t* pos = Begin();
  uint8_t* end = End();
  uint8_t* main_end = pos;
  // Internal indirection w/ NO_THREAD_SAFETY_ANALYSIS. Optimally, we'd like to have an annotation
  // like
  //   REQUIRES_AS(visitor.operator(mirror::Object*))
  // on Walk to expose the interprocedural nature of locks here without having to duplicate the
  // function.
  //
  // NO_THREAD_SAFETY_ANALYSIS is a workaround. The problem with the workaround of course is that
  // it doesn't complain at the callsite. However, that is strictly not worse than the
  // ObjectCallback version it replaces.
  auto no_thread_safety_analysis_visit = [&](mirror::Object* obj) NO_THREAD_SAFETY_ANALYSIS {
    visitor(obj);
  };

  {
    MutexLock mu(Thread::Current(), block_lock_);
    // If we have 0 blocks then we need to update the main header since we have bump pointer style
    // allocation into an unbounded region (actually bounded by Capacity()).
    if (num_blocks_ == 0) {
      UpdateMainBlock();
    }
    main_end = Begin() + main_block_size_;
    if (num_blocks_ == 0) {
      // We don't have any other blocks, this means someone else may be allocating into the main
      // block. In this case, we don't want to try and visit the other blocks after the main block
      // since these could actually be part of the main block.
      end = main_end;
    }
  }
  
  ...
}
```

在初始化变量完成后，遍历过程分为两个部分。实现的第二部分，也是遍历的第一部分，是对 Main Block 的遍历。

遍历过程中所获取到的 `mirror::Object` 指针不一定真正指向一个有效的对象，因此需要通过 `mirror::Object.GetClass()` 获取对象所属类并判断是否为空来确定对象是否有效，对于 `BumpPointerSpace` 的内存分配机制，如果发现无效对象，说明在此之后的内存空间无效，直接跳过遍历。

``` c++
template <typename Visitor>
inline void BumpPointerSpace::Walk(Visitor&& visitor) {
  
  ...

  // Walk all of the objects in the main block first.
  while (pos < main_end) {
    mirror::Object* obj = reinterpret_cast<mirror::Object*>(pos);
    // No read barrier because obj may not be a valid object.
    if (obj->GetClass<kDefaultVerifyFlags, kWithoutReadBarrier>() == nullptr) {
      // There is a race condition where a thread has just allocated an object but not set the
      // class. We can't know the size of this object, so we don't visit it and exit the function
      // since there is guaranteed to be not other blocks.
      return;
    } else {
      no_thread_safety_analysis_visit(obj);
      pos = reinterpret_cast<uint8_t*>(GetNextObject(obj));
    }
  }
  
  ...
}
```

实现的第三部分，也是遍历的第二部分，是对除 Main Block 之外的其它内存空间进行遍历。与 Main Block 遍历的区别在于，这一部分的遍历需要考虑到 `BlockHeader` 的存在。

``` c++
template <typename Visitor>
inline void BumpPointerSpace::Walk(Visitor&& visitor) {
  
  ...

  // Walk the other blocks (currently only TLABs).
  while (pos < end) {
    BlockHeader* header = reinterpret_cast<BlockHeader*>(pos);
    size_t block_size = header->size_;
    pos += sizeof(BlockHeader);  // Skip the header so that we know where the objects
    mirror::Object* obj = reinterpret_cast<mirror::Object*>(pos);
    const mirror::Object* end_obj = reinterpret_cast<const mirror::Object*>(pos + block_size);
    CHECK_LE(reinterpret_cast<const uint8_t*>(end_obj), End());
    // We don't know how many objects are allocated in the current block. When we hit a null class
    // assume its the end. TODO: Have a thread update the header when it flushes the block?
    // No read barrier because obj may not be a valid object.
    while (obj < end_obj && obj->GetClass<kDefaultVerifyFlags, kWithoutReadBarrier>() != nullptr) {
      no_thread_safety_analysis_visit(obj);
      obj = GetNextObject(obj);
    }
    pos += block_size;
  }
}
```

> 待证实的猜测：
>
> S1：邓凡平老师所著的《深入理解 Android - Java 虚拟机 ART》中提及，当 `BumpPointerSpace` 用于 TLAB 时，Main Block 是第一个线程的 TLAB，猜测该线程具有特殊性，且创建 TLAB 不是通过 `AllocNewTlab()` 实现，在第二个线程调用 `AllocNewTlab()` 前该 TLAB 中的内容便已固定。
>
> S2：基于 S1 与 `Walk()` 的遍历算法，当 `BumpPointerSpace` 用于 TLAB 时，Main Block 与各个 Block 之间不存在额外分配的对象，即不再作为线程共享的 Space 使用。

## RegionSpace

`RegionSpace` 的内存分配算法与指针碰撞同样经典的另一内存分配算法 —— 空闲列表较为相似，不过相比于真正使用空闲列表的 `FreeListSpace` 而言，`RegionSpace` 在连续空间上的分配算法要稍微简单一点。

> 空闲列表（Free List）：已分配的内存和空闲内存相互交错，虚拟机通过维护一个列表，记录可用的内存块信息，当分配操作发生时，从列表中找到一个足够大的内存块分配给对象实例，并更新列表上的记录。

`RegionSpace` 将内存空间划分为多个相同大小的区块（Region），每一个区块通过类 `Region` 的对象表示，在分配内存时，`RegionSpace` 将找到符合要求的 Region 用于分配内存。

### 创建

`RegionSpace` 同样具备用于创建的静态函数 `RegionSpace.Create()`。

``` c++
RegionSpace* RegionSpace::Create(
    const std::string& name, MemMap&& mem_map, bool use_generational_cc) {
  return new RegionSpace(name, std::move(mem_map), use_generational_cc);
}
```

> Android 8.0 对 `RegionSpace` 的创建方式进行了重构。
>
> Android 7.1 及以前，`RegionSpace` 提供了一个在函数内部创建设置 `MemMap` 对象的 `RegionSpace.Create()` 重载，但在此之后对 `MemMap` 的创建设置逻辑移动到 `RegionSpace.CreateMemMap()` 中，其返回的 `MemMap` 对象传入到本文所提的 `RegionSpace.Create()` 函数中。

相比 `BumpPointerSpace`，`RegionSpace` 的构造函数可就不那么简单了。其实现如下。

``` c++
RegionSpace::RegionSpace(const std::string& name, MemMap&& mem_map, bool use_generational_cc)
    : ContinuousMemMapAllocSpace(name,
                                 std::move(mem_map),
                                 mem_map.Begin(),
                                 mem_map.End(),
                                 mem_map.End(),
                                 kGcRetentionPolicyAlwaysCollect),
      region_lock_("Region lock", kRegionSpaceRegionLock),
      use_generational_cc_(use_generational_cc),
      time_(1U),
      num_regions_(mem_map_.Size() / kRegionSize),
      madvise_time_(0U),
      num_non_free_regions_(0U),
      num_evac_regions_(0U),
      max_peak_num_non_free_regions_(0U),
      non_free_region_index_limit_(0U),
      current_region_(&full_region_),
      evac_region_(nullptr),
      cyclic_alloc_region_index_(0U) {
  
  ...

  regions_.reset(new Region[num_regions_]);
  uint8_t* region_addr = mem_map_.Begin();
  for (size_t i = 0; i < num_regions_; ++i, region_addr += kRegionSize) {
    regions_[i].Init(i, region_addr, region_addr + kRegionSize);
  }
  mark_bitmap_ =
      accounting::ContinuousSpaceBitmap::Create("region space live bitmap", Begin(), Capacity());
  
  ...
}
```

`nums_regions_` 成员变量表示 `RegionSpace` 包含的 Region 数量，初始值为内存空间大小除以 单个 Region 大小（`kRegionSize`），`num_non_free_regions_` 则为已使用 Region 数量。

构造函数首先创建并初始化了 `Region` 数组 `regions_`，之后对标记对象用的位图 `mark_bitmap_` 进行创建。

除此之外还存在三个重要的成员变量需要说明。`full_region_` 是一个内部数据无用的 `Region` 对象，用于表示一个内存资源不足的 Region。`current_region_` 为用于分配的当前 Region，`evac_region_` 为用于垃圾回收时释放的当前 Region。

> 奇怪的是，自前文所提及的 `RegionSpace` 创建方式重构后，成员变量 `full_region_` 的默认值初始化过程似乎消失了。
>
> ~~我们仍未知道那天所看见的成员变量的初始化过程。~~

### 分配

``` c++
inline mirror::Object* RegionSpace::Alloc(Thread* self ATTRIBUTE_UNUSED,
                                          size_t num_bytes,
                                          /* out */ size_t* bytes_allocated,
                                          /* out */ size_t* usable_size,
                                          /* out */ size_t* bytes_tl_bulk_allocated) {
  num_bytes = RoundUp(num_bytes, kAlignment);
  return AllocNonvirtual<false>(num_bytes, bytes_allocated, usable_size,
                                bytes_tl_bulk_allocated);
}
```

和 `BumpPointerSpace` 相似，在内存大小对齐后调用了实际用于分配的另一个函数。由于函数逻辑较长，在这里对其进行拆分。

注意调用 `AllocNonvirtual()` 时的模版参数为 `false`。

``` c++
template<bool kForEvac>
inline mirror::Object* RegionSpace::AllocNonvirtual(size_t num_bytes,
                                                    /* out */ size_t* bytes_allocated,
                                                    /* out */ size_t* usable_size,
                                                    /* out */ size_t* bytes_tl_bulk_allocated) {
  DCHECK_ALIGNED(num_bytes, kAlignment);
  mirror::Object* obj;
  if (LIKELY(num_bytes <= kRegionSize)) {
    // Non-large object.
    obj = (kForEvac ? evac_region_ : current_region_)->Alloc(num_bytes,
                                                             bytes_allocated,
                                                             usable_size,
                                                             bytes_tl_bulk_allocated);
    if (LIKELY(obj != nullptr)) {
      return obj;
    }
    MutexLock mu(Thread::Current(), region_lock_);
    // Retry with current region since another thread may have updated
    // current_region_ or evac_region_.  TODO: fix race.
    obj = (kForEvac ? evac_region_ : current_region_)->Alloc(num_bytes,
                                                             bytes_allocated,
                                                             usable_size,
                                                             bytes_tl_bulk_allocated);
    if (LIKELY(obj != nullptr)) {
      return obj;
    }
    
    ...

  } else {
    
    ...

  }
  return nullptr;
}
```

根据模版函数的值决定在 `current_region_` 或 `evac_region_` 分配内存，在此逻辑分支中选择了 `current_region_`。如果在指定的 `Region` 中分配成功，则直接返回该对象。

这里相同的代码出现了两次，区别在于第二次执行之前加了锁，目的是防止并发过程中出现其它线程已将 `current_region_` 更新为一个可用 Region 的情况。

当然，由于当前 `current_region_` 指向了 `full_region_`，因此分配必然返回 `nullptr`，因此进入第二部分的逻辑，寻找下一个可用的 Region。

``` c++
template<bool kForEvac>
inline mirror::Object* RegionSpace::AllocNonvirtual(size_t num_bytes,
                                                    /* out */ size_t* bytes_allocated,
                                                    /* out */ size_t* usable_size,
                                                    /* out */ size_t* bytes_tl_bulk_allocated) {
  DCHECK_ALIGNED(num_bytes, kAlignment);
  mirror::Object* obj;
  if (LIKELY(num_bytes <= kRegionSize)) {
    
    ...

    Region* r = AllocateRegion(kForEvac);
    if (LIKELY(r != nullptr)) {
      obj = r->Alloc(num_bytes, bytes_allocated, usable_size, bytes_tl_bulk_allocated);
      CHECK(obj != nullptr);
      // Do our allocation before setting the region, this makes sure no threads race ahead
      // and fill in the region before we allocate the object. b/63153464
      if (kForEvac) {
        evac_region_ = r;
      } else {
        current_region_ = r;
      }
      return obj;
    }
  } else {
    
    ...

  }
  return nullptr;
}
```

可见函数通过调用 `AllocateRegion()` 寻找下一个可用的 Region，然后 ~~梅开三度~~ 再次分配，并将 `current_region_` 或 `evac_region_` 指向更新后的 Region。

``` c++
RegionSpace::Region* RegionSpace::AllocateRegion(bool for_evac) {
  if (!for_evac && (num_non_free_regions_ + 1) * 2 > num_regions_) {
    return nullptr;
  }
  for (size_t i = 0; i < num_regions_; ++i) {
    // When using the cyclic region allocation strategy, try to
    // allocate a region starting from the last cyclic allocated
    // region marker. Otherwise, try to allocate a region starting
    // from the beginning of the region space.
    size_t region_index = kCyclicRegionAllocation
        ? ((cyclic_alloc_region_index_ + i) % num_regions_)
        : i;
    Region* r = &regions_[region_index];
    if (r->IsFree()) {
      r->Unfree(this, time_);
      if (use_generational_cc_) {
        ...
      }
      if (for_evac) {
        ...
      } else {
        r->SetNewlyAllocated();
        ++num_non_free_regions_;
      }
      ...
      return r;
    }
  }
  return nullptr;
}
```

由于 `RegionSpace` 对应的垃圾回收算法为复制回收，因此必须预留一半的内存空间，当已使用的 Region 数超过总数的一半时，将返回 `nullptr`，不再允许分配。

之后对 `regions_` 数组进行遍历，寻找可用的 Region，通过 `Unfree()` 进行标记后返回。

> `kCyclicRegionAllocation` 是循环区域分配策略的标志。
>
> 在循环区域分配策略中，搜索可用 Region 将从上次分配的结束位置开始，而非从头开始。这种策略可以减少 Region 重用，并有助于更早发现垃圾回收错误，但这种分配策略也会造成 Region 级别的内存碎片化，因此该策略仅在调试模式下被启用。

回到 `AllocNonvirtual()`，当对象大小大于单个 Region 的大小时，就必须采用多个 Region 容纳单个对象，此时调用 `AllocLarge()` 进行分配。

``` c++
template<bool kForEvac>
inline mirror::Object* RegionSpace::AllocNonvirtual(size_t num_bytes,
                                                    /* out */ size_t* bytes_allocated,
                                                    /* out */ size_t* usable_size,
                                                    /* out */ size_t* bytes_tl_bulk_allocated) {
  DCHECK_ALIGNED(num_bytes, kAlignment);
  mirror::Object* obj;
  if (LIKELY(num_bytes <= kRegionSize)) {
    
    ...

  } else {
    // Large object.
    obj = AllocLarge<kForEvac>(num_bytes, bytes_allocated, usable_size, bytes_tl_bulk_allocated);
    if (LIKELY(obj != nullptr)) {
      return obj;
    }
  }
  return nullptr;
}
```

接下来看看核心部分：`Region.Alloc()`，实现还是眼熟的那个 -- 碰撞指针。

``` c++
inline mirror::Object* RegionSpace::Region::Alloc(size_t num_bytes,
                                                  /* out */ size_t* bytes_allocated,
                                                  /* out */ size_t* usable_size,
                                                  /* out */ size_t* bytes_tl_bulk_allocated) {
  DCHECK(IsAllocated() && IsInToSpace());
  DCHECK_ALIGNED(num_bytes, kAlignment);
  uint8_t* old_top;
  uint8_t* new_top;
  do {
    old_top = top_.load(std::memory_order_relaxed);
    new_top = old_top + num_bytes;
    if (UNLIKELY(new_top > end_)) {
      return nullptr;
    }
  } while (!top_.CompareAndSetWeakRelaxed(old_top, new_top));
  objects_allocated_.fetch_add(1, std::memory_order_relaxed);
  DCHECK_LE(Top(), end_);
  DCHECK_LT(old_top, end_);
  DCHECK_LE(new_top, end_);
  *bytes_allocated = num_bytes;
  if (usable_size != nullptr) {
    *usable_size = num_bytes;
  }
  *bytes_tl_bulk_allocated = num_bytes;
  return reinterpret_cast<mirror::Object*>(old_top);
}
```

`AllocLarge()` 的实现也并不复杂，它搜索了多个空间上连续的 Region 用于存放该对象，**并直接将这些 Region 中第一个的首地址作为对象的首地址**。

`AllocLargeInRange()` 的实现是单纯的算法问题，采用了滑动窗口，这里就不再说明了。

``` c++
template<bool kForEvac>
inline mirror::Object* RegionSpace::AllocLarge(size_t num_bytes,
                                               /* out */ size_t* bytes_allocated,
                                               /* out */ size_t* usable_size,
                                               /* out */ size_t* bytes_tl_bulk_allocated) {
  
  ...

  size_t num_regs_in_large_region = RoundUp(num_bytes, kRegionSize) / kRegionSize;
  
  ...

  MutexLock mu(Thread::Current(), region_lock_);
  if (!kForEvac) {
    // Retain sufficient free regions for full evacuation.
    if ((num_non_free_regions_ + num_regs_in_large_region) * 2 > num_regions_) {
      return nullptr;
    }
  }

  mirror::Object* region = nullptr;
  // Find a large enough set of contiguous free regions.
  if (kCyclicRegionAllocation) {
    
    ...

  } else {
    // Try to find a range of free regions within [0, num_regions_).
    region = AllocLargeInRange<kForEvac>(0,
                                         num_regions_,
                                         num_regs_in_large_region,
                                         bytes_allocated,
                                         usable_size,
                                         bytes_tl_bulk_allocated);
  }
  if (kForEvac && region != nullptr) {
    TraceHeapSize();
  }
  return region;
}
```

与 `BumpPointerSpace` 相同，`RegionSpace` 也可以用于 TLAB，但在实现上更为简单。因为在 `RegionSpace` 中，内存空间已被划分为 Region，因此只需将一个未使用的 Region 设置为线程的 TLAB 即可。

``` c++
bool RegionSpace::AllocNewTlab(Thread* self,
                               const size_t tlab_size,
                               size_t* bytes_tl_bulk_allocated) {
  MutexLock mu(self, region_lock_);
  RevokeThreadLocalBuffersLocked(self, /*reuse=*/ gc::Heap::kUsePartialTlabs);
  Region* r = nullptr;
  uint8_t* pos = nullptr;
  *bytes_tl_bulk_allocated = tlab_size;
  // First attempt to get a partially used TLAB, if available.
  if (tlab_size < kRegionSize) {
    // Fetch the largest partial TLAB. The multimap is ordered in decreasing
    // size.
    auto largest_partial_tlab = partial_tlabs_.begin();
    if (largest_partial_tlab != partial_tlabs_.end() && largest_partial_tlab->first >= tlab_size) {
      r = largest_partial_tlab->second;
      pos = r->End() - largest_partial_tlab->first;
      partial_tlabs_.erase(largest_partial_tlab);
      DCHECK_GT(r->End(), pos);
      DCHECK_LE(r->Begin(), pos);
      DCHECK_GE(r->Top(), pos);
      *bytes_tl_bulk_allocated -= r->Top() - pos;
    }
  }
  if (r == nullptr) {
    // Fallback to allocating an entire region as TLAB.
    r = AllocateRegion(/*for_evac=*/ false);
  }
  if (r != nullptr) {
    uint8_t* start = pos != nullptr ? pos : r->Begin();
    DCHECK_ALIGNED(start, kObjectAlignment);
    r->is_a_tlab_ = true;
    r->thread_ = self;
    r->SetTop(r->End());
    self->SetTlab(start, start + tlab_size, r->End());
    return true;
  }
  return false;
}
```

### 释放

当看到 `Region` 采用指针碰撞分配内存时估计就想到了，`RegionSpace` 同样不支持释放单个对象。

``` c++
size_t Free(Thread*, mirror::Object*) override {
  UNIMPLEMENTED(FATAL);
  return 0;
}
```

`Clear()` 函数的逻辑也较为简单：对成员变量的重新初始化。

``` c++
void RegionSpace::Clear() {
  MutexLock mu(Thread::Current(), region_lock_);
  for (size_t i = 0; i < num_regions_; ++i) {
    Region* r = &regions_[i];
    if (!r->IsFree()) {
      --num_non_free_regions_;
    }
    r->Clear(/*zero_and_release_pages=*/true);
  }
  SetNonFreeRegionLimit(0);
  DCHECK_EQ(num_non_free_regions_, 0u);
  current_region_ = &full_region_;
  evac_region_ = &full_region_;
}
```

### 遍历

`RegionSpace` 的 `Walk()` 函数的实际实现为 `WalkInternal()`，模版参数为 `false`。

``` c++
template <typename Visitor>
inline void RegionSpace::Walk(Visitor&& visitor) {
  WalkInternal</* kToSpaceOnly= */ false>(visitor);
}
```

`WalkInternal()` 实现如下，通过 `Region.isLarge()` 判断是否用于大型对象分配，否则调用 `WalkNonLargeRegion()` 对 Region 内部进行遍历。

``` c++
template<bool kToSpaceOnly, typename Visitor>
inline void RegionSpace::WalkInternal(Visitor&& visitor) {
  // TODO: MutexLock on region_lock_ won't work due to lock order
  // issues (the classloader classes lock and the monitor lock). We
  // call this with threads suspended.
  Locks::mutator_lock_->AssertExclusiveHeld(Thread::Current());
  for (size_t i = 0; i < num_regions_; ++i) {
    Region* r = &regions_[i];
    if (r->IsFree() || (kToSpaceOnly && !r->IsInToSpace())) {
      continue;
    }
    if (r->IsLarge()) {
      // We may visit a large object with live_bytes = 0 here. However, it is
      // safe as it cannot contain dangling pointers because corresponding regions
      // (and regions corresponding to dead referents) cannot be allocated for new
      // allocations without first clearing regions' live_bytes and state.
      mirror::Object* obj = reinterpret_cast<mirror::Object*>(r->Begin());
      DCHECK(obj->GetClass() != nullptr);
      visitor(obj);
    } else if (r->IsLargeTail()) {
      // Do nothing.
    } else {
      WalkNonLargeRegion(visitor, r);
    }
  }
}
```

`WalkNonLargeRegion()` 的遍历方式与遍历 `BumpPointerSpace` 的 Main Block 逻辑相似。

``` c++
template<typename Visitor>
inline void RegionSpace::WalkNonLargeRegion(Visitor&& visitor, const Region* r) {
  DCHECK(!r->IsLarge() && !r->IsLargeTail());
  // For newly allocated and evacuated regions, live bytes will be -1.
  uint8_t* pos = r->Begin();
  uint8_t* top = r->Top();
  // We need the region space bitmap to iterate over a region's objects
  // if
  // - its live bytes count is invalid (i.e. -1); or
  // - its live bytes count is lower than the allocated bytes count.
  //
  // In both of the previous cases, we do not have the guarantee that
  // all allocated objects are "alive" (i.e. valid), so we depend on
  // the region space bitmap to identify which ones to visit.
  //
  // On the other hand, when all allocated bytes are known to be alive,
  // we know that they form a range of consecutive objects (modulo
  // object alignment constraints) that can be visited iteratively: we
  // can compute the next object's location by using the current
  // object's address and size (and object alignment constraints).
  const bool need_bitmap =
      r->LiveBytes() != static_cast<size_t>(-1) &&
      r->LiveBytes() != static_cast<size_t>(top - pos);
  if (need_bitmap) {
    GetLiveBitmap()->VisitMarkedRange(
        reinterpret_cast<uintptr_t>(pos),
        reinterpret_cast<uintptr_t>(top),
        visitor);
  } else {
    while (pos < top) {
      mirror::Object* obj = reinterpret_cast<mirror::Object*>(pos);
      if (obj->GetClass<kDefaultVerifyFlags, kWithoutReadBarrier>() != nullptr) {
        visitor(obj);
        pos = reinterpret_cast<uint8_t*>(GetNextObject(obj));
      } else {
        break;
      }
    }
  }
}
```