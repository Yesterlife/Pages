---
layout: post
title: "星屑"
description: "浅析 ART 堆实现 - 内存管理 #2"
cover_url: https://i.loli.net/2020/12/01/G1WQaI2m6JO4BVA.png
cover_meta: illustration by [Nengoro](https://www.pixiv.net/artworks/82213401)
tags: 
  - Develop
  - Android
---

## DlMallocSpace

`DlMallocSpace` 采用一种知名的开源内存分配算法 dlmalloc 进行内存分配，本文不对算法本身进行解析，因此对 `DlMallocSpace` 的说明相对简单。

### 创建

`DlMallocSpace` 通过静态函数 `Create()` 创建对象，实现上与其它 Space 区别不大，内部调用了 `CreateMemMap()` 创建匿名映射 `MemMap` 对象，然后调用 `CreateFromMemMap()` 进行构造。

``` c++
DlMallocSpace* DlMallocSpace::Create(const std::string& name,
                                     size_t initial_size,
                                     size_t growth_limit,
                                     size_t capacity,
                                     bool can_move_objects) {
  uint64_t start_time = 0;
  if (VLOG_IS_ON(heap) || VLOG_IS_ON(startup)) {
    start_time = NanoTime();
    LOG(INFO) << "DlMallocSpace::Create entering " << name
        << " initial_size=" << PrettySize(initial_size)
        << " growth_limit=" << PrettySize(growth_limit)
        << " capacity=" << PrettySize(capacity);
  }

  // Memory we promise to dlmalloc before it asks for morecore.
  // Note: making this value large means that large allocations are unlikely to succeed as dlmalloc
  // will ask for this memory from sys_alloc which will fail as the footprint (this value plus the
  // size of the large allocation) will be greater than the footprint limit.
  size_t starting_size = kPageSize;
  MemMap mem_map = CreateMemMap(name, starting_size, &initial_size, &growth_limit, &capacity);
  if (!mem_map.IsValid()) {
    LOG(ERROR) << "Failed to create mem map for alloc space (" << name << ") of size "
               << PrettySize(capacity);
    return nullptr;
  }
  DlMallocSpace* space = CreateFromMemMap(std::move(mem_map),
                                          name,
                                          starting_size,
                                          initial_size,
                                          growth_limit,
                                          capacity,
                                          can_move_objects);
  // We start out with only the initial size possibly containing objects.
  if (VLOG_IS_ON(heap) || VLOG_IS_ON(startup)) {
    LOG(INFO) << "DlMallocSpace::Create exiting (" << PrettyDuration(NanoTime() - start_time)
        << " ) " << *space;
  }
  return space;
}
```

`CreateFromMemMap()` 调用 dlmalloc 接口 `CreateMspace()` 以管理内存映射区域，最终调用构造函数。

``` c++
DlMallocSpace* DlMallocSpace::CreateFromMemMap(MemMap&& mem_map,
                                               const std::string& name,
                                               size_t starting_size,
                                               size_t initial_size,
                                               size_t growth_limit,
                                               size_t capacity,
                                               bool can_move_objects) {
  DCHECK(mem_map.IsValid());
  void* mspace = CreateMspace(mem_map.Begin(), starting_size, initial_size);
  if (mspace == nullptr) {
    LOG(ERROR) << "Failed to initialize mspace for alloc space (" << name << ")";
    return nullptr;
  }

  // Protect memory beyond the starting size. morecore will add r/w permissions when necessory
  uint8_t* end = mem_map.Begin() + starting_size;
  if (capacity - starting_size > 0) {
    CheckedCall(mprotect, name.c_str(), end, capacity - starting_size, PROT_NONE);
  }

  // Everything is set so record in immutable structure and leave
  uint8_t* begin = mem_map.Begin();
  if (Runtime::Current()->IsRunningOnMemoryTool()) {
    ...
  } else {
    return new DlMallocSpace(std::move(mem_map),
                             initial_size,
                             name,
                             mspace,
                             begin,
                             end,
                             begin + capacity,
                             growth_limit,
                             can_move_objects,
                             starting_size);
  }
}
```

构造函数也很简单，仅对成员变量进行了初始化操作。

``` c++
DlMallocSpace::DlMallocSpace(MemMap&& mem_map,
                             size_t initial_size,
                             const std::string& name,
                             void* mspace,
                             uint8_t* begin,
                             uint8_t* end,
                             uint8_t* limit,
                             size_t growth_limit,
                             bool can_move_objects,
                             size_t starting_size)
    : MallocSpace(name,
                  std::move(mem_map),
                  begin,
                  end,
                  limit,
                  growth_limit,
                  /* create_bitmaps= */ true,
                  can_move_objects,
                  starting_size, initial_size),
      mspace_(mspace) {
  CHECK(mspace != nullptr);
}
```

### 分配

`Alloc()` 是 `AllocNonvirtual()` 的一层外壳。

``` c++
mirror::Object* Alloc(Thread* self,
                      size_t num_bytes,
                      size_t* bytes_allocated,
                      size_t* usable_size,
                      size_t* bytes_tl_bulk_allocated) override REQUIRES(!lock_) {
  return AllocNonvirtual(self, num_bytes, bytes_allocated, usable_size,
                          bytes_tl_bulk_allocated);
}
```

`AllocNonvirtual()` 调用了 `AllocWithoutGrowthLocked()` 进行分配，之后对分配的空间进行清零操作。

``` c++
inline mirror::Object* DlMallocSpace::AllocNonvirtual(Thread* self, size_t num_bytes,
                                                      size_t* bytes_allocated,
                                                      size_t* usable_size,
                                                      size_t* bytes_tl_bulk_allocated) {
  mirror::Object* obj;
  {
    MutexLock mu(self, lock_);
    obj = AllocWithoutGrowthLocked(self, num_bytes, bytes_allocated, usable_size,
                                   bytes_tl_bulk_allocated);
  }
  if (LIKELY(obj != nullptr)) {
    // Zero freshly allocated memory, done while not holding the space's lock.
    memset(obj, 0, num_bytes);
  }
  return obj;
}
```

`AllocWithoutGrowthLocked()` 最终调用了 dlmalloc 接口 `mspace_malloc()` 实现了对象分配，在分配成功时调用 `AllocationSizeNonvirtual()` 用于确定此次真实分配的空间有多大

``` c++
inline mirror::Object* DlMallocSpace::AllocWithoutGrowthLocked(
    Thread* /*self*/, size_t num_bytes,
    size_t* bytes_allocated,
    size_t* usable_size,
    size_t* bytes_tl_bulk_allocated) {
  mirror::Object* result = reinterpret_cast<mirror::Object*>(mspace_malloc(mspace_, num_bytes));
  if (LIKELY(result != nullptr)) {
    ...
    size_t allocation_size = AllocationSizeNonvirtual(result, usable_size);
    DCHECK(bytes_allocated != nullptr);
    *bytes_allocated = allocation_size;
    *bytes_tl_bulk_allocated = allocation_size;
  }
  return result;
}
```

`AllocationSizeNonvirtual()` 通过调用 dlmalloc 接口 `mspace_usable_size()` 用于确定真实分配的空间大小。

``` c++
inline size_t DlMallocSpace::AllocationSizeNonvirtual(mirror::Object* obj, size_t* usable_size) {
  void* obj_ptr = const_cast<void*>(reinterpret_cast<const void*>(obj));
  size_t size = mspace_usable_size(obj_ptr);
  if (usable_size != nullptr) {
    *usable_size = size;
  }
  return size + kChunkOverhead;
}
```

### 释放

`Free()` 通过调用 dlmalloc 接口 `mspace_free()` 释放对象。

``` c++
size_t DlMallocSpace::Free(Thread* self, mirror::Object* ptr) {
  MutexLock mu(self, lock_);
  ...
  const size_t bytes_freed = AllocationSizeNonvirtual(ptr, nullptr);
  if (kRecentFreeCount > 0) {
    RegisterRecentFree(ptr);
  }
  mspace_free(mspace_, ptr);
  return bytes_freed;
}
```

`Clear()` 与其它 Space 无差，均为对内存进行清零与成员变量复位。

``` c++
void DlMallocSpace::Clear() {
  size_t footprint_limit = GetFootprintLimit();
  madvise(GetMemMap()->Begin(), GetMemMap()->Size(), MADV_DONTNEED);
  live_bitmap_.Clear();
  mark_bitmap_.Clear();
  SetEnd(Begin() + starting_size_);
  mspace_ = CreateMspace(mem_map_.Begin(), starting_size_, initial_size_);
  SetFootprintLimit(footprint_limit);
}
```

### 遍历

由于 dlmalloc 本身的局限性，当对 `DlMallocSpace` 进行遍历时，仅可以确定正在遍历内存的起始地址、内存地址、可被使用大小，而无法遍历实际的对象。

`Walk()` 实现如下。

``` c++
void DlMallocSpace::Walk(void(*callback)(void *start, void *end, size_t num_bytes, void* callback_arg),
                      void* arg) {
  MutexLock mu(Thread::Current(), lock_);
  mspace_inspect_all(mspace_, callback, arg);
  callback(nullptr, nullptr, 0, arg);  // Indicate end of a space.
}
```

## RosAllocSpace

`RosAllocSpace` 采用了一种称为 *Run Of Slot* 的动态内存分配算法，该 Space 的内存分配通过管理类 `RosAllocSpace` 进行操作。

与 `DlMallocSpace` 相似，对于内存空间的管理实际上均交由内存管理类进行实现，因此这里仅一笔带过 `RosAllocSpace` 对对象的创建、分配、释放、遍历，对于内存分配的最终操作还是得看 `RosAlloc`。

### 创建

与其它 Space 区别不大，通过静态函数 `Create()` 创建对象，内部调用了 `CreateMemMap()` 创建匿名映射 `MemMap` 对象，然后调用 `CreateFromMemMap()` 进行构造。

``` c++
RosAllocSpace* RosAllocSpace::Create(const std::string& name,
                                     size_t initial_size,
                                     size_t growth_limit,
                                     size_t capacity,
                                     bool low_memory_mode,
                                     bool can_move_objects) {
  
  ...

  // Memory we promise to rosalloc before it asks for morecore.
  // Note: making this value large means that large allocations are unlikely to succeed as rosalloc
  // will ask for this memory from sys_alloc which will fail as the footprint (this value plus the
  // size of the large allocation) will be greater than the footprint limit.
  size_t starting_size = Heap::kDefaultStartingSize;
  MemMap mem_map = CreateMemMap(name, starting_size, &initial_size, &growth_limit, &capacity);
  if (!mem_map.IsValid()) {
    LOG(ERROR) << "Failed to create mem map for alloc space (" << name << ") of size "
               << PrettySize(capacity);
    return nullptr;
  }

  RosAllocSpace* space = CreateFromMemMap(std::move(mem_map),
                                          name,
                                          starting_size,
                                          initial_size,
                                          growth_limit,
                                          capacity,
                                          low_memory_mode,
                                          can_move_objects);
  // We start out with only the initial size possibly containing objects.
  if (VLOG_IS_ON(heap) || VLOG_IS_ON(startup)) {
    LOG(INFO) << "RosAllocSpace::Create exiting (" << PrettyDuration(NanoTime() - start_time)
        << " ) " << *space;
  }
  return space;
}
```

`CreateFromMemMap()` 逻辑如下，传入的 `MemMap` 内存空间交由 `CreateRosAlloc()` 构建的 `RosAlloc` 对象管理，之后在构造器中转变为成员变量。

``` c++
RosAllocSpace* RosAllocSpace::CreateFromMemMap(MemMap&& mem_map,
                                               const std::string& name,
                                               size_t starting_size,
                                               size_t initial_size,
                                               size_t growth_limit,
                                               size_t capacity,
                                               bool low_memory_mode,
                                               bool can_move_objects) {
  DCHECK(mem_map.IsValid());

  bool running_on_memory_tool = Runtime::Current()->IsRunningOnMemoryTool();

  allocator::RosAlloc* rosalloc = CreateRosAlloc(mem_map.Begin(),
                                                 starting_size,
                                                 initial_size,
                                                 capacity,
                                                 low_memory_mode,
                                                 running_on_memory_tool);
  if (rosalloc == nullptr) {
    LOG(ERROR) << "Failed to initialize rosalloc for alloc space (" << name << ")";
    return nullptr;
  }

  // Protect memory beyond the starting size. MoreCore will add r/w permissions when necessory
  uint8_t* end = mem_map.Begin() + starting_size;
  if (capacity - starting_size > 0) {
    CheckedCall(mprotect, name.c_str(), end, capacity - starting_size, PROT_NONE);
  }

  // Everything is set so record in immutable structure and leave
  uint8_t* begin = mem_map.Begin();
  // TODO: Fix RosAllocSpace to support ASan. There is currently some issues with
  // AllocationSize caused by redzones. b/12944686
  if (running_on_memory_tool) {
    ...
  } else {
    return new RosAllocSpace(std::move(mem_map),
                             initial_size,
                             name,
                             rosalloc,
                             begin,
                             end,
                             begin + capacity,
                             growth_limit,
                             can_move_objects,
                             starting_size,
                             low_memory_mode);
  }
}
```

`CreateRosAlloc()` 通过调用 `RosAlloc` 构造器构造管理对象。

``` c++
allocator::RosAlloc* RosAllocSpace::CreateRosAlloc(void* begin, size_t morecore_start,
                                                   size_t initial_size,
                                                   size_t maximum_size, bool low_memory_mode,
                                                   bool running_on_memory_tool) {
  // clear errno to allow PLOG on error
  errno = 0;
  // create rosalloc using our backing storage starting at begin and
  // with a footprint of morecore_start. When morecore_start bytes of
  // memory is exhaused morecore will be called.
  allocator::RosAlloc* rosalloc = new art::gc::allocator::RosAlloc(
      begin, morecore_start, maximum_size,
      low_memory_mode ?
          art::gc::allocator::RosAlloc::kPageReleaseModeAll :
          art::gc::allocator::RosAlloc::kPageReleaseModeSizeAndEnd,
      running_on_memory_tool);
  if (rosalloc != nullptr) {
    rosalloc->SetFootprintLimit(initial_size);
  } else {
    PLOG(ERROR) << "RosAlloc::Create failed";
  }
  return rosalloc;
}
```

构造器完成了成员变量的初始化工作。

``` c++
RosAllocSpace::RosAllocSpace(MemMap&& mem_map,
                             size_t initial_size,
                             const std::string& name,
                             art::gc::allocator::RosAlloc* rosalloc,
                             uint8_t* begin,
                             uint8_t* end,
                             uint8_t* limit,
                             size_t growth_limit,
                             bool can_move_objects,
                             size_t starting_size,
                             bool low_memory_mode)
    : MallocSpace(name,
                  std::move(mem_map),
                  begin,
                  end,
                  limit,
                  growth_limit,
                  true,
                  can_move_objects,
                  starting_size, initial_size),
      rosalloc_(rosalloc), low_memory_mode_(low_memory_mode) {
  CHECK(rosalloc != nullptr);
}
```

### 分配

像剥洋葱一般剥离函数的所有包装，分配函数 `Alloc()` 的调用链将到达 `AllocCommon()`。

> 调用链：`RosAllocSpace.Alloc()` --> `RosAllocSpace.AllocNonvirtual()` --> `RosAllocSpace.AllocCommon()`。

而 `AllocCommon()` 的实际处理逻辑也在 `rosalloc_`，即前文提及用于管理内存分配的 `RosAlloc` 成员变量，剩余的逻辑便是在其它 Space 中见惯的，对计数器的更改与返回参数的计算。

``` c++
template<bool kThreadSafe>
inline mirror::Object* RosAllocSpace::AllocCommon(Thread* self, size_t num_bytes,
                                                  size_t* bytes_allocated, size_t* usable_size,
                                                  size_t* bytes_tl_bulk_allocated) {
  size_t rosalloc_bytes_allocated = 0;
  size_t rosalloc_usable_size = 0;
  size_t rosalloc_bytes_tl_bulk_allocated = 0;
  if (!kThreadSafe) {
    Locks::mutator_lock_->AssertExclusiveHeld(self);
  }
  mirror::Object* result = reinterpret_cast<mirror::Object*>(
      rosalloc_->Alloc<kThreadSafe>(self, num_bytes, &rosalloc_bytes_allocated,
                                    &rosalloc_usable_size,
                                    &rosalloc_bytes_tl_bulk_allocated));
  if (LIKELY(result != nullptr)) {
    
    ...

    DCHECK(bytes_allocated != nullptr);
    *bytes_allocated = rosalloc_bytes_allocated;
    DCHECK_EQ(rosalloc_usable_size, rosalloc_->UsableSize(result));
    if (usable_size != nullptr) {
      *usable_size = rosalloc_usable_size;
    }
    DCHECK(bytes_tl_bulk_allocated != nullptr);
    *bytes_tl_bulk_allocated = rosalloc_bytes_tl_bulk_allocated;
  }
  return result;
}
```

### 释放

实际对内存的操作也在 `rosalloc_` 中。

``` c++
size_t RosAllocSpace::Free(Thread* self, mirror::Object* ptr) {
  if (kDebugSpaces) {
    CHECK(ptr != nullptr);
    CHECK(Contains(ptr)) << "Free (" << ptr << ") not in bounds of heap " << *this;
  }
  if (kRecentFreeCount > 0) {
    MutexLock mu(self, lock_);
    RegisterRecentFree(ptr);
  }
  return rosalloc_->Free(self, ptr);
}
```

老样子，`Clear()` 同样是内存清零与成员变量复位。

``` c++
void RosAllocSpace::Clear() {
  size_t footprint_limit = GetFootprintLimit();
  madvise(GetMemMap()->Begin(), GetMemMap()->Size(), MADV_DONTNEED);
  live_bitmap_.Clear();
  mark_bitmap_.Clear();
  SetEnd(begin_ + starting_size_);
  delete rosalloc_;
  rosalloc_ = CreateRosAlloc(mem_map_.Begin(),
                             starting_size_,
                             initial_size_,
                             NonGrowthLimitCapacity(),
                             low_memory_mode_,
                             Runtime::Current()->IsRunningOnMemoryTool());
  SetFootprintLimit(footprint_limit);
}
```

### 遍历

`Walk()` 是对 `InspectAllRosAlloc()` 的包装。

``` c++
void RosAllocSpace::Walk(void(*callback)(void *start, void *end, size_t num_bytes, void* callback_arg),
                         void* arg) {
  InspectAllRosAlloc(callback, arg, true);
}
```

`InspectAllRosAlloc()` 对内存遍历的最终实现为 `rosalloc_` 的 `InspectAll()`，但该函数不可直接调用，这是由于 `RosAlloc` 存在一个重要的特性，**必须在所有线程挂起的情况下才可遍历对象**。

因此 `InspectAllRosAlloc()` 依照 `RosAlloc` 遍历的一个重要特性额外添加了逻辑，代码中总共呈现了三个分支。

- 第一分支，对应代码中的 `Locks::mutator_lock_->IsExclusiveHeld(self)`，此时虚拟机处于挂起状态，可直接调用。

- 第二分支，对应代码中的 `Locks::mutator_lock_->IsSharedHeld(self)`，此时当前线程处于挂起状态，但其它线程还没有，需要先通过构造 `ScopedThreadSuspension()` 释放 `mutator_lock_` 锁，再调用 `InspectAllRosAllocWithSuspendAll()` 实现内存遍历。

- 第三分支，对应代码中的 `else` 分支，直接调用 `InspectAllRosAllocWithSuspendAll()`。

``` c++
void RosAllocSpace::InspectAllRosAlloc(void (*callback)(void *start, void *end, size_t num_bytes, void* callback_arg),
                                       void* arg, bool do_null_callback_at_end) NO_THREAD_SAFETY_ANALYSIS {
  // TODO: NO_THREAD_SAFETY_ANALYSIS.
  Thread* self = Thread::Current();
  if (Locks::mutator_lock_->IsExclusiveHeld(self)) {
    // The mutators are already suspended. For example, a call path
    // from SignalCatcher::HandleSigQuit().
    rosalloc_->InspectAll(callback, arg);
    if (do_null_callback_at_end) {
      callback(nullptr, nullptr, 0, arg);  // Indicate end of a space.
    }
  } else if (Locks::mutator_lock_->IsSharedHeld(self)) {
    // The mutators are not suspended yet and we have a shared access
    // to the mutator lock. Temporarily release the shared access by
    // transitioning to the suspend state, and suspend the mutators.
    ScopedThreadSuspension sts(self, kSuspended);
    InspectAllRosAllocWithSuspendAll(callback, arg, do_null_callback_at_end);
  } else {
    // The mutators are not suspended yet. Suspend the mutators.
    InspectAllRosAllocWithSuspendAll(callback, arg, do_null_callback_at_end);
  }
}
```

`InspectAllRosAllocWithSuspendAll()` 的逻辑与 `InspectAllRosAlloc()` 中的第一分支类似，但在调用 `rosalloc_` 的 `InspectAll()` 前通过加锁挂起所有线程。

``` c++
void RosAllocSpace::InspectAllRosAllocWithSuspendAll(
    void (*callback)(void *start, void *end, size_t num_bytes, void* callback_arg),
    void* arg, bool do_null_callback_at_end) NO_THREAD_SAFETY_ANALYSIS {
  // TODO: NO_THREAD_SAFETY_ANALYSIS.
  Thread* self = Thread::Current();
  ScopedSuspendAll ssa(__FUNCTION__);
  MutexLock mu(self, *Locks::runtime_shutdown_lock_);
  MutexLock mu2(self, *Locks::thread_list_lock_);
  rosalloc_->InspectAll(callback, arg);
  if (do_null_callback_at_end) {
    callback(nullptr, nullptr, 0, arg);  // Indicate end of a space.
  }
}
```

### RosAlloc

> 由于 `RosAlloc` 逻辑相对复杂，部分代码的说明以注释的方式添加。

在对 *Run Of Slot* 分配算法与 `RosAlloc` 实现进行讲解之前，提前对相关的概念进行说明可以有效降低理解难度。

- 内存页

内存资源在 `RosAlloc` 被划分为多个内存页进行管理，每个内存页的大小被常量 `kPageSize` 所定义：4096 字节。

内存资源的首地址被保存在了成员变量 `base_` 中，而每个内存页的状态被保存在 `page_map_` 中，在之后的逻辑中可将 `page_map_` 作为数组看待，数组长度等于内存资源包含的内存页数量。

> 这里 `page_map_` 可以作为数组看待的意思是，`page_map_` 是通过动态分配的方式创建的。类似于通过 `malloc()` 分配位于堆上的数组，只不过这里用的不是 `malloc()`，而是内存映射，这个 `MemMap` 对象被保存在 `page_map_mem_map_` 成员变量中。

- Slot

Slot 是分配内存空间的最小单位，`RosAlloc` 提供了 `kNumOfSizeBrackets` 种粒度类型的 Slot（`kNumOfSizeBrackets` 的值为 42），从 8 Byte 到 2048 Byte。

Slot 粒度的大小被存放在文件级数组 `bracketSizes` 中。举个例子，`bracketSizes[0]` 的值为 8，对应 8 Byte 大小的 Slot，`bracketSizes[41]` 的值为 2048，对应 2048 Byte 大小的 Slot。

- Run

Run 代表一个内存管理单位，它在 `RosAlloc` 中扮演的角色近似于 `RegionSpace` 中的 Region。Run 对应了 `RosAlloc` 的内部类 `Run`。

一个 Run 中仅包含一种粒度类型的 Slot，其管理的内存页数量与其对应 Slot 的粒度相关，内存页数保存在文件级数组 `numsOfPages` 中。举个例子，如果一个 Run 对应的 Slot 为粒度类型 15，Slot 的大小为 `bracketSizes[15]`，则 Run 管理内存页数为 `numsOfPages[15]`，管理内存大小字节数为 `numsOfPages[15] * kPageSize`。

但实际 Run 管理的 Slot 数量远没有  `numsOfPages[x] * kPageSize / bracketSizes[15]` 那么多，因为 Run 不仅包括了 Slot，还包括了存储有关数据的 Header，因此 Slot 占用的空间需要排除 Header 的大小，这一值被计算之后存于文件级数组 `numsOfSlots` 中，如上文例子中的 Run 包含的 Slot 的数量为 `numsOfSlots[15]`。

`Run` 中的 Slot 通过成员数组 `slot` 管理。

- Free Page Run

FreePageRun 与 Run 是一个相对的概念，它代表 `RosAlloc` 管理的内存页中不作为 Run 的空闲内存页。

`FreePageRun` 是一个不存在任何成员变量的工具类，这意味着可直接将内存地址转换为 `FreePageRun` 的指针后进行操作。

`FreePageRun` 需要与 `RosAlloc` 高度绑定，许多成员函数都需要传入其对应的 `RosAlloc` 对象，因为虽然 `FreePageRun` 不存在成员变量，但仍需要对一些数据进行记录，而这一部分数据其实保存在 `RosAlloc` 的成员变量中，如 `RosAlloc` 的成员变量 `free_page_runs_` 为存储当前对象中 `FreePageRun` 的容器。

#### 创建

见构造函数。

``` c++
RosAlloc::RosAlloc(void* base, size_t capacity, size_t max_capacity,
                   PageReleaseMode page_release_mode, bool running_on_memory_tool,
                   size_t page_release_size_threshold)
    : base_(reinterpret_cast<uint8_t*>(base)), footprint_(capacity),
      capacity_(capacity), max_capacity_(max_capacity),
      lock_("rosalloc global lock", kRosAllocGlobalLock),
      bulk_free_lock_("rosalloc bulk free lock", kRosAllocBulkFreeLock),
      page_release_mode_(page_release_mode),
      page_release_size_threshold_(page_release_size_threshold),
      is_running_on_memory_tool_(running_on_memory_tool) {
  
  ...

  /*
   * 对分配的内存空间进行清零操作
   */

  // Zero the memory explicitly (don't rely on that the mem map is zero-initialized).
  if (!kMadviseZeroes) {
    memset(base_, 0, max_capacity);
  }

  ...

  /*
   * 懒初始化前文提及的文件级变量，如 bracketSizes、numsOfPages、numsOfSlots 等
   */

  if (!initialized_) {
    Initialize();
  }

  ...

  for (size_t i = 0; i < kNumOfSizeBrackets; i++) {

    /*
     * 创建锁数组，每一种粒度类型的 Slot 都对应了一个锁，以提高内存分配的并发效率
     */
    
    size_bracket_lock_names_[i] =
        StringPrintf("an rosalloc size bracket %d lock", static_cast<int>(i));
    size_bracket_locks_[i] = new Mutex(size_bracket_lock_names_[i].c_str(), kRosAllocBracketLock);

    /*
     * current_runs_ 数组各个元素代表当前用于分配各个粒度 Slot 的 Run 对象
     *
     * 初始状态它们都等于 dedicated_full_run_，它的用途与 RegionSpace 中的 full_region_ 类似，代表了一个不可继续分配内存空间的对象，仅用于标识，没有实际用途
     */

    current_runs_[i] = dedicated_full_run_;
  }

  ...

  /*
   * 计算当前使用内存页数与最大可用内存页数
   *
   * 当前使用内存页包括已作为 Run 使用的和未作为 Run（也就是处于 Free Page Run 类型）的内存页，
   * 在当前内存页逐步用于 Run 而不足时，会增加当前使用内存页的大小，最大阈值即最大可用内存页数，这个值是通过 RosAlloc 管理内存空间的大小计算所得的
   *
   * 在后面的解析中也需要时刻注意 “当前使用内存” 和 “最大可用内存” 的区别
   */

  size_t num_of_pages = footprint_ / kPageSize;
  size_t max_num_of_pages = max_capacity_ / kPageSize;
  std::string error_msg;

  /*
   * 通过匿名内存映射的方式创建 page_map_ 数组，用于标识内存页状态
   */

  page_map_mem_map_ = MemMap::MapAnonymous("rosalloc page map",
                                           RoundUp(max_num_of_pages, kPageSize),
                                           PROT_READ | PROT_WRITE,
                                           /*low_4gb=*/ false,
                                           &error_msg);

  ...

  /*
   * 赋值成员变量
   */

  page_map_ = page_map_mem_map_.Begin();
  page_map_size_ = num_of_pages;
  max_page_map_size_ = max_num_of_pages;

  /*
   * 初始化 free_page_run_size_map_ 的大小，其类型为 vector，用于保存 Free Page Run 空间大小的信息
   *
   * 注意，该 vector 的长度不代表 RosAlloc 中 Free Page Run 的数量！
   * 虽然其类型为 vector，但在使用上其实更接近 map，vector 下标对应的是 Free Page Run 的起始地址，如 free_page_run_size_map_[15] 表示与第 15 号内存页起始地址相同的 Free Page Run 所包含的内存大小
   */

  free_page_run_size_map_.resize(num_of_pages);

  /*
   * 当前 RosAlloc 不包含任何 Run，因此可所有当前使用内存视为一个大型的 Free Page Run，其首地址为管理内存的首地址，大小为当前使用内存的大小
   */
  
  FreePageRun* free_pages = reinterpret_cast<FreePageRun*>(base_);
  
  ...
  
  free_pages->SetByteSize(this, capacity_);
  
  ...

  /*
   * 释放 Free Page Run 中所有内存页，这涉及到内存页的多种状态，后文会进行说明
   */

  free_pages->ReleasePages(this);

  ...

  /*
   * 将当前内存页记录到 free_page_runs_ 中
   */

  free_page_runs_.insert(free_pages);
  
  ...
}
```

在这里需要对额外两个部分进行展开，以进一步说明无成员变量的 `FreePageRun` 是如何与 `RosAlloc` 进行联动的。

第一个部分是 `SetByteSize()` 与 `ByteSize()`，后一个函数在第二个部分的 `ReleasePages()` 中被调用。这两个函数是 Free Page Run 包含内存大小的 Getter 与 Setter。

`SetByteSize()` 实现如下，需要传入 `FreePageRun` 对象对应的 `RosAlloc` 对象。

``` c++
void SetByteSize(RosAlloc* rosalloc, size_t byte_size)
    REQUIRES(rosalloc->lock_) {
  DCHECK_EQ(byte_size % kPageSize, static_cast<size_t>(0));
  uint8_t* fpr_base = reinterpret_cast<uint8_t*>(this);
  size_t pm_idx = rosalloc->ToPageMapIndex(fpr_base);
  rosalloc->free_page_run_size_map_[pm_idx] = byte_size;
}
```

`ToPageMapIndex()` 获取到的是当前 `FreePageRun` 对应的第一个内存页的地址（该函数的实现是一个简单的除法操作，这里不再展开），之后通过修改 `rosalloc` 的 `free_page_run_size_map_` 记录实现修改。

`ByteSize()` 同理。

``` c++
size_t ByteSize(RosAlloc* rosalloc) const REQUIRES(rosalloc->lock_) {
  const uint8_t* fpr_base = reinterpret_cast<const uint8_t*>(this);
  size_t pm_idx = rosalloc->ToPageMapIndex(fpr_base);
  size_t byte_size = rosalloc->free_page_run_size_map_[pm_idx];
  DCHECK_GE(byte_size, static_cast<size_t>(0));
  DCHECK_ALIGNED(byte_size, kPageSize);
  return byte_size;
}
```

第二个部分是 `ReleasePages()`，函数实现了对 Free Page Run 所包含的所有内存页的释放操作。

在通过 `ByteSize()` 获取到 Free Page Run 包含内存的大小后，通过 `ShouldReleasePages()` 根据释放模式进行判断，满足条件时通过 `ReleasePageRange()` 进行释放。

``` c++
void ReleasePages(RosAlloc* rosalloc) REQUIRES(rosalloc->lock_) {
  uint8_t* start = reinterpret_cast<uint8_t*>(this);
  size_t byte_size = ByteSize(rosalloc);
  DCHECK_EQ(byte_size % kPageSize, static_cast<size_t>(0));
  if (ShouldReleasePages(rosalloc)) {
    rosalloc->ReleasePageRange(start, start + byte_size);
  }
}
```

`ReleasePageRange()` 便是一个简单的限定范围内存空间释放。先对对应的内存空间进行清零操作，再通过遍历范围内所有内存页的状态（该状态存储在 `page_map_` 数组中），当内存页状态为 `kPageMapEmpty` 时，释放内存量 `reclaimed_bytes` 增长，然后将内存页状态置换为 `kPageMapReleased`。

> `kPageMapEmpty` 表示暂未能用于分配的状态，可能是已经清零的，也可能仍为脏内存页。
>
> `kPageMapReleased` 表示已释放，可用于 Run 的内存页。

``` c++
size_t RosAlloc::ReleasePageRange(uint8_t* start, uint8_t* end) {
  
  ...

  if (!kMadviseZeroes) {
    // TODO: Do this when we resurrect the page instead.
    memset(start, 0, end - start);
  }
  CHECK_EQ(madvise(start, end - start, MADV_DONTNEED), 0);
  size_t pm_idx = ToPageMapIndex(start);
  size_t reclaimed_bytes = 0;
  // Calculate reclaimed bytes and upate page map.
  const size_t max_idx = pm_idx + (end - start) / kPageSize;
  for (; pm_idx < max_idx; ++pm_idx) {
    DCHECK(IsFreePage(pm_idx));
    if (page_map_[pm_idx] == kPageMapEmpty) {
      // Mark the page as released and update how many bytes we released.
      reclaimed_bytes += kPageSize;
      page_map_[pm_idx] = kPageMapReleased;
    }
  }
  return reclaimed_bytes;
}
```

#### 分配

`RosAlloc` 通过函数 `Alloc()` 分配内存，最终 `RosAlloc` 将通过所需分配内存的大小选择合适粒度的 Slot，并寻找对应的可用 Run，在其内部进行分配。

`Alloc()` 的实现如下。

``` c++
template<bool kThreadSafe>
inline ALWAYS_INLINE void* RosAlloc::Alloc(Thread* self, size_t size, size_t* bytes_allocated,
                                           size_t* usable_size,
                                           size_t* bytes_tl_bulk_allocated) {
  if (UNLIKELY(size > kLargeSizeThreshold)) {
    return AllocLargeObject(self, size, bytes_allocated, usable_size,
                            bytes_tl_bulk_allocated);
  }
  void* m;
  if (kThreadSafe) {
    m = AllocFromRun(self, size, bytes_allocated, usable_size, bytes_tl_bulk_allocated);
  } else {
    m = AllocFromRunThreadUnsafe(self, size, bytes_allocated, usable_size,
                                 bytes_tl_bulk_allocated);
  }
  // Check if the returned memory is really all zero.
  if (ShouldCheckZeroMemory() && m != nullptr) {
    uint8_t* bytes = reinterpret_cast<uint8_t*>(m);
    for (size_t i = 0; i < size; ++i) {
      DCHECK_EQ(bytes[i], 0);
    }
  }
  return m;
}
```

根据所需分配空间的大小是否超过 `kLargeSizeThreshold`（其值为 2048）个字节的空间，即是否能被最大粒度的 Slot 存储，`Alloc()` 将调用 `AllocLargeObject()` 或 `AllocFromRun()`（绝大部分情况中模版参数 `kThreadSafe` 为 `true`，因此这里不解析 `AllocFromRunThreadUnsafe()` 的实现）。

先看 `AllocFromRun()`。

``` c++
void* RosAlloc::AllocFromRun(Thread* self, size_t size, size_t* bytes_allocated,
                             size_t* usable_size, size_t* bytes_tl_bulk_allocated) {
  
  ...

  /*
   * 通过所需分配空间大小 size 计算合适的 Slot 粒度，返回 Slot 粒度对应的下标和粒度大小
   *
   * 非大型对象分配的情况下，分配空间要小于等于 Slot 的粒度
   *
   * bracket_size 是一个返回参数
   */

  size_t bracket_size;
  size_t idx = SizeToIndexAndBracketSize(size, &bracket_size);
  void* slot_addr;

  /*
   * kNumThreadLocalSizeBrackets 的值为 16，对应 Slot 粒度为 128 Byte，当分配粒度低于此时执行此分支
   *
   * 该分支会尝试在线程本地空间中进行分配，但需要注意到是这里所提到的线程本地空间不是前文提到的 TLAB，两者概念类似但实现不同，Thread 对 RosAlloc 提供了单独的支持
   */

  if (LIKELY(idx < kNumThreadLocalSizeBrackets)) {

    /*
     * 通过 Thread.GetRosAllocRun() 获取一个可用于分配的 Run 对象
     *
     * Thread 内部的 tlsPtr_ 包含一个 16 元素的 Run 数组 rosalloc_runs，用于记录 128 Byte 以下 16 种粒度 Slot 对应的 Run，
     * 初始状态下该数组的默认值均为 dedicated_full_run_，在这种情况下分配空间必然失败
     */

    // Use a thread-local run.
    Run* thread_local_run = reinterpret_cast<Run*>(self->GetRosAllocRun(idx));
    
    ...

    slot_addr = thread_local_run->AllocSlot();
    
    ...

    /*
     * 分配失败，进行下一步操作
     */

    if (UNLIKELY(slot_addr == nullptr)) {
      ...

      /*
       * 对粒度进行加锁，因为前面的分配在线程本地空间中操作，所以无需加锁
       */

      MutexLock mu(self, *size_bracket_locks_[idx]);
      bool is_all_free_after_merge;
      
      /*
       * 将 Run 内用于线程本地空间的空闲 Slot 合并到普通的空闲 Slot 列表中，合并成功则返回 true
       *
       * 当然对 dedicated_full_run_ 执行这一操作是返回 false 的
       */

      if (thread_local_run->MergeThreadLocalFreeListToFreeList(&is_all_free_after_merge)) {
        ...
      } else {
        ...

        /*
         * 取消将当前获取到的 Run 用于线程本地空间
         */

        if (thread_local_run != dedicated_full_run_) {
          thread_local_run->SetIsThreadLocal(false);
          ...
        }

        /*
         * 调用 RefillRun() 重新获取一个对应 Slot 粒度的、可用的 Run
         *
         * 之后将其用于线程本地空间
         */

        thread_local_run = RefillRun(self, idx);
        
        if (UNLIKELY(thread_local_run == nullptr)) {
          self->SetRosAllocRun(idx, dedicated_full_run_);
          return nullptr;
        }
        ...
        thread_local_run->SetIsThreadLocal(true);
        self->SetRosAllocRun(idx, thread_local_run);
        ...
      }
      
      ...

      // Account for all the free slots in the new or refreshed thread local run.
      *bytes_tl_bulk_allocated = thread_local_run->NumberOfFreeSlots() * bracket_size;

      /*
       * 重新分配空间
       */

      slot_addr = thread_local_run->AllocSlot();
      
      ...

    } else {
      // The slot is already counted. Leave it as is.
      *bytes_tl_bulk_allocated = 0;
    }
    
    ...

    *bytes_allocated = bracket_size;
    *usable_size = bracket_size;
  } else {

    /*
     * 对粒度进行加锁，注意这里加锁在分配空间之前
     */

    // Use the (shared) current run.
    MutexLock mu(self, *size_bracket_locks_[idx]);

    /*
     * 分配空间
     */
    
    slot_addr = AllocFromCurrentRunUnlocked(self, idx);
    
    ...

    if (LIKELY(slot_addr != nullptr)) {
      *bytes_allocated = bracket_size;
      *usable_size = bracket_size;
      *bytes_tl_bulk_allocated = bracket_size;
    }
  }
  // Caller verifies that it is all 0.
  return slot_addr;
}
```

先看分配空间不小于 128 Byte 的逻辑，最终调用到了 `AllocFromCurrentRunUnlocked()` 进行分配，看一下其中的逻辑。

``` c++
inline void* RosAlloc::AllocFromCurrentRunUnlocked(Thread* self, size_t idx) {

  /*
   * 获取当前粒度所用的 Run
   */
  
  Run* current_run = current_runs_[idx];
  ...
  void* slot_addr = current_run->AllocSlot();

  /*
   * 分配失败时，执行下一步操作
   */

  if (UNLIKELY(slot_addr == nullptr)) {
    
    ...

    /*
     * 调用 RefillRun() 重新获取一个对应 Slot 粒度的、可用的 Run
     */

    current_run = RefillRun(self, idx);
    if (UNLIKELY(current_run == nullptr)) {
      // Failed to allocate a new run, make sure that it is the dedicated full run.
      current_runs_[idx] = dedicated_full_run_;
      return nullptr;
    }
    ...
    current_run->SetIsThreadLocal(false);
    current_runs_[idx] = current_run;

    /*
     * 重新分配空间
     */
    
    slot_addr = current_run->AllocSlot();
    ...
  }
  return slot_addr;
}
```

可见，在两部分操作中，寻找新的可用 `Run` 对象均调用了 `RefillRun()`，在 `Run` 对象分配空间均调用了 `Run.AllocSlot()`。

先看一下 `AllocSlot()` 如何在 `Run` 中分配空间，该方法较为简单，直接获取一个可用的 `Slot` 对象返回。

``` c++
inline void* RosAlloc::Run::AllocSlot() {
  Slot* slot = free_list_.Remove();
  if (kTraceRosAlloc && slot != nullptr) {
    const uint8_t idx = size_bracket_idx_;
    ...
  }
  return slot;
}
```

之后是 `RefillRun()`。

``` c++
RosAlloc::Run* RosAlloc::RefillRun(Thread* self, size_t idx) {
  
  /*
   * 从 non_full_runs_ 中获取一个还未完全用尽 Slot 资源的 Run 对象
   *
   * non_full_runs_ 类型为一个 set 数组
   */

  auto* const bt = &non_full_runs_[idx];
  if (!bt->empty()) {
    // If there's one, use it as the current run.
    auto it = bt->begin();
    Run* non_full_run = *it;
    ...

    /*
     * 从 non_full_runs_ 中将其移除
     */
    
    bt->erase(it);

    return non_full_run;
  }

  /*
   * 如果 non_full_runs_ 记录的 Run 对象中不包含可用的了，通过 AllocRun() 创建一个新的
   */

  // If there's none, allocate a new run and use it as the current run.
  return AllocRun(self, idx);
}
```

创建 `Run` 对象的实现位于 `AllocRun()` 中。

``` c++
RosAlloc::Run* RosAlloc::AllocRun(Thread* self, size_t idx) {
  RosAlloc::Run* new_run = nullptr;
  {
    MutexLock mu(self, lock_);
    new_run = reinterpret_cast<Run*>(AllocPages(self, numOfPages[idx], kPageMapRun));
  }
  if (LIKELY(new_run != nullptr)) {
    if (kIsDebugBuild) {
      new_run->magic_num_ = kMagicNum;
    }
    new_run->size_bracket_idx_ = idx;
    DCHECK(!new_run->IsThreadLocal());
    DCHECK(!new_run->to_be_bulk_freed_);
    if (kUsePrefetchDuringAllocRun && idx < kNumThreadLocalSizeBrackets) {
      // Take ownership of the cache lines if we are likely to be thread local run.
      if (kPrefetchNewRunDataByZeroing) {
        // Zeroing the data is sometimes faster than prefetching but it increases memory usage
        // since we end up dirtying zero pages which may have been madvised.
        new_run->ZeroData();
      } else {
        const size_t num_of_slots = numOfSlots[idx];
        const size_t bracket_size = bracketSizes[idx];
        const size_t num_of_bytes = num_of_slots * bracket_size;
        uint8_t* begin = reinterpret_cast<uint8_t*>(new_run) + headerSizes[idx];
        for (size_t i = 0; i < num_of_bytes; i += kPrefetchStride) {
          __builtin_prefetch(begin + i);
        }
      }
    }
    new_run->InitFreeList();
  }
  return new_run;
}
```

排除对各种成员变量的初始化操作，可发现 `Run` 对象管理的内存页是通过 `AllocPages()` 获得的，而该函数便是寻找 Free Page Run 进行分配的。

``` c++
void* RosAlloc::AllocPages(Thread* self, size_t num_pages, uint8_t page_map_type) {
  lock_.AssertHeld(self);
  DCHECK(page_map_type == kPageMapRun || page_map_type == kPageMapLargeObject);
  FreePageRun* res = nullptr;

  /*
   * 计算所需获取的内存大小
   */
  
  const size_t req_byte_size = num_pages * kPageSize;

  /*
   * 遍历 free_page_runs_，寻找包含内存空间不小于 req_byte_size 的 Free Page Run
   */

  // Find the lowest address free page run that's large enough.
  for (auto it = free_page_runs_.begin(); it != free_page_runs_.end(); ) {
    FreePageRun* fpr = *it;
    ...
    size_t fpr_byte_size = fpr->ByteSize(this);
    ...
    if (req_byte_size <= fpr_byte_size) {

      /*
       * 将找到的 FreePageRun 对象移出 free_page_runs_
       */

      // Found one.
      it = free_page_runs_.erase(it);
      ...

      /*
       * 如果分配空间小于 Free Page Run 包含内存空间的大小，则进行分割
       *
       * 前一部分占 req_byte_size 的空间返回给外部使用，剩余的部分作为新的 Free Page Run 扔回 free_page_runs_
       */

      if (req_byte_size < fpr_byte_size) {
        // Split.
        FreePageRun* remainder =
            reinterpret_cast<FreePageRun*>(reinterpret_cast<uint8_t*>(fpr) + req_byte_size);
        ...
        remainder->SetByteSize(this, fpr_byte_size - req_byte_size);
        ...
        // Don't need to call madvise on remainder here.
        free_page_runs_.insert(remainder);
        ...
        fpr->SetByteSize(this, req_byte_size);
        DCHECK_EQ(fpr->ByteSize(this) % kPageSize, static_cast<size_t>(0));
      }
      res = fpr;
      break;
    } else {
      ++it;
    }
  }

  /*
   * 不存在可用的 Free Page Run，此时需要对当前使用内存进行扩容
   */

  // Failed to allocate pages. Grow the footprint, if possible.
  if (UNLIKELY(res == nullptr && capacity_ > footprint_)) {

    /*
     * 计算当前使用内存的末尾是不是一个 Free Page Run，并记录其起始地址与大小
     *
     * 如果末尾不是一个 Free Page Run，则将其当做一个大小为 0 的 Free Page Run
     */

    FreePageRun* last_free_page_run = nullptr;
    size_t last_free_page_run_size;
    auto it = free_page_runs_.rbegin();
    if (it != free_page_runs_.rend() && (last_free_page_run = *it)->End(this) == base_ + footprint_) {
      // There is a free page run at the end.
      ...
      last_free_page_run_size = last_free_page_run->ByteSize(this);
    } else {
      // There is no free page run at the end.
      last_free_page_run_size = 0;
    }
    ...
    if (capacity_ - footprint_ + last_free_page_run_size >= req_byte_size) {

      /*
       * 进行扩容操作
       */

      // If we grow the heap, we can allocate it.
      size_t increment = std::min(std::max(2 * MB, req_byte_size - last_free_page_run_size),
                                  capacity_ - footprint_);
      ...
      size_t new_footprint = footprint_ + increment;
      size_t new_num_of_pages = new_footprint / kPageSize;
      ...
      page_map_size_ = new_num_of_pages;
      ...
      free_page_run_size_map_.resize(new_num_of_pages);
      ArtRosAllocMoreCore(this, increment);

      /*
       * 对末尾的 Free Page Run 重新设置
       */

      if (last_free_page_run_size > 0) {
        // There was a free page run at the end. Expand its size.
        ...
        last_free_page_run->SetByteSize(this, last_free_page_run_size + increment);
        ...
      } else {
        // Otherwise, insert a new free page run at the end.
        FreePageRun* new_free_page_run = reinterpret_cast<FreePageRun*>(base_ + footprint_);
        ...
        new_free_page_run->SetByteSize(this, increment);
        ...
        free_page_runs_.insert(new_free_page_run);
        ...
      }
      ...
      footprint_ = new_footprint;

      /*
       * 与上一部分操作相似：
       *
       * 将 FreePageRun 对象移出 free_page_runs_
       * 
       * 如果分配空间小于 Free Page Run 包含内存空间的大小，则进行分割
       *
       * 前一部分占 req_byte_size 的空间返回给外部使用，剩余的部分作为新的 Free Page Run 扔回 free_page_runs_
       */

      // And retry the last free page run.
      it = free_page_runs_.rbegin();
      ...
      FreePageRun* fpr = *it;
      ...
      size_t fpr_byte_size = fpr->ByteSize(this);
      ...
      free_page_runs_.erase(fpr);
      ...
      if (req_byte_size < fpr_byte_size) {
        // Split if there's a remainder.
        FreePageRun* remainder = reinterpret_cast<FreePageRun*>(reinterpret_cast<uint8_t*>(fpr) + req_byte_size);
        ...
        remainder->SetByteSize(this, fpr_byte_size - req_byte_size);
        ...
        free_page_runs_.insert(remainder);
        ...
        fpr->SetByteSize(this, req_byte_size);
        ...
      }
      res = fpr;
    }
  }

  /*
   * 根据传入的内存页用途设置内存页状态
   *
   * 举个例子，当通过该函数分配的这些内存页作为 Run 使用时，此时传入的 page_map_type 为 kPageMapRun，
   * 则将分配成功的这些内存页，首页设置为 kPageMapRun，其余页设置为 kPageMapRunPart
   */

  if (LIKELY(res != nullptr)) {
    // Update the page map.
    size_t page_map_idx = ToPageMapIndex(res);
    for (size_t i = 0; i < num_pages; i++) {
      DCHECK(IsFreePage(page_map_idx + i));
    }
    switch (page_map_type) {
    case kPageMapRun:
      page_map_[page_map_idx] = kPageMapRun;
      for (size_t i = 1; i < num_pages; i++) {
        page_map_[page_map_idx + i] = kPageMapRunPart;
      }
      break;
    case kPageMapLargeObject:
      page_map_[page_map_idx] = kPageMapLargeObject;
      for (size_t i = 1; i < num_pages; i++) {
        page_map_[page_map_idx + i] = kPageMapLargeObjectPart;
      }
      break;
    default:
      LOG(FATAL) << "Unreachable - page map type: " << static_cast<int>(page_map_type);
      UNREACHABLE();
    }
    
    ...

    return res;
  }

  ...

  return nullptr;
}
```

最后回头看看 `AllocLargeObject()`，其逻辑也较为简单，分配对象不通过 Run 实现，而是通过 `AllocPages()` 分配内存页，直接在内存页中存储对象。

``` c++
void* RosAlloc::AllocLargeObject(Thread* self, size_t size, size_t* bytes_allocated,
                                 size_t* usable_size, size_t* bytes_tl_bulk_allocated) {
  
  ...

  size_t num_pages = RoundUp(size, kPageSize) / kPageSize;
  void* r;
  {
    MutexLock mu(self, lock_);
    r = AllocPages(self, num_pages, kPageMapLargeObject);
  }
  if (UNLIKELY(r == nullptr)) {
    ...
    return nullptr;
  }
  const size_t total_bytes = num_pages * kPageSize;
  *bytes_allocated = total_bytes;
  *usable_size = total_bytes;
  *bytes_tl_bulk_allocated = total_bytes;
  
  ...

  return r;
}
```

#### 释放

`Free()` 是对 `FreeInternal()` 的包装。

``` c++
size_t RosAlloc::Free(Thread* self, void* ptr) {
  ReaderMutexLock rmu(self, bulk_free_lock_);
  return FreeInternal(self, ptr);
}
```

`FreeInternal()` 通过寻找对象所在的内存页，并判断内存页类型，根据不同类型实现不同方式的回收。

``` c++
size_t RosAlloc::FreeInternal(Thread* self, void* ptr) {

  ...

  /*
   * 获取所在内存页的下标
   */

  size_t pm_idx = RoundDownToPageMapIndex(ptr);
  Run* run = nullptr;
  {
    MutexLock mu(self, lock_);
    DCHECK_LT(pm_idx, page_map_size_);
    uint8_t page_map_entry = page_map_[pm_idx];
    
    ...

    switch (page_map_[pm_idx]) {

      /*
       * 如果内存页用于大型对象，说明该对象就是大型对象，并可通过大型对象的分配实现，直接断言对象地址为内存页首地址
       *
       * 这里调用了 return，因此不会涉及到后面关于 Run 的操作
       */
      
      case kPageMapLargeObject:
        return FreePages(self, ptr, false);

      /*
       * 基于上一个 case，该 case 不可达
       */

      case kPageMapLargeObjectPart:
        LOG(FATAL) << "Unreachable - page map type: " << static_cast<int>(page_map_[pm_idx]);
        UNREACHABLE();

      /*
       * 如果对象在 Run 中进行分配，找到 Run 管理的第一个内存页
       *
       * 注意该 case 没有 break 操作，在找到对应内存页后执行与下一个 case 相同的操作
       */

      case kPageMapRunPart: {
        // Find the beginning of the run.
        do {
          --pm_idx;
          DCHECK_LT(pm_idx, capacity_ / kPageSize);
        } while (page_map_[pm_idx] != kPageMapRun);
        FALLTHROUGH_INTENDED;
      
      /*
       * 确定对象所在的 Run
       */

      case kPageMapRun:
        run = reinterpret_cast<Run*>(base_ + pm_idx * kPageSize);
        ...
        break;

      /*
       * 以下 case 不可达，对象不可能存在于未使用的内存页中
       */

      case kPageMapReleased:
      case kPageMapEmpty:
        LOG(FATAL) << "Unreachable - page map type: " << static_cast<int>(page_map_[pm_idx]);
        UNREACHABLE();
      }
      default:
        LOG(FATAL) << "Unreachable - page map type: " << static_cast<int>(page_map_[pm_idx]);
        UNREACHABLE();
    }
  }
  ...

  /*
   * 在 Run 中释放对象
   */

  return FreeFromRun(self, ptr, run);
}
```

接下来看 `FreeFromRun()`。

``` c++
size_t RosAlloc::FreeFromRun(Thread* self, void* ptr, Run* run) {

  ...

  /*
   * 确定 Run 对应 Slot 的粒度于大小
   */

  const size_t idx = run->size_bracket_idx_;
  const size_t bracket_size = bracketSizes[idx];
  bool run_was_full = false;
  MutexLock brackets_mu(self, *size_bracket_locks_[idx]);
  ...
  if (LIKELY(run->IsThreadLocal())) {
    // It's a thread-local run. Just mark the thread-local free bit map and return.
    
    ...

    run->AddToThreadLocalFreeList(ptr);
    ...
    // A thread local run will be kept as a thread local even if it's become all free.
    return bracket_size;
  }

  /*
   * 通过 FreeSlot() 释放 Slot，内部操作为将 Slot 清零
   */
  
  // Free the slot in the run.
  run->FreeSlot(ptr);
  auto* non_full_runs = &non_full_runs_[idx];

  /*
   * 如果释放完这一 Slot 后，Run 中所有的 Slot 都是空闲的，即整个 Run 都是空闲的
   * 此时将 Run 释放掉，将管理的内存空间转交给 Free Page Run
   */
  
  if (run->IsAllFree()) {

    /*
     * 从 non_full_runs 中移除记录
     */

    // It has just become completely free. Free the pages of this run.
    std::set<Run*>::iterator pos = non_full_runs->find(run);
    if (pos != non_full_runs->end()) {
      non_full_runs->erase(pos);
      ...
    }

    /*
     * 从 current_runs_ 中移除记录
     */

    if (run == current_runs_[idx]) {
      current_runs_[idx] = dedicated_full_run_;
    }
    ...

    /*
     * 释放内存页
     */

    run->ZeroHeaderAndSlotHeaders();
    {
      MutexLock lock_mu(self, lock_);
      FreePages(self, run, true);
    }
  } else {
    // It is not completely free. If it wasn't the current run or
    // already in the non-full run set (i.e., it was full) insert it
    // into the non-full run set.
    if (run != current_runs_[idx]) {
      auto* full_runs = kIsDebugBuild ? &full_runs_[idx] : nullptr;

      /*
       * 另一种情况是，如果此时 Run 是从所有 Slot 都已使用的状态下释放 Slot，则需将其添加到 non_full_runs 中
       */

      auto pos = non_full_runs->find(run);
      if (pos == non_full_runs->end()) {
        ...
        non_full_runs->insert(run);
        ...
      }
    }
  }
  return bracket_size;
}
```

再看两者均使用到的 `FreePages()`。

``` c++
size_t RosAlloc::FreePages(Thread* self, void* ptr, bool already_zero) {
  lock_.AssertHeld(self);

  /*
   * 获取内存页状态
   */
  
  size_t pm_idx = ToPageMapIndex(ptr);
  ...
  uint8_t pm_type = page_map_[pm_idx];
  ...
  uint8_t pm_part_type;

  /*
   * 转换内存页状态为对应的附属状态，便于后部逻辑处理
   */
  
  switch (pm_type) {
  case kPageMapRun:
    pm_part_type = kPageMapRunPart;
    break;
  case kPageMapLargeObject:
    pm_part_type = kPageMapLargeObjectPart;
    break;
  default:
    LOG(FATAL) << "Unreachable - " << __PRETTY_FUNCTION__ << " : " << "pm_idx=" << pm_idx << ", pm_type="
               << static_cast<int>(pm_type) << ", ptr=" << std::hex
               << reinterpret_cast<intptr_t>(ptr);
    UNREACHABLE();
  }

  /*
   * 遍历相同附属状态的内存页，将其状态设为 kPageMapEmpty
   */

  // Update the page map and count the number of pages.
  size_t num_pages = 1;
  page_map_[pm_idx] = kPageMapEmpty;
  size_t idx = pm_idx + 1;
  size_t end = page_map_size_;
  while (idx < end && page_map_[idx] == pm_part_type) {
    page_map_[idx] = kPageMapEmpty;
    num_pages++;
    idx++;
  }
  const size_t byte_size = num_pages * kPageSize;

  /*
   * 释放内存页
   */

  if (already_zero) {
    ...
  } else if (!DoesReleaseAllPages()) {
    memset(ptr, 0, byte_size);
  }

  /*
   * 找到相邻的 Free Page Run，将释放的内存页与它们进行合并
   *
   * 这一部分便涉及到操作系统中对内存释放的各种情况了，如释放内存的前后是否同样也为空闲内存，这里不再细讲
   */

  ...

  // Turn it into a free run.
  FreePageRun* fpr = reinterpret_cast<FreePageRun*>(ptr);
  ...
  fpr->SetByteSize(this, byte_size);
  ...
  if (!free_page_runs_.empty()) {
    ...
    for (auto it = free_page_runs_.upper_bound(fpr); it != free_page_runs_.end(); ) {
      FreePageRun* h = *it;
      ...
      if (fpr->End(this) == h->Begin()) {
        ...
        it = free_page_runs_.erase(it);
        ...
        fpr->SetByteSize(this, fpr->ByteSize(this) + h->ByteSize(this));
        ...
      } else {
        // Not adjacent. Stop.
        ...
        break;
      }
    }
    // Try to coalesce in the lower address direction.
    for (auto it = free_page_runs_.upper_bound(fpr); it != free_page_runs_.begin(); ) {
      --it;

      FreePageRun* l = *it;
      ...
      if (l->End(this) == fpr->Begin()) {
        ...
        it = free_page_runs_.erase(it);
        ...
        l->SetByteSize(this, l->ByteSize(this) + fpr->ByteSize(this));
        ...
        fpr = l;
      } else {
        // Not adjacent. Stop.
        ...
        break;
      }
    }
  }

  // Insert it.
  
  ...

  fpr->ReleasePages(this);
  ...
  free_page_runs_.insert(fpr);
  
  ...

  return byte_size;
}
```

#### 遍历

见 `InspectAll()`。

``` c++
void RosAlloc::InspectAll(void (*handler)(void* start, void* end, size_t used_bytes, void* callback_arg),
                          void* arg) {
  // Note: no need to use this to release pages as we already do so in FreePages().
  if (handler == nullptr) {
    return;
  }
  MutexLock mu(Thread::Current(), lock_);
  size_t pm_end = page_map_size_;
  size_t i = 0;

  /*
   * 遍历内存页
   */

  while (i < pm_end) {
    uint8_t pm = page_map_[i];
    switch (pm) {

      /*
       * 未使用的内存页跳过遍历
       */

      case kPageMapReleased:
        // Fall-through.
      case kPageMapEmpty: {
        // The start of a free page run.
        FreePageRun* fpr = reinterpret_cast<FreePageRun*>(base_ + i * kPageSize);
        DCHECK(free_page_runs_.find(fpr) != free_page_runs_.end());
        size_t fpr_size = fpr->ByteSize(this);
        DCHECK_ALIGNED(fpr_size, kPageSize);
        void* start = fpr;
        ...
        void* end = reinterpret_cast<uint8_t*>(fpr) + fpr_size;
        handler(start, end, 0, arg);
        size_t num_pages = fpr_size / kPageSize;
        ...
        i += fpr_size / kPageSize;
        DCHECK_LE(i, pm_end);
        break;
      }

      /*
       * 遍历大型对象时，直接将内存页地址作为对象地址调用回调，并跳过对象占用的内存页数
       */

      case kPageMapLargeObject: {
        // The start of a large object.
        size_t num_pages = 1;
        size_t idx = i + 1;
        while (idx < pm_end && page_map_[idx] == kPageMapLargeObjectPart) {
          num_pages++;
          idx++;
        }
        void* start = base_ + i * kPageSize;
        void* end = base_ + (i + num_pages) * kPageSize;
        size_t used_bytes = num_pages * kPageSize;
        handler(start, end, used_bytes, arg);
        ...
        i += num_pages;
        ...
        break;
      }
      case kPageMapLargeObjectPart:
        LOG(FATAL) << "Unreachable - page map type: " << static_cast<int>(pm);
        UNREACHABLE();
      
      /*
       * 对 Run 中的对象进行遍历时，调用 Run.InspectAllSlots() 进行遍历，并跳过 Run 占用的内存页数
       */

      case kPageMapRun: {
        // The start of a run.
        Run* run = reinterpret_cast<Run*>(base_ + i * kPageSize);
        DCHECK_EQ(run->magic_num_, kMagicNum);
        // The dedicated full run doesn't contain any real allocations, don't visit the slots in
        // there.
        run->InspectAllSlots(handler, arg);
        size_t num_pages = numOfPages[run->size_bracket_idx_];
        ...
        i += num_pages;
        DCHECK_LE(i, pm_end);
        break;
      }
      case kPageMapRunPart:
        LOG(FATAL) << "Unreachable - page map type: " << static_cast<int>(pm);
        UNREACHABLE();
      default:
        LOG(FATAL) << "Unreachable - page map type: " << static_cast<int>(pm);
        UNREACHABLE();
    }
  }
}
```

对 Run 中对象的遍历位于 `InspectAllSlots()` 中。

``` c++
void RosAlloc::Run::InspectAllSlots(void (*handler)(void* start, void* end, size_t used_bytes, void* callback_arg),
                                    void* arg) {
  size_t idx = size_bracket_idx_;

  /*
   * 计算 slot 数组的首地址、元素个数与粒度
   */
  
  uint8_t* slot_base = reinterpret_cast<uint8_t*>(this) + headerSizes[idx];
  size_t num_slots = numOfSlots[idx];
  size_t bracket_size = IndexToBracketSize(idx);
  
  ...

  /*
   * 将位于 free_list_ 中的所有空闲 Slot 记录到 is_free 中
   */

  // Free slots are on the free list and the allocated/used slots are not. We traverse the free list
  // to find out and record which slots are free in the is_free array.
  std::unique_ptr<bool[]> is_free(new bool[num_slots]());  // zero initialized
  for (Slot* slot = free_list_.Head(); slot != nullptr; slot = slot->Next()) {
    size_t slot_idx = SlotIndex(slot);
    DCHECK_LT(slot_idx, num_slots);
    is_free[slot_idx] = true;
  }

  /*
   * 将位于 thread_local_free_list_ 中的所有空闲 Slot 记录到 is_free 中
   */
  
  if (IsThreadLocal()) {
    for (Slot* slot = thread_local_free_list_.Head(); slot != nullptr; slot = slot->Next()) {
      size_t slot_idx = SlotIndex(slot);
      DCHECK_LT(slot_idx, num_slots);
      is_free[slot_idx] = true;
    }
  }

  /*
   * 一次性遍历从 free_list_ 与 thread_local_free_list_ 中搜集到的非空闲与空闲 Slot
   */

  for (size_t slot_idx = 0; slot_idx < num_slots; ++slot_idx) {
    uint8_t* slot_addr = slot_base + slot_idx * bracket_size;
    if (!is_free[slot_idx]) {
      handler(slot_addr, slot_addr + bracket_size, bracket_size, arg);
    } else {
      handler(slot_addr, slot_addr + bracket_size, 0, arg);
    }
  }
}
```

## LargeObjectMapSpace

`LargeObjectMapSpace` 作为一种非连续空间的大型对象管理 Space，对于对象的分配方式其实简单到令人惊讶。

每一个对象都通过匿名内存映射分配一块新的内存区域。

以至于这一段实际上都没有分点解析的必要。

静态函数 `Create()` 用于创建 `LargeObjectMapSpace` 对象。

``` c++
LargeObjectMapSpace* LargeObjectMapSpace::Create(const std::string& name) {
  if (Runtime::Current()->IsRunningOnMemoryTool()) {
    ...
  } else {
    return new LargeObjectMapSpace(name);
  }
}

LargeObjectMapSpace::LargeObjectMapSpace(const std::string& name)
    : LargeObjectSpace(name, nullptr, nullptr, "large object map space lock") {}
```

`Alloc()` 直接通过匿名内存映射分配对象，并将分配完成的对象记录到 map 类型的 `large_objects_` 中。

``` c++
mirror::Object* LargeObjectMapSpace::Alloc(Thread* self, size_t num_bytes,
                                           size_t* bytes_allocated, size_t* usable_size,
                                           size_t* bytes_tl_bulk_allocated) {
  std::string error_msg;
  MemMap mem_map = MemMap::MapAnonymous("large object space allocation",
                                        num_bytes,
                                        PROT_READ | PROT_WRITE,
                                        /*low_4gb=*/ true,
                                        &error_msg);
  if (UNLIKELY(!mem_map.IsValid())) {
    LOG(WARNING) << "Large object allocation failed: " << error_msg;
    return nullptr;
  }
  mirror::Object* const obj = reinterpret_cast<mirror::Object*>(mem_map.Begin());
  const size_t allocation_size = mem_map.BaseSize();
  MutexLock mu(self, lock_);
  large_objects_.Put(obj, LargeObject {std::move(mem_map), false /* not zygote */});
  DCHECK(bytes_allocated != nullptr);

  if (begin_ == nullptr || begin_ > reinterpret_cast<uint8_t*>(obj)) {
    begin_ = reinterpret_cast<uint8_t*>(obj);
  }
  end_ = std::max(end_, reinterpret_cast<uint8_t*>(obj) + allocation_size);

  *bytes_allocated = allocation_size;
  if (usable_size != nullptr) {
    *usable_size = allocation_size;
  }
  DCHECK(bytes_tl_bulk_allocated != nullptr);
  *bytes_tl_bulk_allocated = allocation_size;
  num_bytes_allocated_ += allocation_size;
  total_bytes_allocated_ += allocation_size;
  ++num_objects_allocated_;
  ++total_objects_allocated_;
  return obj;
}
```

`Free()` 对匿名内存映射进行释放。

``` c++
size_t LargeObjectMapSpace::Free(Thread* self, mirror::Object* ptr) {
  MutexLock mu(self, lock_);
  auto it = large_objects_.find(ptr);
  if (UNLIKELY(it == large_objects_.end())) {
    ScopedObjectAccess soa(self);
    Runtime::Current()->GetHeap()->DumpSpaces(LOG_STREAM(FATAL_WITHOUT_ABORT));
    LOG(FATAL) << "Attempted to free large object " << ptr << " which was not live";
  }
  const size_t map_size = it->second.mem_map.BaseSize();
  DCHECK_GE(num_bytes_allocated_, map_size);
  size_t allocation_size = map_size;
  num_bytes_allocated_ -= allocation_size;
  --num_objects_allocated_;
  large_objects_.erase(it);
  return allocation_size;
}
```

`Walk()` 通过对 `large_objects_` 进行遍历以遍历所有对象。

``` c++
void LargeObjectMapSpace::Walk(DlMallocSpace::WalkCallback callback, void* arg) {
  MutexLock mu(Thread::Current(), lock_);
  for (auto& pair : large_objects_) {
    MemMap* mem_map = &pair.second.mem_map;
    callback(mem_map->Begin(), mem_map->End(), mem_map->Size(), arg);
    callback(nullptr, nullptr, 0, arg);
  }
}
```