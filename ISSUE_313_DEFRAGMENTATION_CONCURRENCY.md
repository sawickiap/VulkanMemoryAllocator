# Defragmentation concurrency investigation - issue #313

## Summary

[VMA issue #313](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/issues/313)
reports intermittent memory corruption when defragmentation and ordinary allocator
operations run on different threads. The investigation found three genuine
synchronization problems in the defragmentation implementation:

1. Creating and destroying a defragmentation context changed and sorted a
   `VmaBlockVector` without taking its mutex.
2. Completing a `COPY` move changed TLSF allocation metadata through
   `SwapBlockAllocation` without taking the block-vector mutex.
3. `EXTENSIVE` pass completion sampled the block count before and after
   `VmaBlockVector::Free`, then used the difference without holding one lock
   across the operation. A concurrent allocation could make the second count
   larger than the first, causing unsigned underflow. The same code also read
   `VmaBlockVector::m_Blocks` and block metadata without the vector mutex.

The first two problems had already been fixed in Git history. This work adds a
regression test and fixes the third problem.

The ticket's attached standalone reproducer has additional application-side
data races and lifetime violations. In particular, it may destroy an allocation
while that allocation is present in the active `pMoves` array. VMA documents
that every allocation returned in `pMoves` must remain alive until
`vmaEndDefragmentationPass`. Those test-side problems must be separated from the
library synchronization problems above.

## How the problem was observed and reported

The original report from 2023 described:

- an x86-64 application using an AMD GPU;
- one thread performing defragmentation;
- another thread making normal allocation and deallocation calls;
- occasional corruption of the first bytes of a `VmaAllocation` referenced by
  a defragmentation move;
- an application-level mutex around the complete defragmentation operation as
  a workaround.

Later discussion added more evidence:

- ThreadSanitizer output showing concurrent access to block sorting state,
  block metadata, and the TLSF free list;
- a production failure where a move source appeared to have a null block;
- a standalone Windows reproducer using many allocation threads and one
  defragmentation thread;
- a patch proposing locks around defragmentation context construction and
  destruction;
- a later observation that `SwapBlockAllocation` updates allocation user data
  stored in TLSF metadata, so it must use the same block-vector mutex as normal
  allocation and free operations.

The issue remained open after the first fixes because failures became less
frequent but did not disappear in all reported workloads.

## Investigation

### Ticket and attached files

The full issue discussion and its attachments were inspected. The attachments
included:

- `vma_race_test.zip`, a standalone Vulkan/VMA workload;
- a proposed `vk_mem_alloc.h` patch;
- a ThreadSanitizer log;
- an image of the proposed context-lifetime locking change.

The standalone workload was built with both its bundled VMA header and the
current repository header. A 30-second run did not fail on this machine, which
is consistent with the intermittent nature of the report.

Two problems were found in that standalone workload:

1. `allocations.size()` is read concurrently with modifications to the same
   vector.
2. Allocation threads may destroy any allocation, including one returned in an
   active defragmentation move. The ThreadSanitizer log's final use-after-free
   reports match this behavior.

It also ignores the result returned by `vmaEndDefragmentationPass` and then
checks an older `VkResult`, which can cause many unnecessary passes.

These findings do not invalidate the genuine library reports earlier in the
same log. They explain why the supplied program may still fail after the
library's metadata locks are corrected.

### API contract

The public documentation says that:

- calls operating on the allocator can be made from multiple threads when
  internal synchronization is enabled;
- access to the same `VmaAllocation` object must be synchronized externally;
- allocations returned in an active defragmentation `pMoves` array must not be
  destroyed before the pass ends.

The new regression test follows these rules. Worker-owned allocations are
drained before every call to `vmaBeginDefragmentationPass`. Workers resume only
after the move list is complete, so they can mutate the same pool concurrently
without owning or destroying any allocation referenced by the active moves.

### Git history

Relevant changes found in Git history include:

- `7b9c21f`: added block locks while beginning a defragmentation pass;
- `31910c8` and `1022be6`: addressed defragmentation and mapping interaction;
- `05973d8`: moved allocation-object destruction under the block-vector free
  lock;
- [`fc48eb1`](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/commit/fc48eb1):
  locked defragmentation context construction and destruction while changing
  incremental sorting state;
- [`c3c7a2c`](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/commit/c3c7a2c):
  locked `SwapBlockAllocation` while completing a `COPY` move.

The last two commits were merged upstream through
[PR #529](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/pull/529)
and are present in VMA 3.4.0 and the current `sawickiap/master`.

### Remaining problem in the current code

`VmaDefragmentationContext_T::DefragmentPassEnd` still used this pattern:

1. take a read lock and record `prevCount`;
2. release the lock;
3. call `VmaBlockVector::Free`, which takes its own write lock;
4. take another read lock and record `currentCount`;
5. use `prevCount - currentCount`.

Other threads can allocate or free blocks between these steps. Therefore the
difference does not describe what the defragmentation free did. If
`currentCount > prevCount`, unsigned subtraction underflows. This affects
defragmentation statistics and `StateExtensive::firstFreeBlock`.

The same `EXTENSIVE` state update then called `vector->GetBlock(...)` and
`IsEmpty()` without holding the block-vector mutex, even though `GetBlock` is
explicitly documented internally as valid only while that mutex is locked.

On the local GPU, `bufferImageGranularity` is 1, so `EXTENSIVE` normally falls
back to the `FULL` algorithm. Testing this code path required using VMA's
debug-only minimum granularity override.

## Solution

### Block-free result

The internal `VmaBlockVector::Free` function now returns the size of the Vulkan
memory block that it released, or zero if no block was released.

This result is produced while the block-vector write lock is held. Pass
completion no longer infers its own result by comparing two snapshots that may
include unrelated operations from other threads.

`DefragmentPassEnd` now uses the returned values to update:

- `deviceMemoryBlocksFreed`;
- `bytesFreed`;
- the number of released blocks used by `StateExtensive`.

This also makes the reported statistics describe blocks released by the
defragmentation operation rather than unrelated concurrent changes.

### `EXTENSIVE` state synchronization

The `EXTENSIVE` state update now:

- takes the block-vector read lock before reading the vector or block metadata;
- clamps `firstFreeBlock` to the current block count before indexing;
- avoids unsigned block-count subtraction;
- normalizes `firstFreeBlock` again at the start of
  `ComputeDefragmentation_Extensive`, which already runs under the
  block-vector write lock.

The second normalization is necessary because other threads may add or remove
blocks after one pass ends but before the next pass begins.

### Regression test

`TestDefragmentationConcurrentAllocations` was added to `src/Tests.cpp`.

The test:

- creates a custom TLSF pool backed by local Vulkan device memory;
- runs eight worker threads that allocate and free memory from the pool;
- repeatedly creates and destroys defragmentation contexts while workers are
  active, covering the context sorting locks;
- creates a fragmented set of stable allocations;
- tests both `FAST` and `EXTENSIVE` defragmentation;
- resumes allocation workers only after each move list is generated;
- exercises `COPY` and `DESTROY` pass outcomes;
- verifies that every move source belongs to the stable allocation set;
- verifies block-free statistics for the fixed-size custom pool.

The sample CMake configuration now exposes
`VMA_DEBUG_MIN_BUFFER_IMAGE_GRANULARITY`. Its default is 1. Setting it to 256
forces the real `EXTENSIVE` state machine to run on hardware whose native
granularity is 1.

## Validation

Local system:

- GPU: AMD Radeon RX 9060 XT;
- driver: AMD proprietary driver 25.20.42.14;
- Vulkan API: 1.4.329 reported by `vulkaninfo`;
- physical-device `bufferImageGranularity`: 1 byte;
- build: Visual Studio 2022, `RelWithDebInfo`.

The focused test was built against the previous header at commit `5ede0ae`
(which already contains `fc48eb1` and `c3c7a2c`, but not the solution in this
change), with granularity forced to 256. It failed during `EXTENSIVE`
processing:

- one sequence failed on comparison run 2;
- a second sequence failed on comparison run 6;
- the test stopped at `TEST(success)` after approximately 177,000 to 181,000
  concurrent allocator operations.

The same focused test with the updated header completed 100 consecutive runs.

Two complete sample test-suite runs also passed:

| Configuration | Concurrent operations in the new test | Passes | Moves | Result |
|---|---:|---:|---:|---|
| Default granularity (1) | 53,383 | 12 | 516 | `Done, all PASSED.` |
| Forced granularity (256) | 169,811 | 39 | 828 | `Done, all PASSED.` |

The full-run logs are available in the ignored `build` directory:

- `build/issue313-final-suite.stdout.log`;
- `build/issue313-final-suite.stderr.log`;
- `build/issue313-final-forced-suite.stdout.log`;
- `build/issue313-final-forced-suite.stderr.log`.

GPU capability reports generated during the investigation are also available:

- `build/vulkaninfo.log`;
- `build/vulkaninfo-full.log`.

### Repeating the forced-granularity test

From a Visual Studio developer environment:

```powershell
cmake -S . -B build/issue313-forced `
  -DVMA_ENABLE_INSTALL=ON `
  -DVMA_BUILD_SAMPLES=ON `
  -DVMA_DEBUG_MIN_BUFFER_IMAGE_GRANULARITY=256
cmake --build build/issue313-forced --config RelWithDebInfo --target VmaSample -- /m

Set-Location bin
../build/issue313-forced/src/RelWithDebInfo/VmaSample.exe -t
```

The sample must run with `bin` as its working directory because it loads shader
artifacts using relative paths.

## Limitations and follow-up

ThreadSanitizer was not available in the local Windows environment, and WSL was
not installed. The attached ThreadSanitizer reports were therefore analyzed
rather than regenerated locally.

The broad ticket contains both library defects and invalid allocation lifetime
in attached application code. The new test covers allocator synchronization
while respecting the documented move lifetime. Any remaining report should
first verify that allocations in the active `pMoves` array cannot be destroyed
or otherwise accessed concurrently by the application.
