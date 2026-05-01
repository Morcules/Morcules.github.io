---
title: "Reducing Atomic Overhead in a Multithreaded C Allocator by 50%"
date: 2026-05-01
categories: [Networking, C]
tags: [performance, multi-threading, memory]
---

When working with multithreaded C code, atomic operations often look cheap, until they become your biggest bottleneck.
In my case, a small change in how threads acquired memory reduced CPU usage by 40-50%, cutting execution from roughly 2 billion cycles to 1 billion cycles when sending 50MB of data.

## Performance Changes
### Before:
- CPU Cycles : 2,273,393,860
- User Cpu Time : 0.58s
- System Cpu Time : 0.23s
### After:
- CPU Cycles : 1,005,063,001
- User Cpu Time : 0.24s
- System Cpu Time : 0.17s

## The Problem: Multithreaded Allocator
I was building a networking library where multiple threads needed fast access to pre-allocated memory.
To handle this, I implemented a custom allocator:
- multiple allocator stacks
- each stack protected by an atomic spinlock
- threads scan stacks until they find an available one
- stack count is dynamically scaled

Allocator structure:
```
struct SwiftNetMemoryAllocator {
    SwiftNetMemoryAllocatorStack* stacks\[64\];
};
```
Each stack had:
```
_Atomic bool lock
```

threads would attempt to acquire it using atomic_compare_exchange_strong

## Why This Was Slow
At first glance, this design seems simple, safe and common.
But the issue was how fast threads searched for a free stack.

If there were 10 stacks:
- a thread might perform up to N failed atomic_compare_exchange_strong attempts 
- each failed attempt = wasted CPU cycles

### That means:
- Decreasing performance as stacks scale

Even though each atomic is fast on modern machines, doing many of them in a loop is not.
The real problem wasn't locking, it was having multiple locks.

## The Optimization: Atomic Bitmap
Instead of storing a lock per stack, I introduced a bitmap:
- each bit represents whether a stack is in use or free
- stored in a single atomic integer (uint64_t)

### Before:
N stacks -> up to N atomic operations
### After:
1 bitmap -> 1 atomic operation (when uncontended)

Using a bitmap alone isn’t enough. You need a fast way to find a free slot.
The most efficient solution is: ctz (count trailing zeros)

## Example:
```
if (bitmap == 0) {
  // No free slots. Create a new stack.
}

// __builtin_ctzll is undefined if input is 0
uint32_t index = __builtin_ctzll(bitmap);
```

This lets you:
find the first available stack in a few CPU cycles

## Outro
This simple change might seem too small to make a huge difference, but in my case the library was allocating around 50,000 items.
Between allocating, freeing, cleaning up, and handling spinlocks on a stack, the overhead adds up quickly.
If we assume each spin-lock costs around ~500 CPU cycles, that alone results in roughly 100 million wasted CPU cycles.

But the real impact is even larger due to cache inefficiency.
When the CPU reads or writes to RAM, it stores recently accessed data in cache lines. If you're repeatedly touching or spinning on 10 different structs, you're effectively polluting the cache with unrelated data. This increases cache misses and can push useful data out of L1/L2 into L3 or even force frequent trips to main memory.
Without effective caching, memory access can become 10×–100× slower depending on cache level, because each miss forces the CPU to wait on RAM instead of fast on-chip cache.
