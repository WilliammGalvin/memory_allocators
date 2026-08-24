# Memory allocators

## Pool allocator

**How it works:** Pre-allocates a contiguous block of memory divided into fixed-size chunks. All free chunks are linked into a singly-linked free-list (the pointer is stored in the unused chunk itself, so no extra memory overhead). Allocation pops the head of the free-list; deallocation pushes the chunk back onto the head.

**Use case:** Repeated alloc/free of same-sized objects — order objects, network packet structs, game entities, connection handles.

**Logic:**

- `allocate()`: return `free_list_head`, advance `free_list_head = *(void**)free_list_head`
- `deallocate(ptr)`: `*(void**)ptr = free_list_head; free_list_head = ptr`
- No coalescing needed since all blocks are the same size — O(1) always.

---

## Arena allocator

**How it works:** A large contiguous buffer with a single "current offset" pointer. Each allocation just returns the current offset and bumps it forward by the requested (aligned) size. No per-object deallocation — the whole arena is reset or destroyed at once.

**Use case:** Per-frame, per-request, or per-task scratch memory where everything shares the same lifetime (e.g., parsing a message, rendering a frame, building a temporary AST).

**Logic:**

- `allocate(n)`: `align(offset); ptr = base + offset; offset += n; return ptr`
- `reset()`: `offset = 0`
- No `deallocate` per object — bulk-free only. Cheapest possible allocator (no metadata, no branching on free).

---

## Stack allocator

**How it works:** Same bump-pointer mechanism as an arena, but exposes a "marker" (the current offset) that can be saved and later restored, giving LIFO deallocation. Freeing rewinds the offset to a previously saved marker, freeing everything allocated after it.

**Use case:** Nested/scoped temporary allocations with strict hierarchical lifetimes — recursive algorithm scratch space, temporary buffers within a function scope that unwind in reverse order.

**Logic:**

- `get_marker()`: return current `offset`
- `allocate(n)`: same as arena bump
- `free_to_marker(m)`: `offset = m` (optionally run destructors for objects allocated after `m`, tracked separately if needed)

---

## Slab allocator

**How it works:** Generalizes the pool allocator — maintains multiple pools ("slabs"), each dedicated to a specific object type or size class. Each slab is itself a set of pages, each page divided into equal-sized slots with its own free-list. Slabs can also pre-construct/cache objects to skip constructor/destructor overhead on reuse.

**Use case:** Systems with many distinct fixed-size object types allocated/freed at high frequency — kernel object allocation (Linux/Solaris slab allocator), engine subsystems with several hot object types.

**Logic:**

- Maintain a `slab` per type: `{ page_list, free_list, object_size }`
- `allocate<T>()`: routes to the slab for `T`, then behaves like a pool allocator
- When a slab's free-list is empty, allocate a new page from a backing allocator and carve it into free slots
- Optionally keeps objects "constructed" between reuse cycles (cache-style slab)

---

## Buddy allocator

**How it works:** Manages memory as blocks whose sizes are powers of two. On allocation, it finds the smallest available block ≥ requested size; if that block is larger than needed, it's recursively split in half ("buddies") until it fits. On free, the block is merged back with its buddy if the buddy is also free, recursively coalescing back up.

**Use case:** Variable-size allocations where you want bounded fragmentation and fast coalescing without full general-purpose allocator overhead — used in OS page allocators, GPU memory managers.

**Logic:**

- Maintain free-lists indexed by size class (order `k` = block size `2^k`)
- `allocate(n)`: round up to next power of two, find smallest non-empty free-list ≥ that order, split down as needed, push unused half-blocks to lower-order free-lists
- `deallocate(ptr)`: compute buddy address via `addr XOR block_size`, check if buddy is free; if so, merge and repeat one order up; else push to free-list at current order

---

## Free-list allocator

**How it works:** General-purpose allocator for variable-size blocks. Maintains a list (or segregated lists by size bucket) of free blocks, each with a header storing size and next-pointer. Allocation searches the list for a suitable block (first-fit, best-fit, or segregated-fit) and may split it; deallocation adds the block back and attempts to coalesce with adjacent free neighbors.

**Use case:** General allocation with unpredictable, varying sizes and lifetimes — useful as your "fallback"/comparison allocator against the specialized ones above, or for subsystems where allocation patterns aren't uniform.

**Logic:**

- Each free block header: `{ size, next }`
- `allocate(n)`: walk free-list, pick first/best block ≥ `n`, split off remainder back into free-list if large enough, return the rest
- `deallocate(ptr)`: insert block back into free-list, check adjacent memory addresses for free neighbors and merge (coalesce) to reduce fragmentation
- Segregated variant: bucket free-lists by size class (e.g., powers of two) to make search O(1)-ish, at the cost of some internal fragmentation

---

## Ring buffer

**How it works:** Fixed-size contiguous buffer treated as circular — a write/head pointer and a read/tail pointer both wrap around modulo the buffer size. Allocation advances the head; "freeing" (in the allocator sense) advances the tail once older allocations are no longer needed, reclaiming that space for reuse.

**Use case:** Producer-consumer pipelines with predictable, sequential reuse — network I/O buffers, lock-free SPSC/MPSC queues, logging buffers, audio/video frame buffers.

**Logic:**

- `allocate(n)`: if `head + n` would overrun `tail` (wrapping), fail or block; else claim `[head, head+n)`, advance `head = (head + n) % capacity`
- `free_oldest(n)`: advance `tail = (tail + n) % capacity` once consumer is done with that region
- Lock-free variants use atomic CAS on `head`/`tail` for concurrent producer/consumer access without mutexes
