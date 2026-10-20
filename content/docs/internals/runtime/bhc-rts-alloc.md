+++
title = "bhc-rts-alloc"
description = "Memory allocation"
weight = 2
+++

# bhc-rts-alloc

Memory allocation primitives for the Basel Haskell Compiler runtime.

## Overview

This crate is the bottom layer of the RTS: the other four runtime crates depend on it. It defines the shared allocation vocabulary (alignments, memory regions, errors, statistics), a pinned allocator for memory that must not move, and a static allocator for the no-GC `Embedded` profile.

**Status:** the allocators here are implemented and unit-tested, and the `bhc_pinned_*` C functions are present in `libbhc_rts.a`. Compiled Haskell programs do not use them yet: generated code allocates every heap object with `bhc_alloc` in `bhc-rts`, which uses the system allocator and never frees (see [bhc-rts](@/docs/internals/runtime/bhc-rts.md)).

## Core Types

| Item | Purpose |
|------|---------|
| `Alignment` | `Default` (8), `CacheLine` (64), `Simd128` (16), `Simd256` (32), `Simd512` (64), `Page` (4096) |
| `MemoryRegion` | `HotArena`, `PinnedHeap`, `GeneralHeap`, `DeviceMemory(DeviceMemoryKind)` |
| `DeviceMemoryKind` | `Cuda(u32)`, `Rocm(u32)` (device index) |
| `BlockMeta` | Size, alignment, region and pinned flag of a block |
| `AllocError` | `OutOfMemory`, `InvalidLayout`, `ArenaExhausted`, `AlignmentError` |
| `Allocator` trait | `unsafe fn allocate(&self, Layout)` / `deallocate` |
| `RawBlock` | A raw pointer with its layout and region, for allocator implementations |
| `Zeroable` | Marker trait for types that are valid when all-zero |
| `AllocStats` | Bytes and counts of allocations, deallocations and failures |
| `align_up`, `is_aligned` | Alignment helpers |

## Pinned Heap

`PinnedAllocator` implements `Allocator` over the system allocator, for memory that foreign code or devices hold pointers into. `PinnedBuffer<T>` is an owned, typed pinned buffer (`new`, `new_zeroed`, `new_aligned`, `as_ptr`, `copy_from_slice`), freed on drop.

The C interface:

```c
void *bhc_pinned_alloc(size_t size, size_t alignment);
void *bhc_pinned_alloc_zeroed(size_t size, size_t alignment);
void  bhc_pinned_free(void *ptr, size_t size, size_t alignment);
void *bhc_pinned_realloc(void *ptr, size_t old_size, size_t new_size, size_t alignment);
```

## Static Allocation (`static_alloc`)

For the `Embedded` profile, where dynamic allocation is not available:

- `StaticAllocator`: bump allocation inside a fixed buffer handed over once with `unsafe fn init(&self, &'static mut [u8])` (or `init_raw`). Memory is never freed. `alloc`, `alloc_zeroed`, `alloc_value`, `alloc_array` and `alloc_array_zeroed` return errors when the buffer is full; `capacity`, `used`, `remaining` and `stats` report usage. It is `const`-constructible, so it can live in a `static`.
- `BoundedAllocator<const MAX_SIZE: usize>`: a `StaticAllocator` that also rejects any single allocation larger than `MAX_SIZE`.
- `EmbeddedConfig`: heap size, largest single allocation, panic-on-failure and zero-memory settings, with presets `minimal()` (8 KB), `standard()` (64 KB) and `large()` (256 KB).
- `NoGcGuard`: a marker value for code regions where GC must not run.

```rust
use bhc_rts_alloc::static_alloc::StaticAllocator;
use std::alloc::Layout;

static HEAP: StaticAllocator = StaticAllocator::new();
static mut BUFFER: [u8; 64 * 1024] = [0; 64 * 1024];

unsafe { HEAP.init(&mut BUFFER) };
let ptr = HEAP.alloc(Layout::new::<[u32; 100]>())?;
```
