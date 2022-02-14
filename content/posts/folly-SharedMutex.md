---
title: "Folly SharedMutex实现"
date: 2022-02-12T20:53:09+08:00
tags: [folly,mutex]
categories: [devlop]
---

在多线程并发读写开发中，尤其是读多写少的情况，使用读写锁代替普通mutex一般可以提升读并发度。 在多个线程频繁获取和释放读锁时，read lock count计数修改会反复触发cpu cacheline同步，导致实际性能可能达不到预期。[folly SharedMutex](https://github.com/facebook/folly/blob/main/folly/SharedMutex.h) 通过将reader信息存储在全局静态区域，来减少cacheline同步。


## class structure 

### 类定义
```cpp
using Futex = std::atomic<std::uint32_t>;

class SharedMutexImpl {
    // 32 bits of state
    Futex state_{};
}
```
SharedMutexImpl 仅使用32位的atomic来存储读写锁内部状态
![](/img/shared_mutex/shared_mutex_state.png)
其中一些常用状态含义为（详见[代码注释](https://github.com/facebook/folly/blob/main/folly/SharedMutex.h#L826))
* kIncrHasS = 1 << 11, kHasS = ~(kIncrHasS - 1); 第11-31位表示shared lock计数, 每次增减 kIncrHasS。
* kHasE = 1 << 7; exlusive排它标志位
* kBegunE = 1 << 6;  开始获取写锁
* kHasU = 1 << 5;  upgrade lock
* kMayDefer = 1 << 9; 是否有读锁信息存储在全局静态区
### Global Shared readers
 
```cpp
constexpr uint32_t kMaxDeferredReadersAllocated = 256 * 2;
static constexpr uint32_t kDeferredSeparationFactor = 4;
typedef Atom<uintptr_t> DeferredReaderSlot;
alignas(hardware_destructive_interference_size) static DeferredReaderSlot
    deferredReaders[shared_mutex_detail::kMaxDeferredReadersAllocated 
      * kDeferredSeparationFactor];
```
`deferredReaders`定义为2048个cacheline对齐的`DeferredReaderSlot`，uintptr_t保存SharedMutex实例唯一的token，标记当前slot关联的SharedMutex被某个线程持有reader lock。

通过将reader lock状态保存在`Global Shared readers`中， 避免获取和释放读锁时反复修改`state_`变量。 但这样也带来一些问题：
1. 申请写锁时会比一般的RWMutex慢，因为需要先保证所有相关的DeferredReaderSlot释放，再去检查和修改`state_`。
2. 全局静态的slot数量固定（2048个)， 在读并发较高或者SharedMutex实例数足够多时，slot会不够用。这时`SharedMutex`退化成普通的RWMutex，读锁申请释放还是对`state_`加减`kIncrHasS`。
![](https://blog.the-pans.com/content/images/2020/07/Screen-Shot-2020-07-01-at-2.47.35-PM.png)
## 实现 TODO
### lock_shared
### unlock_shared
### lock
### unlock
