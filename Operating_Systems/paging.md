# Paging

Legend: plain bullets are the main narrative. *Italic indented lines* are skippable explanations. `> Aside:` blocks are side facts. **[+]** remarks I added

**Source:** [Virtual Memory from First Principles](https://blog.codingconfessions.com/p/virtual-memory), Abhinav Upadhyay, Coding Confessions.

---

# Conceptual map: where paging touches the rest of the OS

Paging isn't a standalone topic. Three other OS ideas lean on it. Keep these in mind as you read.

1. **Process creation (`fork`).**
   - `fork` is cheap because it copies page tables, not memory. Everything about it rests on page tables and permission bits.
   - Pointer: copy-on-write in section 5.
2. **Process states (scheduling).**
   - The states: **running, ready, blocked**, plus the **suspended** variants (**ready-suspended, blocked-suspended**). Suspended means the process has been moved out to swap.
   - Pointer: swap and where pages live, in sections 7 and 8.
   - **[+]** *This is the textbook model, where a whole process is swapped out. Modern Linux swaps individual pages instead, so a process is rarely "suspended" as a whole. The idea of a process whose memory is on disk is the same.*
3. **Shared memory, message passing, synchronization (IPC).**
   - Shared memory is two processes' page tables pointing at the same physical frames. Permission bits say who can read or write them.
   - So the **MMU** is what manages *who can see what*.
   - **[+]** *One distinction: the MMU gives you the sharing and the permissions. It does not give you synchronization. Shared memory is one IPC style next to message passing, and processes sharing frames still need locks or semaphores to coordinate.*
   - Pointer: sections 3 and 4.

---

#Section 0: Questions

- Why introduce virtual memeory?
- Who manages the region bw heap and stack, and used for what?A
- For address translation, what does the HW do vs what is in the SW? 
- Why herirachial tables?
- Whta does paging aloen ( meaning without demand paging) solve? what problem does it solve?
-




# Section 1: The Idea

## 1. Why introduce virtual memory?

- **Illusion of abundant memory.** A process can address more memory than the RAM that exists.
- **Process isolation and memory protection.** Each process gets its own private view of memory, and the hardware enforces its limits.
- **Multiprogramming.** Many processes can be resident at once, each believing it owns the machine.
- Without this layer, every process would need to know about every other: where they live, what they use, how to stay out of their way.
  - *The picture: processes on one side, resources on the other. Without an intermediary you get a connected polygon, every process wired to every other. With the OS in the middle you get a star: each process talks only to the OS, and the OS talks to the resources.*

**Question:** how is process isolation different from memory protection?

- **[+]** Isolation is the property: process A cannot even name process B's memory. Each has its own address space.
- **[+]** Protection is the mechanism: permission checks (read/write/execute, user/kernel) on each access. It enforces isolation, and it also guards things isolation alone doesn't, like the kernel's memory, or code pages inside your own process that should be read-only.

---

## 2. The virtual address space

- Every process sees one contiguous address space, laid out as **code, data, heap, stack**.
- This is a form of segmentation, and it works in sync with paging: the segments are what we cut into pages.
  - *Code is the program's instructions. Data holds globals. Heap grows up for dynamic allocation. Stack grows down for function calls.*

> **Aside: what lives between the heap and the stack?**
> The gap is where **memory-mapped regions** (`mmap`) get placed. They're used for large allocations and for things like thread stacks. **[+]** Shared libraries are mapped here too. glibc's `malloc` switches from the heap to `mmap` above a size threshold (128 KB by default).

---

## 3. Address translation

- Goal: turn a virtual address into a physical address.
- We build it up in steps:
  1. Cut memory into fixed-size **pages** (virtual) and **frames** (physical).
  2. A flat table mapping every page to a frame would be enormous.
  3. So we make the table **hierarchical**: a tree of small tables. Only the branches in use exist.
- Why hierarchy helps:
  - It shrinks the page table, since unused regions cost nothing.
  - It makes lookup cache-friendly, since each table is small and fits a page.
  - *Note on wording: the thing that shrinks is the page table, not the page itself.* **[+]**

**Question:** where do the page directories reside?

- In physical memory, like any other kernel data.
- The top-level table (the global directory) is found through a register. On x86 that's **CR3**, which holds the *physical* address of the top table.
- Each entry holds a **physical address**: of the next-level table, or, at the last level, of the final frame.
- So a translation is a chain: CR3 → top table → next table → … → frame.

> **Aside: how is a page-table walk different from a normal load/store?**
> A normal access is checked for permissions. The accesses made *during* the walk are not. The tables are kernel-built structures, addressed physically, so the hardware trusts them. **[+]** One precision: it's the MMU that performs the walk, not kernel code running at that moment. The kernel builds and owns the tables. The permission bits inside the final entry are then checked against your actual access.

---

## 4. The role of hardware

- Paging is a **software idea supported by hardware**. The OS defines the policy and builds the tables. The hardware does the translation on every access.
- The **MMU** is hardware. It performs the translation.
- The **TLB** is a cache inside the MMU. It remembers recent virtual-to-physical translations, so most accesses skip the walk entirely.
  - *Without it, every memory access would cost several extra memory accesses (one per level).* **[+]**

Have a diagram in the notebook explaning waht happens once we get a request
- MMU stores vir addr in register. 
- finds in TLB, or does a Page walk(deos involve Reading PTE from DRAM)
- Gets the frame, sends read request to CPU with a tag:data ( not MMU)
- the value from returned form DRAM goes to cache, and then CPU reg

All in all, MMU HW handles read adn write form DRAM as well. 

**Question:** how do Intel and the other MMU designers work in sync with Linux?

- **[+]** The architecture manual is the contract. The hardware vendor specifies exactly how the MMU, CR3, and the table format behave. The OS is written against that spec. They don't need to coordinate per release.
- **[+]** In practice vendors also contribute patches to Linux directly for new hardware features.

---

## Where we are

- Paging, on its own, **breaks fragmentation**. A program no longer needs one contiguous block of RAM. Its pages can sit in different frames, anywhere.
- What we have **not** introduced yet: the program doesn't need to be fully in memory to run.

---

# Section 2: Implementation

Paging is a mechanism: memory in fixed-size units. On top of it sit three policy questions:

- **When to fetch?** Demand paging or pre-paging.
- **Where to place?** Any free frame. Frames are all the same size, so this is trivial.
- **What to evict?** Page replacement.

This section covers *when to fetch* and the faults that come with it. Eviction is next.

---

## 5. Demand paging

- Don't allocate or load everything at once. Allocate as demand grows.
- **Code:** touching one function doesn't load the whole program. Huge libraries are the big win, since only the pages you actually touch ever come in.
- **Memory:** `malloc(10 GB)` doesn't hand you 10 GB of RAM. You get address space. Frames arrive as you use it.
- It works because programs rarely touch all of what they own. Oversubscription becomes cheap.
  - *Paging alone fixes fragmentation. Demand paging is the policy that makes memory go further. Same mechanism, different question: when to load.*

**Lazy allocation: even the first frame**

- A fresh page starts as a page table entry with **no physical frame**. Present bit = 0.
- Nothing happens until you first touch it. Then the MMU raises a **page fault**, a CPU exception, because the present bit is 0.
- The kernel's handler works out that this page was never allocated, allocates a frame, fills in the entry, and the instruction retries.
  - **[+]** *The handler first checks that the address is legal, meaning it falls inside a region the process actually owns. If not, that's a segfault. If yes, it's a demand-allocation. Anonymous pages get a zero-filled frame.*

> **Aside: copy-on-write for `fork` is the same idea.**
> `fork` doesn't copy memory. The child gets its own page table entries pointing to the **same physical frames**, and those pages are marked **read-only**. Reading is free and shared. The first write triggers a fault, and only then is that page copied.
> **[+]** Correction on detail: both parent and child are marked read-only. Whoever writes first gets the copy. If the other is left as the sole owner, its page just becomes writable again, with no copy needed.

---

## 6. Page faults

- Two kinds: **minor** and **major**.
- **Minor:** the kernel can fix it on its own, with no disk I/O. Demand-allocation above is a minor fault.
- **Major:** fixing it requires reading from disk.
- To see when a fault is major, we first need to know what kinds of memory exist.

---

## 7. Kinds of memory

- **Anonymous:** not backed by a file. Heap and stack. The ordinary memory we allocate.
- **File-backed:** a mapping of a file on disk. Includes program code and `mmap`'d files.
  - **Dirty vs clean:** a clean page matches the disk. A dirty page has been modified and not yet written back.
  - **The page cache:** file-backed pages in RAM live in the **page cache**, where recently accessed files are kept. It is not a reserved region. It uses whatever RAM is free and shrinks when something else needs it.
  - **Cheapest to evict:** these are the easiest pages to reclaim, easier than anonymous memory. A clean page is just dropped, since the file is its backup. Anonymous memory has no file, so it must be written to swap first.
    - **[+]** *The page cache also serves ordinary `read`/`write` calls, not only `mmap`. And "first to go" is a tendency, not a rule: Linux weighs file pages against anonymous ones (see `swappiness`).*
- **Kernel pages:** memory the kernel itself uses.
- **DMA pages:** memory devices read and write directly.
- **Shared pages:** physical pages mapped by more than one process. Later.

*Why this matters beyond faults: it decides what can be evicted and at what cost.* **[+]** A clean file page can just be dropped, since the file is the backup. A dirty one must be written out first. Anonymous pages have no file, so they need swap. Kernel and DMA pages are generally pinned, because the hardware addresses them physically.

---

## 8. Major faults: where can a page be?

- Present bit = 0 means "not in RAM right now." That has more than one cause.
  - **Never allocated:** the entry is empty. That's the minor fault from section 5.
  - **In swap:** the entry holds a disk location. The page was pushed out earlier. **Major fault**: fetch from disk.
  - **In a file:** a file-backed page not yet read in. **Major fault**: read from the file.
- So a page lives in one of three places: **RAM, swap, or a file on disk.**
- If it's in RAM and mapped, there's no fault.

> **[+] Aside: two corrections on file-backed faults.**
> A file-backed fault is not *always* major. If another process already read that file page into RAM (the page cache), the fault just maps it in, which is minor. Also, once a page is mapped, later reads of it don't fault again. You fault once per page, not once per read, until it's evicted.

---

## Where we are

- Pages are fetched lazily, on first touch.
- A fault means "not in RAM." Minor if the kernel can fix it alone, major if it needs the disk.
- RAM is finite, though. We haven't said what happens when it runs out.

**Next: page reclaim and replacement policies. What to evict.**

---

# Left out for clarity

Topics from the source that we skipped to keep the main line clean. One line each, for later.

- **Memory-mapped regions in full:** the gap between heap and stack holds file mappings, shared libraries and large allocations, with a file's pages appearing directly in the address space.
- **`mmap` vs `read()`:** `read()` copies file data from the page cache into a user buffer. `mmap` maps the page cache pages directly, which avoids the copy but pays page faults on first touch.
  - **[+]** *Which wins depends on the access pattern. `mmap` tends to suit large files with random or repeated access. `read()` tends to suit sequential streaming.*
- **`MAP_SHARED` vs `MAP_PRIVATE`:** shared writes are visible to other processes and reach the file. Private writes are copy-on-write and never do.
- **`msync` and `fsync`:** `msync` pushes dirty mapped pages out to the file. `fsync` forces a file's dirty pages onto the disk, which is the durability guarantee.
- **Direct I/O (`O_DIRECT`):** bypasses the page cache and moves data straight between user buffer and disk. Databases use it because they run their own cache.
- **`tmpfs`:** a file system that lives in RAM. It has a file interface but no file on disk behind it, so it sits between file-backed and anonymous memory.
  - **[+]** *Its pages can be swapped out. `/dev/shm` is a common place for shared memory.*
- **The OOM killer:** when reclaim can't free enough memory, the kernel picks a process to kill. It belongs right after page replacement.
