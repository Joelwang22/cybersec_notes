# Operating Systems Study Notes

<a id="contents"></a>
# Contents

- <a id="toc-section-1-foundations-the-machine-and-the-operating-system"></a>[1. Foundations: the machine and the operating system](#section-1-foundations-the-machine-and-the-operating-system)
- <a id="toc-section-2-processes-objects-and-handles"></a>[2. Processes, objects, and handles](#section-2-processes-objects-and-handles)
- <a id="toc-section-3-threads-and-scheduling"></a>[3. Threads and scheduling](#section-3-threads-and-scheduling)
- <a id="toc-section-4-memory-management"></a>[4. Memory management](#section-4-memory-management)
- <a id="toc-section-5-linking-portable-executables-and-loading"></a>[5. Linking, Portable Executables, and loading](#section-5-linking-portable-executables-and-loading)
- <a id="toc-section-6-windows-management-registry-services-and-wow64"></a>[6. Windows management: Registry, services, and WoW64](#section-6-windows-management-registry-services-and-wow64)
- <a id="toc-section-7-windows-security"></a>[7. Windows security](#section-7-windows-security)
- <a id="toc-section-8-synchronization-and-concurrency"></a>[8. Synchronization and concurrency](#section-8-synchronization-and-concurrency)
- <a id="toc-section-9-inter-process-communication-ipc"></a>[9. Inter-process communication (IPC)](#section-9-inter-process-communication-ipc)
- <a id="toc-section-10-hooking-injection-and-detection"></a>[10. Hooking, injection, and detection](#section-10-hooking-injection-and-detection)
- <a id="toc-section-11-high-value-distinctions"></a>[11. High-value distinctions](#section-11-high-value-distinctions)
- <a id="toc-section-12-whole-course-mental-model"></a>[12. Whole-course mental model](#section-12-whole-course-mental-model)
- <a id="toc-section-13-final-self-test"></a>[13. Final self-test](#section-13-final-self-test)
- <a id="toc-section-appendix-course-coverage-tracker"></a>[Appendix: course coverage tracker](#section-appendix-course-coverage-tracker)


---

<a id="section-1-foundations-the-machine-and-the-operating-system"></a>
# [1. Foundations: the machine and the operating system](#toc-section-1-foundations-the-machine-and-the-operating-system)

## 1.1 What a computer is doing

A computer repeatedly transforms state according to instructions.

- **CPU:** executes machine instructions.
- **Registers:** tiny, extremely fast storage locations inside a CPU core. They hold current operands, addresses, flags, and execution state.
- **RAM:** holds the active code and data for running programs. It is larger and slower than registers/cache, and loses its contents when power is removed.
- **Secondary storage:** SSD/HDD storage is much larger and persistent, but slower than RAM.
- **I/O devices:** keyboards, displays, disks, and network devices exchange data with the system through controllers and drivers.
- **Bus/interconnect:** carries data and control signals between components.

The basic CPU cycle is:

`fetch instruction -> decode instruction -> execute -> update state -> repeat`

A **thread**, not a process in the abstract, supplies the instruction stream and register state that a CPU executes.

### CPU terminology

- **CPU package:** the physical chip installed in a socket.
- **Physical core:** an independent execution core inside a package.
- **Logical processor:** a scheduling target exposed to the OS. Simultaneous multithreading may expose more than one logical processor per physical core.
- **Architecture / instruction set:** the machine-language contract, such as x86 or x64.

More logical processors allow more threads to execute at once, but do not guarantee proportional speedups. Threads may compete for caches, memory bandwidth, locks, and other shared resources.

## 1.2 Bits, binary, hexadecimal, and addresses

A **bit** is a 0 or 1. A group of 8 bits is a **byte**.

Hexadecimal is base 16 and uses `0-9` and `A-F`. One hex digit represents exactly four bits:

| Binary | Hex | Binary | Hex |
|---|---:|---|---:|
| `0000` | `0` | `1000` | `8` |
| `0001` | `1` | `1001` | `9` |
| `0010` | `2` | `1010` | `A` |
| `0011` | `3` | `1011` | `B` |
| `0100` | `4` | `1100` | `C` |
| `0101` | `5` | `1101` | `D` |
| `0110` | `6` | `1110` | `E` |
| `0111` | `7` | `1111` | `F` |

Windows tools use hex for addresses, flags, access masks, and file structures because bit boundaries are visible.

An N-bit unsigned value has `2^N` possible values. A 32-bit byte address can mathematically name `2^32` bytes = 4 GiB. A 64-bit pointer is 8 bytes wide, but current systems do not make the entire mathematical 64-bit range usable.

An address names one byte; it does not describe a whole region. A memory region is best written as a **half-open range**:

`base <= address < base + size`

Example: base `0x4000`, size `0x1000` contains `0x4000` through `0x4FFF`. Its first excluded address is `0x5000`.

## 1.3 Why an operating system exists

The OS has two central jobs:

1. **Abstraction:** present stable concepts instead of raw device details.
2. **Resource management and protection:** share processors, memory, storage, and devices safely and fairly.

Without an OS, every program would have to understand each exact disk controller, display, filesystem, and network device. The OS and its drivers expose abstractions such as:

- files instead of raw disk sectors;
- processes and threads instead of manually controlled CPU state;
- virtual memory instead of raw physical RAM addresses;
- sockets and pipes instead of device-specific communication;
- windows and input events instead of direct display/keyboard access.

Historically, operating systems evolved to solve several problems:

- **Batch processing:** automatically start the next job instead of waiting for a human operator.
- **Multiprogramming:** run other work while one job waits for slow I/O.
- **Preemption:** interrupt a running thread after a time interval so no cooperative program can keep the CPU forever.
- **Memory protection:** stop ordinary programs from overwriting one another or the kernel.
- **Hardware independence:** place device-specific behavior behind drivers and APIs.

A helpful model is:

`application intent -> API contract -> OS policy check -> OS mechanism -> result/error`

## 1.4 Multiprocessing, concurrency, and preemption

- **Multiprocessing:** the machine has multiple processors/cores capable of simultaneous work.
- **Concurrency:** multiple tasks make progress during overlapping time periods. They may be interleaved on one core.
- **Parallelism:** multiple tasks literally execute at the same moment on different logical processors.
- **Preemption:** the OS can stop a running thread, save its context, and schedule another.

If a thread waits for disk or network I/O, it is not ready to use a CPU. The scheduler can run another ready thread rather than leave the processor idle.

## 1.5 Windows architecture

Modern desktop Windows belongs to the Windows NT family. A simplified path from application to hardware is:

`application -> Win32 DLL -> Ntdll / system-call boundary -> executive/kernel -> driver -> hardware`

Important pieces:

- **Win32 API:** documented application-facing interface. DLLs include Kernel32, Advapi32, User32, and Gdi32.
- **Ntdll.dll:** provides the user-mode side of the Native API and system-call transition code. It also contains loader/runtime support.
- **Executive:** kernel-mode managers for objects, processes/threads, virtual memory, I/O, security, configuration, and other resources.
- **Kernel:** low-level scheduling, traps, interrupts, and synchronization.
- **Drivers:** kernel components that manage devices or participate in I/O stacks.
- **HAL:** hides some hardware differences from higher kernel components.

Visible system processes include:

- `System`: kernel-backed system activity;
- `services.exe`: Service Control Manager;
- `lsass.exe`: security policy and authentication support;
- `svchost.exe`: hosts one or more DLL-based services.

Exact arrangements differ between Windows versions and configurations.

## 1.6 User mode and kernel mode

Execution mode is a CPU-enforced privilege boundary.

- **User mode:** ordinary applications cannot execute privileged instructions or directly access kernel memory/hardware.
- **Kernel mode:** the Windows kernel and most drivers can access system-wide memory and privileged operations.

A user-mode failure is usually contained to one process. A kernel-mode bug can corrupt the entire machine or crash Windows.

Administrator status is not kernel mode. An elevated administrator program still executes in user mode; elevation changes its **security token and permissions**, not its processor privilege level.

### Controlled entries into the kernel

- **System call:** a thread requests a protected OS service.
- **Exception:** synchronous event caused by the current instruction, such as an invalid access or page fault.
- **Interrupt:** usually asynchronous hardware notification, such as device completion or a timer.

For a system call, the processor enters a trusted kernel entry point. Windows validates user-supplied pointers, lengths, access rights, and object state, performs the operation, then returns to user mode.

## 1.7 API, ABI, and system call

These are related but not identical:

- **API:** source-level contract: function purpose, parameters, return value, errors, and lifetime rules.
- **ABI:** binary contract: calling convention, data sizes/layout, register/stack use, and symbol representation.
- **System call:** controlled transition into kernel code.

A Win32 call is not automatically a system call. Some Win32 functions work entirely in user mode; others call a lower Native API function that crosses into the kernel.

Prefer documented Win32 contracts for applications. Native system-call details are implementation details and may change.

## 1.8 Reading a Windows API contract

Common parameter annotations:

- `[in]`: caller supplies data.
- `[out]`: function writes data for caller.
- `[in, out]`: both.
- `optional`: null may be valid.

A pointer does not automatically mean output. It may point to input, output, an array, a structure, or optional data.

The Unicode `W` variants, such as `CreateFileW`, are the modern default. `A` variants use legacy ANSI code-page conversion.

### Win32 type clues

- `BYTE` = 8-bit value.
- `WORD` = 16-bit value.
- `DWORD` = 32-bit value.
- `BOOL` = integer Boolean convention.
- `HANDLE` = opaque handle value.
- `LPVOID` = pointer to unspecified data.
- `LPCTSTR` = pointer to constant text, not a numeric value.
- types ending in `_PTR` are pointer-sized and change width with architecture.

## 1.9 Calling Win32 from Python

**pywin32** supplies Python wrappers for many Win32 APIs. It usually handles native declarations and converts failures into `pywintypes.error` exceptions.

**ctypes** calls DLL exports directly. It is useful when pywin32 has no wrapper or when learning the native boundary, but the caller must define:

- `argtypes` and `restype`;
- exact integer and pointer widths;
- structures and alignment;
- calling convention;
- Unicode/ANSI variant;
- error checking;
- cleanup.

A wrong `ctypes` declaration can truncate pointers or corrupt memory. Load Windows DLLs with last-error support when required, test the documented failure sentinel, capture the error immediately, and raise/report it before another call overwrites it.

Resource cleanup belongs next to acquisition. Typical pairs include:

- `OpenProcess -> CloseHandle`
- `CreateFile -> CloseHandle`
- `LoadLibrary -> FreeLibrary`
- `MapViewOfFile -> UnmapViewOfFile`
- `OpenService -> CloseServiceHandle`
- `VirtualAlloc(... reserve ...) -> VirtualFree(... MEM_RELEASE ...)`

## 1.10 Inspecting Windows

Start with a question, not with a tool.

- **Process Explorer:** snapshot of processes, ancestry, threads, handles, modules, tokens, integrity, and current resource use.
- **Process Monitor:** time-ordered trace of file, Registry, process, thread, and image-load operations.
- **WinObj:** Object Manager namespace.
- **VMMap:** one process's virtual address-space regions.
- **RAMMap:** system-wide physical-memory use.

A snapshot tells what exists now. A trace tells what happened over time. PID, TID, process creation time, image path, and architecture connect evidence. PIDs can be reused, so a PID alone is not permanent identity.

`Access denied` is useful evidence: it reveals an access check or protection boundary, not merely a tool failure.


---

<a id="section-2-processes-objects-and-handles"></a>
# [2. Processes, objects, and handles](#toc-section-2-processes-objects-and-handles)

## 2.1 Program versus process

A **program** is passive code and data stored in a file. A **process** is a live OS-managed container created from a program image.

A process contains or refers to:

- a private virtual address space;
- one or more threads;
- a handle table;
- a primary access token;
- loaded executable images and DLLs;
- environment variables and command line;
- process ID, parent metadata, times, quotas, and accounting data.

The executable is input to process creation; it is not the running process itself. The same executable can produce many independent processes.

Process isolation is selective, not absolute. Processes normally have separate virtual address spaces and handle tables, but they can intentionally share kernel objects, mapped memory, files, or IPC endpoints.

## 2.2 Process ancestry and identity

The process tree records creation ancestry: which process created which child. It does not automatically imply lifetime ownership. A parent can exit while a child continues.

For dependable identity, combine:

`PID + creation time + image path + architecture`

Windows can reuse a PID after a process exits. If group lifetime control is required, a **Job Object** can manage related processes as a unit and enforce limits or termination policy.

## 2.3 Threads and the process container

The process owns shared resources; its threads perform execution.

Threads in one process share:

- address space and global variables;
- executable code and loaded DLLs;
- process handles;
- security context by default.

Each thread has its own:

- registers/instruction pointer;
- user and kernel stacks;
- thread-local storage;
- scheduling state and priority;
- thread ID;
- optional impersonation token.

This sharing makes communication cheap but creates synchronization problems.

## 2.4 Process creation

`CreateProcess` conceptually creates two things together:

1. the process container;
2. its initial thread.

A simplified sequence is:

1. Resolve executable and command line.
2. Validate the image and architecture.
3. Create process/address-space structures.
4. Create the initial thread and stacks.
5. Map the executable and core runtime support.
6. Establish token, environment, current directory, and inherited resources.
7. Loader resolves DLL dependencies and initializes modules.
8. Initial thread reaches the program entry point.

The caller normally receives a process handle, thread handle, PID, and TID. The handles must be closed even after the process/thread has ended.

Executable selection and command-line parsing are separate concerns. Quoting mistakes can select the wrong executable or give the child incorrect arguments.

### Handle inheritance

Inheritance occurs only when both are true:

- the individual handle is marked inheritable; and
- process creation enables handle inheritance.

Modern code should restrict inheritance to an explicit list when possible. Accidentally inherited handles can leak authority and keep files or pipe ends alive.

A hidden window is not a hidden process. Window display settings affect the UI; the process remains visible to OS inspection tools.

## 2.5 Kernel objects and the Object Manager

Windows represents many resources as **kernel objects**, including processes, threads, files, events, mutexes, semaphores, sections, and tokens.

The Object Manager provides common behavior:

- creation and deletion;
- type information;
- naming and namespace lookup;
- security descriptors and access checks;
- reference counting;
- handles.

A named object can be found by other processes. Its name provides discoverability, but does not grant permission. The caller must still pass an access check.

Object Manager paths are not ordinary filesystem paths. Names such as device objects, symbolic links, and named events exist in an object namespace. WinObj displays this namespace.

## 2.6 Handles

A **handle** is a process-local value indexing an entry in that process's handle table. The entry refers to a kernel object and records the access rights granted when the handle was created/opened.

Think of a handle as:

`reference to object + granted authority`

Consequences:

- The same numeric handle value can mean different things in different processes.
- A PID names a process candidate; a process handle is checked access to that process object.
- Requesting more rights can make an open fail when a smaller request would succeed.
- A handle does not automatically gain new rights if an object's permissions later change.
- Closing a handle releases that process's reference, not every reference everywhere.

Use least privilege: request only the access bits required for the intended operation.

### Reference counting and object lifetime

Kernel objects commonly remain alive while references exist. References may come from handles and from the kernel itself.

`create/open -> reference count increases`

`close -> one reference released`

`last reference gone -> object can be deleted`

An execution object may remain inspectable after execution ends if handles still refer to it. Likewise, a file may remain open because another process inherited or duplicated the handle.

## 2.7 Duplicate and inherited handles

- **Inheritance:** transfers selected handles during child creation.
- **DuplicateHandle:** explicitly creates a handle-table entry in another process, potentially with different allowed rights.
- **Named object open:** another process locates the object by name and requests its own handle.

These are three different ways to share access to one underlying object.

## 2.8 Files, handles, buffering, and durability

Writing bytes through a file handle does not necessarily mean the physical device has persisted them. Data can remain in application, OS, filesystem, controller, or device caches.

- `CloseHandle` ends one reference and normally causes buffered state to be finalized through ordinary rules.
- `FlushFileBuffers` asks Windows to push buffered file data toward durable storage.
- Flushing frequently improves durability guarantees but hurts performance.

File lifetime and data durability are related but different questions.


---

<a id="section-3-threads-and-scheduling"></a>
# [3. Threads and scheduling](#toc-section-3-threads-and-scheduling)

## 3.1 Thread basics

A **thread** is the basic schedulable execution entity. A process with no running threads cannot execute application instructions.

Thread creation allocates more than a function call: a thread object, ID, user stack, kernel stack, register context, and scheduling state are needed.

Creating many threads has costs:

- stack/address-space reservation;
- kernel bookkeeping;
- scheduling and context switching;
- cache disruption;
- synchronization contention.

The right thread count depends on workload. CPU-bound work tends to benefit up to available processor capacity; I/O-bound work can use more threads because many will wait. Measurement matters.

### CPython note

Python `threading.Thread` uses native Windows threads, but the normal CPython Global Interpreter Lock prevents multiple threads from executing Python bytecode simultaneously in one interpreter. Threads still overlap I/O well, and native extensions may release the GIL. The GIL is not a general synchronization mechanism for application invariants.

## 3.2 Thread context and stacks

A **thread context** is the machine state needed to resume a thread:

- instruction pointer;
- stack pointer;
- general registers;
- flags and architecture-specific state.

During a context switch, Windows saves the outgoing thread's state and restores another's. Context switching enables sharing but consumes time and disturbs caches/TLB state.

Each thread normally has:

- **user stack:** application calls, local variables, return addresses while in user mode;
- **kernel stack:** trusted call frames while that thread executes kernel code.

A stack trace is a sampled path of nested function calls. It is evidence about what the thread was doing at that instant, not its complete history.

## 3.3 Thread states

Important simplified states:

- **Running:** currently executing on a logical processor.
- **Ready:** able to run, waiting for a processor.
- **Waiting/blocked:** cannot proceed until an event, lock, I/O completion, timer, or other condition occurs.
- **Terminated:** execution has ended; the thread object may remain until references close.

Ready and waiting are different. A ready thread wants CPU time; a waiting thread is not eligible until its condition is satisfied.

Suspending a thread is not synchronization. Arbitrarily freezing a thread can leave it holding locks or leave shared data midway through an update.

## 3.4 Creating and ending threads safely

For Python application work, prefer `threading.Thread` over raw `CreateThread`, because Python must manage interpreter state.

- `start()` begins execution.
- `join()` waits for completion. It does not request cancellation.
- a stop flag/event asks the thread to finish cooperatively.

Safe shutdown pattern:

1. signal a stop event;
2. unblock any waiting thread if necessary;
3. thread observes the signal;
4. thread restores invariants and releases resources;
5. caller joins it;
6. remaining handles/resources are closed.

Forced termination is dangerous because it can skip `finally` blocks, leave locks owned, leak memory, and strand shared state. Daemon threads only change Python's process-shutdown policy; they do not make abrupt cleanup correct.

## 3.5 Scheduling

Windows scheduling is primarily **preemptive and priority-driven**. Each logical processor selects a ready thread to run.

A simplified decision:

1. Threads become ready because they were created, unblocked, or preempted.
2. The scheduler favors the highest-priority ready work allowed on a processor.
3. A thread runs until it blocks, finishes, exhausts its quantum, yields, or is preempted by higher-priority work.
4. Its context is saved; another thread's context is restored.

Windows schedules threads, not whole processes. Process-level settings often establish a base from which individual thread priorities are calculated.

## 3.6 Affinity

**Affinity** restricts which logical processors may run a process/thread. It is a constraint, not extra capacity.

Possible uses include controlled benchmarks or specialized locality requirements. Misuse can reduce performance by preventing the scheduler from balancing load.

If an experiment changes affinity, record and restore the original mask.

## 3.7 Priority, boosts, starvation, and inversion

Priority ranks ready threads. Raising priority does not create CPU capacity or make blocked I/O finish faster.

Windows uses temporary **dynamic priority boosts** to improve responsiveness, for example when interactive or I/O-bound work becomes ready. Boosts decay rather than permanently changing the base priority.

- **Starvation:** a ready participant waits indefinitely because others keep winning access.
- **Priority inversion:** high-priority work waits for a resource held by lower-priority work, potentially while medium-priority work delays the owner.
- **Real-time priority danger:** excessively high threads can starve essential system work, including input or cleanup.

Elapsed time and CPU time are different:

- **Elapsed/wall time:** total clock duration.
- **CPU time:** time actually executing instructions on processors.
- **Waiting time:** time not eligible because the thread awaits something.

Benchmarks should use repeated runs, stable inputs, and the correct metric. One fast run may reflect cache warmth or scheduling luck.


---

<a id="section-4-memory-management"></a>
# [4. Memory management](#toc-section-4-memory-management)

## 4.1 The memory hierarchy

Storage layers trade speed, size, and persistence:

`registers -> CPU caches -> RAM -> SSD/HDD`

Fast layers are small and expensive; slow layers are large and cheaper.

Caches work because programs show **locality**:

- **Temporal locality:** recently used data is likely to be used again.
- **Spatial locality:** nearby data is likely to be used soon.

CPU caches transfer data in **cache lines**. Virtual memory manages **pages**. These are different mechanisms and sizes.

Cache coherence keeps multiple processors' views consistent, but does not make shared writes free. Threads repeatedly writing the same cache line cause coherence traffic. **False sharing** occurs when threads modify unrelated variables that happen to occupy the same cache line.

Do not confuse:

- **CPU cache:** hardware cache for memory bytes/instructions;
- **file cache:** OS caching of file data in physical memory;
- **working set:** pages of a process currently resident in RAM.

## 4.2 Virtual memory

A pointer in a normal process is a **virtual address**, meaningful inside that process's address space. The Memory Manager and CPU translate virtual pages to physical frames or other backing.

Virtual memory provides:

- isolation between processes;
- a private, consistent address-space view;
- page-level permissions;
- sparse address spaces and flexible placement;
- sharing where explicitly requested;
- the ability for not all mapped data to be resident at once.

The same virtual address in two processes can refer to different data. Shared memory may map the same backing at different virtual addresses.

## 4.3 Pages, frames, page tables, and TLBs

- **Page:** fixed-size block of virtual address space.
- **Frame/page frame:** fixed-size block of physical RAM.
- **Page table:** per-process translation structures mapping virtual pages and storing state/permissions.
- **PTE:** page-table entry containing mapping and control bits.
- **TLB:** CPU cache of recent address translations, not application data.

On a typical system, a page is `0x1000` = 4096 bytes = `2^12`. Therefore:

- low 12 address bits = offset within the page;
- remaining high bits identify the virtual page;
- page base = `address & ~(0x1000 - 1)`;
- page offset = `address & (0x1000 - 1)`.

Example: address `0x12345` lies on page base `0x12000` at offset `0x345`.

Query actual system values. Windows commonly uses 4 KiB pages but also supports large pages.

### Page size versus allocation granularity

- **Page size:** unit used for commitment, protection, translation, and faults; commonly 4 KiB.
- **Allocation granularity:** alignment for reservation base addresses; commonly 64 KiB on Windows.

They answer different questions.

## 4.4 Address translation

Conceptually:

1. CPU separates virtual address into virtual-page number and page offset.
2. It checks the TLB for a cached translation.
3. On TLB miss, hardware walks page tables.
4. The PTE says whether the mapping is present and what access is allowed.
5. If valid and resident, physical frame + unchanged offset forms the physical access.
6. If state/access requires OS work, a page fault occurs.

Page-table permissions can enforce read, write, and execute rules. Protection is checked by hardware and OS-maintained mapping state.

## 4.5 Reserve, commit, and resident

These terms are frequently confused:

- **Reserved:** virtual address range is set aside so other allocations cannot use it. It does not yet promise usable initialized storage.
- **Committed:** Windows promises backing for the pages if needed; accessing them can produce real storage.
- **Resident:** the page currently occupies a physical RAM frame.

A committed page need not currently be resident. A large virtual size does not mean the process is consuming the same amount of physical RAM.

Related measurements:

- **Virtual size:** range of virtual address space represented by mappings/reservations.
- **Commit charge/private commit:** committed private memory requiring backing.
- **Working set:** pages currently resident for the process.
- **Private working set:** resident pages not shareable with other processes.

## 4.6 Page faults, paging, and the page file

A **page fault** means the current memory access cannot complete using the present translation state. It is a control transfer to the OS, not automatically an error and not automatically a disk read.

Possible cases:

- **Demand-zero fault:** first use of a committed page; Windows supplies a zeroed page.
- **Soft fault:** page can be resolved from memory, such as a standby list or another mapping; no storage read needed.
- **Hard fault:** data must be read from storage, such as an image/file or page file.
- **Copy-on-write fault:** a private writable copy must be made.
- **Protection fault:** access violates permissions and may become an exception.
- **Invalid access:** address is not valid/committed and cannot be resolved normally.

General sequence:

`memory reference -> PTE cannot satisfy it -> trap to Memory Manager -> validate -> obtain/map page or reject -> update translation -> retry instruction or raise exception`

### Paging and the page file

Under memory pressure, Windows can remove pages from a process working set. Clean file-backed pages can be discarded and read again from their file. Modified private pages need backing, commonly the page file, before their RAM frame can be reused.

The page file supports system commit and private-page backing. It is not simply “extra RAM,” and not every page fault reads it.

High page-fault counts alone do not prove memory pressure. Combine them with available memory, hard-fault/disk activity, commit, and working-set behavior.

## 4.7 Shared memory and copy-on-write

Processes share memory through a common **section/file-mapping object**. Each process maps a view of that backing into its own address space.

Important: they share backing, not pointer values. Use offsets inside a defined shared layout, because the views may be mapped at different base addresses.

Sharing memory does not define a safe protocol. Processes still need:

- data layout and version;
- size/length fields;
- ownership rules;
- synchronization for visibility and ordering;
- initialization and shutdown states;
- validation of untrusted contents.

### Copy-on-write (COW)

Initially, multiple mappings can refer to one physical page while appearing private. A write causes a fault; Windows allocates/copies a private page for the writer and changes its PTE. Other processes retain the original page.

COW saves memory until modification while preserving isolation.

## 4.8 VirtualAlloc and protection

`VirtualAlloc` manages virtual-memory regions/pages.

Common stages:

1. reserve an aligned range;
2. commit pages within it;
3. access them under current protection;
4. optionally change protection with `VirtualProtect`;
5. decommit pages or release the full reservation correctly.

Protections include no access, read-only, read/write, execute/read, etc. Protection expresses permitted access, not intended data type.

Avoid writable-and-executable memory where possible. A safer code-generation transition is controlled `RW -> RX`, with the appropriate instruction-cache handling, rather than permanent `RWX`.

Whole-reservation release uses the original allocation base, `MEM_RELEASE`, and size zero. APIs have exact lifetime rules; an interior pointer is not a substitute for the original base.

## 4.9 Heaps

Page allocation is too coarse and expensive for every small object. A **heap allocator** obtains larger virtual-memory regions and suballocates smaller blocks in user mode.

- `VirtualAlloc`: region/page-level control, protections, reservation/commit.
- `HeapAlloc`: efficient small-block allocation from a heap.

Each process starts with a process heap and may create more. A heap can grow by acquiring more virtual memory.

`HeapAlloc` and `HeapFree` must use the same heap. `HeapSize` describes an allocated block according to that allocator. A 100-byte allocation may sit inside a much larger heap region in VMMap.

## 4.10 Inspecting address space

`VirtualQueryEx` returns contiguous runs of pages with matching state, protection, and type.

- **BaseAddress:** first page of the returned region.
- **AllocationBase:** start of the larger original allocation. Several regions can share it.
- **RegionSize:** size of this returned run.
- **State:** free, reserved, or committed.
- **Type:** private, mapped, or image.
- **Protect:** current access protection.

Do not use `AllocationBase` as every row's range start. The row range is `BaseAddress` through `BaseAddress + RegionSize - 1`.

VMMap classifies one process's virtual regions. RAMMap explains system-wide physical pages. Neither magically exposes all source-level ownership; interpretation still matters.


---

<a id="section-5-linking-portable-executables-and-loading"></a>
# [5. Linking, Portable Executables, and loading](#toc-section-5-linking-portable-executables-and-loading)

## 5.1 From source code to a running program

Several stages solve different problems:

1. **Preprocessing** expands language-level directives such as includes/macros.
2. **Compilation** translates source into machine code and metadata, often one object file per source unit.
3. **Linking** combines object files/libraries, resolves symbols, and lays out a final executable image.
4. **Loading** maps that image and its runtime dependencies into a process.
5. **Execution** begins at the runtime/application entry path.

An error belongs to the stage where its required information becomes available:

- syntax/type error -> compilation;
- unresolved external symbol -> linking;
- missing/incompatible DLL -> loading;
- access violation after start -> execution.

## 5.2 Symbols, object files, and the linker

A **symbol** names code or data such as a function or global variable. A compiler can emit a reference without yet knowing the final address.

An object file contains machine code, data, symbols, unresolved references, and relocation information. The linker:

- combines compatible sections;
- chooses final layout;
- resolves references between object files and libraries;
- emits import information for dynamic dependencies;
- produces the final PE image and supporting metadata.

## 5.3 Static and dynamic linking

### Static linking

Selected library code is copied into the executable at link time.

Advantages:

- fewer runtime library dependencies;
- predictable availability of included code;
- no loader resolution for that copied library code.

Costs:

- larger executable and duplicated code across programs;
- library fixes require rebuilding/relinking the application;
- less physical-memory sharing of common library pages.

A static `.lib` is broadly an archive of object files. The linker normally extracts only needed members.

### Dynamic linking

Code remains in a DLL and is resolved/mapped at runtime.

Advantages:

- one DLL file can serve many programs;
- read-only code pages may be shared in physical memory;
- components can be updated independently when ABI compatibility is preserved;
- runtime feature selection is possible.

Costs:

- dependency and version/architecture problems can appear at load time;
- DLL search/selection becomes a correctness and security issue;
- exported function signatures must remain ABI-compatible.

Dynamic linking can be:

- **implicit:** imports are recorded in the PE; the loader resolves them during startup;
- **explicit:** code calls `LoadLibrary`/`LoadLibraryEx` and `GetProcAddress` at runtime, then `FreeLibrary` when done.

Each successful `LoadLibrary` contributes a module reference that must be balanced according to the API contract. An `HMODULE` corresponds to a loaded module base in that process, but ownership still follows reference rules.

## 5.4 The Portable Executable (PE)

PE is the Windows image format for `.exe`, `.dll`, and many `.sys` files. It is a **mapping recipe**, not a byte-for-byte memory dump.

Main navigation chain:

`DOS header -> PE signature -> COFF/File header -> Optional header -> data directories -> section headers -> section data`

### DOS header

- Starts with `MZ` (`0x4D 0x5A`).
- Field at file offset `0x3C` (`e_lfanew`) points to the PE header.
- DOS stub commonly contains “This program cannot be run in DOS mode.”

### PE signature and File header

The modern signature is `PE\0\0`. The File/COFF header includes:

- **Machine:** target architecture, such as x86 `0x014C` or x64 `0x8664`;
- number of sections;
- size of Optional Header;
- characteristics such as executable/DLL flags.

### Optional Header

Despite its name, it is required for executable images. Key fields include:

- `Magic`: distinguishes PE32 and PE32+ layout;
- `AddressOfEntryPoint`: RVA of initial image entry code;
- `ImageBase`: preferred load base;
- `SectionAlignment` and `FileAlignment`;
- `SizeOfImage` and `SizeOfHeaders`;
- subsystem (console, GUI, etc.);
- data-directory count/array.

Use both `Machine` and `Magic` when checking architecture/layout.

### Data directories

An array of `(RVA, size)` entries points toward major structures, including imports, exports, resources, relocations, exceptions, TLS, and security-related data. A zero entry means that directory is absent.

A data directory points into the image layout. To read it from the file, its RVA normally has to be translated through a section.

## 5.5 Sections and coordinate systems

Common sections:

- `.text`: executable code;
- `.rdata`: read-only data and tables;
- `.data`: initialized writable globals/statics;
- `.rsrc`: icons, dialogs, and other resources;
- `.reloc`: base-relocation information.

Section names are conventions, not security guarantees. Use section characteristics to understand intended mapped permissions.

### File offset, RVA, and VA

- **File offset:** position in the on-disk file.
- **RVA (relative virtual address):** offset from image base in the mapped image.
- **VA (virtual address):** actual process address: `VA = actual image base + RVA`.

Example: if actual base is `0x7FF600000000` and function RVA is `0x1230`, its VA is `0x7FF600001230`.

To translate an RVA into a raw file offset, find the section whose virtual range contains it, then use roughly:

`file offset = PointerToRawData + (RVA - section.VirtualAddress)`

Validate all ranges. A parser must not trust header values to stay within the file.

### Raw size versus virtual size

The file and memory layouts have different alignment. A section can have:

- bytes stored on disk but padded/aligned differently in memory;
- a larger virtual tail that the loader initializes to zero;
- a raw file position unrelated to its RVA.

This is why treating a PE file as a memory dump fails.

## 5.6 ASLR and relocations

The preferred `ImageBase` may be unavailable or deliberately randomized by **Address Space Layout Randomization (ASLR)**. The loader chooses an actual base.

RVA-based internal structure remains stable relative to the chosen base. Absolute addresses embedded in the image may require **base relocations**, which describe locations the loader must adjust by:

`delta = actual base - preferred base`

ASLR makes hard-coded process addresses unreliable across launches.

## 5.7 Imports, exports, and the IAT

### Imports

An import directory describes external DLLs and symbols the image needs. For each imported DLL, descriptors lead to:

- DLL name;
- names/ordinals requested;
- locations the loader must fill with resolved addresses.

The **Import Address Table (IAT)** ultimately contains virtual addresses of resolved imported functions. Compiled code can call indirectly through these slots.

### Exports

A DLL export table publishes functions/data by name and/or ordinal. An export may be a forwarder directing resolution to another DLL.

`GetProcAddress` returns a raw address. That address is not enough to call safely: the caller must know the correct parameters, return type, calling convention, and lifetime of the containing module.

## 5.8 What the Windows loader does

A simplified startup sequence:

1. Kernel process creation establishes core process/thread state and maps the executable plus Ntdll support.
2. User-mode loader code reads PE metadata.
3. It maps sections with appropriate layout/protections.
4. It recursively locates and loads dependencies.
5. It resolves exports and fills import address tables.
6. It applies relocations when necessary.
7. It establishes loader-maintained module state and runtime metadata.
8. It runs constrained initialization notifications.
9. Control eventually reaches runtime/application entry code.

Much of PE dependency resolution and initialization is user-mode loader work, even though file mapping and protected memory operations require kernel support.

Dependencies form a graph, not a simple list. The loader must track module identity and initialization state to avoid loading/initializing the same dependency incorrectly.

### `DllMain` constraints

DLL entry notifications run under loader constraints. Complex work can deadlock or recursively invoke loader-sensitive operations. Keep `DllMain` minimal: avoid broad initialization, waiting on other threads, and unsafe loading behavior. Defer substantial work until after loader initialization.

## 5.9 DLL search security

A bare DLL name is not provenance. The result depends on loader search policy, process configuration, known DLL handling, packaged-app rules, and explicit directories.

Safer practices:

- use supported secure search flags/APIs;
- prefer trusted, explicit locations when appropriate;
- avoid relying on the current directory;
- verify the loaded module path;
- preserve architecture compatibility;
- treat unexpected dependencies as evidence worth investigating.

Use static PE inspection to learn declared imports, Process Monitor for load-time events, and Process Explorer/ListDLLs for the current loaded-module snapshot. Each answers a different question.


---

<a id="section-6-windows-management-registry-services-and-wow64"></a>
# [6. Windows management: Registry, services, and WoW64](#toc-section-6-windows-management-registry-services-and-wow64)

## 6.1 The Registry model

The Windows Registry is a hierarchical database for OS and application configuration.

- **Key:** container resembling a directory; can have subkeys and values.
- **Value:** named typed data stored inside a key.
- **Value name:** separate from the value's data and type.
- **Hive:** persisted/logical body of Registry data loaded into the namespace.
- **Root/predefined handle:** top-level logical entry such as HKLM or HKCU.

A path typically identifies a key, then an operation specifies a value name. The `(data, type)` pair carries meaning.

Common value types:

- `REG_SZ`: Unicode string;
- `REG_EXPAND_SZ`: string containing environment-variable references;
- `REG_MULTI_SZ`: sequence of null-separated strings;
- `REG_DWORD`: 32-bit integer;
- `REG_QWORD`: 64-bit integer;
- `REG_BINARY`: uninterpreted bytes.

## 6.2 Registry roots and views

Important roots:

- **HKLM:** machine-wide configuration.
- **HKCU:** current user's view/configuration.
- **HKU:** loaded user profiles by SID.
- **HKCR:** merged class/association view derived from machine and user data.
- **HKCC:** current hardware-profile view.

Some roots are aliases or merged views rather than separate physical stores. The apparent path is an API namespace, not necessarily a single file location.

Registry keys are secured objects opened with requested rights. Opening returns a key handle; names do not bypass permissions.

## 6.3 Reading and modifying safely

Core operations:

- open existing key;
- create/open key;
- query named value and its type;
- enumerate subkeys/values by index;
- set/delete value;
- close key handle.

`OpenKey` promises not to create. `CreateKeyEx` may create missing state. Choose intentionally.

A reversible write must remember all of the original state:

- Did the value exist?
- If yes, what were its exact data and type?
- If no, cleanup should delete the newly introduced value rather than invent an old value.

Enumeration is a snapshot-like traversal of mutable data; another actor may add/delete items while indexes are being read. Code must tolerate change.

Use HKCU and a dedicated lab key for experiments. Avoid changing system startup/shell/service configuration while learning.

## 6.4 Services and the Service Control Manager

A **Windows service** is a program/component managed under a contract with the **Service Control Manager (SCM)**. It is not merely any background process.

A service record includes configuration such as:

- service name and display name;
- executable or hosting information;
- start type;
- dependencies;
- service account;
- accepted controls and recovery-related settings.

Separate two kinds of state:

- **Configuration:** durable service record.
- **Runtime status:** stopped, start-pending, running, stop-pending, etc., with checkpoint/wait-hint information.

Starting/stopping is asynchronous. A control request returning successfully often means “request accepted,” not “transition complete.” Correct code polls status, respects checkpoint progress/wait hints, handles timeouts, and reaches a terminal state.

## 6.5 Service handles and access

The normal relationship is:

`OpenSCManager -> SCM handle -> OpenService -> service handle -> query/control -> CloseServiceHandle`

The SCM and service are separate secured objects. Rights should match the action:

- enumerate/query SCM database;
- query service status/configuration;
- start service;
- stop/send control;
- change configuration.

Query-only tasks should not request start/stop/configure authority.

Service handles use `CloseServiceHandle`, not an arbitrary cleanup function. With `ctypes`, each service function needs its own exact signature and error rule.

Never use a critical service as a control experiment. Enumeration and status queries are the safe default.

## 6.6 Service hosts and background processes

Some services run in their own executables. Many DLL-based services run inside `svchost.exe` instances.

Consequences:

- one host PID may contain several independently named services;
- a service name is more durable than its current PID;
- a shared host creates some shared failure/resource boundary;
- “background process” does not prove “Windows service.”

Other background activation mechanisms include scheduled tasks, logon/startup entries, COM activation, and ordinary applications.

Correlate SCM enumeration with Process Explorer's service-per-process view, image path, loaded modules, command line, account, and signature. A PID only describes the current host instance.

## 6.7 WoW64

**WoW64** allows 32-bit x86 applications to run on 64-bit Windows through user-mode compatibility components plus kernel support. The processor executes 32-bit instructions using its compatibility capability; WoW64 bridges system interfaces and environment differences.

Key ideas:

- a 32-bit process has 32-bit pointers and loads 32-bit DLLs;
- it cannot load a 64-bit DLL into the same process;
- the native OS remains 64-bit;
- API-visible filesystem and Registry paths may be redirected/viewed differently.

### Filesystem redirection

Historical naming is counterintuitive:

- `System32` contains native system binaries (64-bit on x64 Windows).
- `SysWOW64` contains 32-bit system binaries.
- `Sysnative` is a special alias allowing a 32-bit process to reach the native System32 view in supported contexts.

Do not broadly disable filesystem redirection. Choose an architecture-aware supported path.

### Registry views

Some Registry locations expose distinct 32-bit and 64-bit views. APIs can explicitly request them with `KEY_WOW64_32KEY` or `KEY_WOW64_64KEY`.

Do not assume every key is redirected or manually insert `WOW6432Node`. State which view the program intends and use API flags.

**UAC virtualization is different:** it is a compatibility mechanism that may redirect some failed legacy writes for unelevated applications. It is not the 32/64-bit Registry-view system.

Architecture evidence should agree across file headers, process bitness, pointer width, and loaded-module architecture.


---

<a id="section-7-windows-security"></a>
# [7. Windows security](#toc-section-7-windows-security)

## 7.1 Authentication, authorization, and auditing

These answer different questions:

- **Authentication:** who are you?
- **Authorization:** may this subject perform this action on this object?
- **Auditing:** what security-relevant activity should be recorded?

Windows authorization can be modeled as:

`subject + requested action + secured object -> access decision`

The subject is represented by a security context/token. The object has a security descriptor. The requested action becomes an access mask.

Auditing policy is not permission. A DACL controls discretionary access; a SACL can request auditing for selected attempts, subject to system audit policy.

## 7.2 SIDs

A **Security Identifier (SID)** is a variable-length identity used for authorization. Users, groups, computers, services, and other principals can have SIDs.

Names are human-friendly translations. Authorization data uses SIDs because names can change and may not resolve when a domain/account source is unavailable.

Examples of well-known SIDs:

- Everyone: `S-1-1-0`;
- Local System: `S-1-5-18`;
- built-in/local-domain administrator and guest accounts use well-known relative IDs in their domain SID.

A SID string contains revision, identifier authority, and subauthorities. Do not compare users by display name alone.

## 7.3 Access tokens

After logon/authentication, Windows constructs an **access token** representing effective security state. Important token contents include:

- user SID;
- group SIDs and attributes;
- privileges and their state;
- integrity level;
- elevation/restriction information;
- token type and impersonation information;
- default DACL and other context.

A process normally has a **primary token**. A thread can temporarily have an **impersonation token**, which can become the effective subject for its operations.

Token handles have their own access rights. Permission to query a token is not permission to duplicate, adjust, or assign it.

Group membership alone is not enough; group attributes can mark membership enabled, disabled, deny-only, or otherwise affect access checks.

## 7.4 Security descriptors

A **security descriptor** is a structured policy record attached to a securable object. It can contain:

- **Owner SID**;
- **Primary group SID** (limited significance in typical Windows use);
- **DACL**: discretionary access-control list;
- **SACL**: auditing and mandatory-label-related information;
- control bits describing presence, inheritance, and representation.

The owner generally has special authority to change discretionary permissions, but ownership is not automatic full access to every operation.

## 7.5 ACLs and ACEs

An ACL contains ordered **Access Control Entries (ACEs)**. An ACE includes a trustee SID, type, access mask, flags, and sometimes object-specific data.

Common ACE types include allow and deny. ACE order and applicability matter; do not reduce an ACL to “sum all matching permissions.” Windows uses canonical ordering conventions because an earlier applicable deny/allow can affect remaining requested bits.

Access-mask bits are object-type-specific. `0x00000001` may represent different named rights for a file, process, event, or service. Generic rights such as `GENERIC_READ` must be mapped through that object's generic mapping.

### Crucial DACL states

- **Null DACL:** no DACL restriction; broadly grants access. Dangerous.
- **Empty DACL:** DACL exists with zero ACEs; grants no discretionary access.
- **DACL not requested/read:** caller simply has no information about it.
- **DACL absent/present flags:** must be interpreted from the descriptor APIs, not guessed from an empty Python value.

These states are not interchangeable.

## 7.6 Simplified access-check process

When a caller requests an object handle:

1. Identify effective token (thread impersonation token if present, otherwise process primary token).
2. Map generic requested rights to object-specific rights.
3. Consider privileges/mandatory policy and other object-specific rules.
4. Compare requested bits against applicable ACEs for enabled token SIDs in order.
5. Determine which requested bits are granted or denied.
6. If allowed, create a handle-table entry recording the granted access mask.

This is deliberately simplified: integrity policy, privileges, owner rights, restricted tokens, object callbacks, share modes, protected-process rules, and other mechanisms can also matter.

Important consequences:

- Access is commonly checked when the handle is opened/created.
- The handle preserves its granted rights; later DACL changes do not automatically rewrite it.
- Passing a DACL check does not guarantee a real open succeeds; the object can have additional state or sharing constraints.
- `Access denied` should be explained in terms of requested rights, effective token, policy, and object type.

## 7.7 Privileges

A **privilege** is a named capability in a token for particular system-wide operations, sometimes allowing behavior outside ordinary object DACL rules.

Examples include debugging other processes, backing up/restoring files, changing system time, or shutting down the system.

Principles:

- the privilege must already exist in the token;
- `AdjustTokenPrivileges` can enable/disable existing privileges but cannot invent one;
- the API can report overall success while a requested privilege was not assigned, so the specific result must be checked;
- enable powerful privileges only for the smallest required interval.

Privileges and object access rights are different. `SeDebugPrivilege` does not mean every API operation is universally permitted, especially with modern protected-process and mitigation boundaries.

## 7.8 Impersonation

Impersonation lets a server thread temporarily act under a client's security context.

Typical flow:

1. client connects/authenticates to IPC endpoint;
2. server impersonates the client on the handling thread;
3. downstream access checks use the impersonation token;
4. server performs only the client-authorized work;
5. server calls `RevertToSelf` in guaranteed cleanup.

Only the thread's effective subject changes; the whole process does not necessarily change identity.

Impersonation is useful because a privileged service can access resources on behalf of a client without granting the client the service's full authority. The server must validate what the client asks it to do and must always revert.

Creating a process as another identity is more involved: process creation needs an appropriate **primary token**, exact token/process rights, environment/profile handling, and a full launch contract. An impersonation token is not automatically suitable.

## 7.9 Security components

- **Security Reference Monitor (SRM):** kernel/executive security enforcement, including access checks.
- **LSASS:** user-mode security authority process supporting authentication policy, logon sessions, and related security work.
- **Authentication packages:** validate credentials through supported mechanisms.
- **Winlogon:** participates in interactive logon/session behavior.
- **SAM/domain directory:** identity/account data sources in their respective environments.

Keep policy/storage/authentication components distinct from the kernel component that enforces object access.

## 7.10 UAC and elevation

User Account Control reduces routine use of full administrator authority.

An administrator commonly runs ordinary applications with a filtered token. Elevation launches a process with a different, more capable token after policy/consent/credential handling.

Therefore:

- administrator group membership != currently elevated;
- elevation != kernel mode;
- a manifest can state requested execution level;
- UAC is not a security boundary against a malicious actor already executing as the same user in every scenario, but it is an important least-privilege/consent mechanism.

## 7.11 Integrity levels

Windows Mandatory Integrity Control labels subjects and objects with integrity levels, commonly low, medium, high, and system.

The central default idea is **no write up**: a lower-integrity subject is restricted from writing to a higher-integrity object even if the discretionary DACL might otherwise allow it. Mandatory policy is layered with the DACL; it does not replace it.

Examples:

- ordinary desktop processes usually run at medium integrity;
- elevated processes usually run at high;
- sandboxed/untrusted content may run at low or another restricted level;
- many system services run at system integrity.

Integrity and user/kernel mode are separate axes. Both low- and high-integrity applications normally execute in user mode.

## 7.12 Object reuse protection

When storage or memory is reassigned, Windows prevents a new user from reading leftover data belonging to a previous user, for example by supplying zeroed pages. This is **object reuse protection**.

It does not mean deleted data is impossible to recover from storage media. Sanitization and forensic recovery are different topics.


---

<a id="section-8-synchronization-and-concurrency"></a>
# [8. Synchronization and concurrency](#toc-section-8-synchronization-and-concurrency)

## 8.1 Why concurrent code goes wrong

Concurrency creates more than one possible ordering of operations. A program is correct only if every permitted interleaving preserves its required rules.

An **invariant** is a condition that must remain true, such as:

- list links form one valid structure;
- account balance equals the sum of recorded transactions;
- a queue count matches the number of entries;
- an object cannot be freed while another thread uses it.

Synchronization should protect the whole invariant, not merely one source line.

Shared state includes more than variables. It includes files, handles, object lifetime, buffers, queues, UI state, and ordering assumptions.

The strongest simplification is often to remove sharing: give one thread ownership of mutable data and send it work through a queue.

## 8.2 Race conditions and atomicity

A **race condition** occurs when correctness depends on timing/interleaving that the program did not control.

Example: `counter += 1` conceptually performs:

1. read old counter;
2. compute old + 1;
3. write new counter.

Two threads can both read the same old value and overwrite one another's update. One source statement does not imply one indivisible CPU/OS operation.

An operation is **atomic** relative to particular observers if it appears to happen completely or not at all; no observer sees a partial state. Atomicity must specify the operation, alignment/width, and memory-ordering rules.

The CPython GIL does not make a multi-step business invariant atomic. Python can switch around bytecode execution, C extensions can release the GIL, and the invariant may span several objects or operations.

### Interlocked operations

Windows Interlocked APIs provide atomic operations such as increment, exchange, add, and compare-exchange on correctly aligned values.

**Compare-and-exchange** means:

`if current == expected: current = desired; report prior/current result`

It enables narrow lock-free state transitions. It does not automatically make a complex data structure correct; algorithms also need ordering, lifetime, and ABA/reclamation reasoning.

## 8.3 Efficient waiting

A busy loop repeatedly checks a condition and consumes CPU:

`while not ready: pass`

A blocking wait lets the OS mark the thread waiting and schedule other work. When the object becomes signaled, Windows makes an appropriate waiter ready.

A trace/debugger can change timing and hide or expose races. This **observer effect** is why intermittent concurrency failures need repeated, controlled experiments.

## 8.4 Critical sections and mutexes

Both enforce mutual exclusion, but they have different scope/mechanics.

### Critical section

- process-local;
- used only by threads in one process;
- optimized to acquire in user mode when uncontended;
- may enter kernel waiting when contended;
- has ownership/reentrancy behavior in the Windows primitive.

### Mutex

- kernel waitable object;
- can be named and shared across processes;
- has thread ownership;
- ownership is acquired through a wait;
- owner releases it with `ReleaseMutex`;
- higher overhead than an uncontended process-local lock.

Use the simplest primitive that matches the sharing boundary. Do not choose a cross-process kernel object when a process-local lock is sufficient.

### Correct acquisition pattern

1. wait/acquire;
2. branch on the exact result;
3. only if ownership was obtained, enter protected work;
4. restore invariant in `finally`-style cleanup;
5. release exactly the acquired ownership;
6. close the handle separately when it is no longer needed.

Waiting, releasing ownership, and closing a handle are three different operations.

### Abandoned mutex

If a thread exits while owning a mutex, a later waiter may receive `WAIT_ABANDONED`. That waiter obtains ownership, but Windows is warning that the protected state may be inconsistent because the previous owner did not finish cleanup.

The response is not “continue normally.” Validate or rebuild the invariant, fail safely if it cannot be trusted, then release appropriately.

## 8.5 Semaphores

A **semaphore** stores a count of available permits:

- initial count = permits available at creation;
- successful wait decrements count;
- count zero means new waiters block;
- release increments count, up to a maximum.

Semaphores do not have an owner. Any correctly authorized participant can release permits, so the program must maintain the logical rule “one release per successful acquire.”

Uses include limiting concurrent database connections or bounding a worker pool.

A semaphore with maximum 1 can restrict admission like a binary gate, but it is not identical to an ownership-tracking mutex and has no abandoned-owner signal.

Capacity control and data protection are different. A semaphore allowing three workers into a region does not make their shared data updates safe; another lock/protocol may still be needed.

## 8.6 Events and waitable objects

An **event** stores a Boolean signaled/non-signaled condition. It communicates that a condition occurred or remains true; it does not carry arbitrary message data.

### Manual-reset event

- when set, releases any current waiters and remains signaled;
- future waiters also pass until someone calls `ResetEvent`;
- useful for broadcast state such as “shutdown requested.”

### Auto-reset event

- when signaled, releases one eligible waiter;
- then automatically becomes non-signaled;
- useful for one-unit notification, but not a replacement for a counted queue.

Event correctness depends on the protocol: who sets, who resets, what protected state the signal refers to, and whether signals can be coalesced.

Other waitable objects become signaled for defined reasons:

- process/thread object when execution terminates;
- timer when due;
- mutex when available;
- semaphore when count is positive;
- event when set;
- some I/O-related objects/operations on completion.

Every wait result must be interpreted: success/object index, timeout, abandoned mutex, or failure. A timeout is not successful acquisition.

## 8.7 Names, handles, and synchronization lifetime

A named mutex/event/semaphore can be created by one process and opened by another.

- Name enables discovery.
- Security descriptor controls who may open it.
- Requested access controls allowed operations.
- Each open handle adds a reference.
- The object disappears only after all relevant references are gone (unless another persistence mechanism applies).

Two processes may have different handle numbers for the same object. Handle identity is process-local; object identity is underlying and may be associated with a namespace name.

## 8.8 Deadlock, starvation, and livelock

### Deadlock

Threads form a cycle of dependencies in which none can proceed. Four classic necessary conditions are:

1. **Mutual exclusion:** a resource cannot be shared simultaneously.
2. **Hold and wait:** a participant holds one resource while waiting for another.
3. **No preemption:** resources cannot simply be taken away safely.
4. **Circular wait:** a dependency cycle exists.

Preventing any one condition prevents that class of deadlock. A common design is a global lock order: if all code acquires `A` before `B`, the `A <-> B` cycle cannot form.

### Starvation

One participant waits indefinitely while others continue to make progress, often because the policy is unfair or high-priority work continually arrives.

### Livelock

Participants remain active and react to one another but keep preventing progress, such as both repeatedly backing off and retrying in lockstep.

### Safe design rules

- minimize lock scope and number of simultaneously held locks;
- use one documented global acquisition order;
- do not call unknown/external code while holding internal locks;
- avoid blocking I/O under a lock where practical;
- prefer queues/message ownership over broad shared mutation;
- make cancellation and shutdown part of the protocol;
- treat timeout as a new branch requiring cleanup/recovery, not as automatic rollback.

Stack traces show where threads are stopped. Wait-chain analysis shows dependency relationships. Together they help explain hangs.


---

<a id="section-9-inter-process-communication-ipc"></a>
# [9. Inter-process communication (IPC)](#toc-section-9-inter-process-communication-ipc)

## 9.1 Why IPC must be explicit

Virtual-memory isolation means one process cannot normally dereference another process's pointers. Communication must use an explicit OS-mediated mechanism.

Every IPC design has at least four layers:

1. **Transport:** how bytes/state move (pipe, socket, mapping, etc.).
2. **Protocol:** how those bytes are framed and interpreted.
3. **Synchronization/backpressure:** when participants may read/write and what happens when capacity is exhausted.
4. **Security/trust:** who can connect/open, how peers are authenticated, and how untrusted input is validated.

Transport success does not prove protocol correctness or peer trust.

## 9.2 Protocol design

A byte stream has no inherent meaning. A protocol should define:

- message boundaries (fixed size, delimiter, or length prefix);
- encoding and byte order;
- version/type fields;
- maximum lengths and validation;
- request/response correlation;
- partial read/write behavior;
- EOF/disconnection meaning;
- errors and timeouts;
- startup and shutdown sequence.

Never trust a length field before checking it against a safe maximum and the available data.

**Backpressure** is what happens when producers are faster than consumers: writers may block, buffers may grow, data may be rejected/dropped, or the protocol may signal flow control. It must be a deliberate policy.

## 9.3 Anonymous pipes

An anonymous pipe is a one-way byte stream represented by a read handle and a write handle. It is commonly used between related processes, especially to redirect standard input/output/error.

Typical parent-child setup:

1. create pipe ends;
2. mark only intended child handles inheritable;
3. make the parent's retained ends non-inheritable;
4. place child ends in `STARTUPINFO` as standard handles;
5. create child with intended inheritance;
6. parent immediately closes its copies of child-only ends;
7. each side reads/writes its owned end;
8. close all ends and wait/collect exit status.

Inheritance requires both an inheritable handle and creation-time inheritance permission.

### EOF and pipe leaks

A reader sees EOF only after **every write handle** referring to that pipe end has closed. If the parent accidentally keeps an extra writer or passes it to another child, the reader can wait forever even though the intended writer finished.

For every pipe, draw an ownership table listing each process and which read/write ends it owns. Close unused ends as soon as creation succeeds.

Pipelines magnify mistakes: with N stages, each stage must inherit only its input/output and close every unrelated copy. Exit codes from all stages are part of the result, not just the final output bytes.

## 9.4 Named pipes

A named pipe provides a discoverable server/client IPC endpoint, commonly under a name such as `\\.\pipe\Name`.

Roles:

- server calls `CreateNamedPipe` to create an instance and waits/connects;
- client opens the name (often with file-style APIs);
- both exchange data using read/write operations;
- server can create multiple instances for multiple clients.

### Byte mode versus message mode

- **Byte mode:** a stream; reads need not match write boundaries.
- **Message mode:** preserves message boundaries, although a read buffer may still be too small and require continuation handling.

Connection races are normal: a client can connect around the server's connect call, and APIs document special outcomes for this case. Treat documented “already connected” results as a state transition, not blindly as failure.

### Named-pipe security

The pipe is a securable object. Its security descriptor controls who can open it. A server may inspect or impersonate the connected client, but must still validate every request.

Endpoint permission answers “who may connect?” It does not prove the client will send well-formed or authorized commands. Conversely, knowing a pipe name does not grant access.

## 9.5 File mappings / shared memory

A file-mapping object represents section-backed memory. It can be backed by a disk file or by paging-file-backed storage.

Typical native lifecycle:

1. create/open mapping object;
2. map a view into each process;
3. exchange data through the defined layout;
4. synchronize access separately;
5. unmap each view;
6. close each mapping handle.

Views and object handles are separate resources with separate cleanup.

Processes share backing, not virtual addresses. Never store a raw process pointer as the shared representation. Store an offset from the mapping base or another portable identifier.

### Visibility, ordering, and persistence

- **Visibility:** when another processor/process observes writes.
- **Ordering:** which writes must appear before others.
- **Persistence:** whether changed file-backed data has reached durable storage.

These are separate guarantees. A synchronization primitive can establish a protocol/order; `FlushViewOfFile` concerns propagation toward the mapped file/cache, and additional file/device flushing may be needed for a desired durability guarantee.

## 9.6 Synchronization objects as IPC

Named events, mutexes, and semaphores coordinate processes and therefore are IPC mechanisms, but they usually signal state rather than carry arbitrary data.

A common shared-memory protocol uses:

- mapping for data;
- mutex/lock for exclusive layout updates;
- event/semaphore for notification or available-items count.

Data movement and coordination remain distinct even when used together.

## 9.7 Choosing an IPC mechanism

Choose from constraints rather than familiarity:

| Need | Usually consider | Main concern |
|---|---|---|
| parent-child byte redirection | anonymous pipe | inheritance and closing extra ends |
| local named client/server | named pipe | framing, instances, ACL/client identity |
| large/high-throughput local data | shared mapping | layout and synchronization |
| network or cross-platform endpoint | socket | protocol, authentication, partial I/O |
| signal/coordination only | event/semaphore/mutex | state versus data semantics |
| durable loose coupling | file/database/message service | consistency and recovery |

Also evaluate:

- topology: one-to-one, pipeline, many clients, broadcast;
- local versus remote;
- stream versus messages;
- throughput and latency;
- backpressure;
- lifetime and reconnection;
- security boundary;
- observability and failure recovery.


---

<a id="section-10-hooking-injection-and-detection"></a>
# [10. Hooking, injection, and detection](#toc-section-10-hooking-injection-and-detection)

This module studies OS mechanisms and defensive evidence. The same low-level capabilities can support debugging, accessibility, instrumentation, security products, malware, or unauthorized tampering. Legitimacy depends on authorization, intent, implementation, and scope.

## 10.1 Injection versus hooking

- **Injection:** introduce code or a module into another execution context/process.
- **Hooking:** redirect or intercept an existing call/event path so another handler runs.

They can be combined but are not synonyms. A module may be injected without changing an existing function path; a supported hook can redirect events through code loaded by a documented mechanism.

Technique does not determine intent. Also, a technique working as designed is different from exploiting a vulnerability.

## 10.2 Natural code-loading paths

Before studying unusual loading, remember normal paths:

- executable and implicit imports loaded during startup;
- explicit `LoadLibraryEx` at runtime;
- documented plugin/extension systems;
- Windows hook mechanisms for supported event flows;
- COM or other component activation.

Startup imports are resolved before ordinary application entry. Architecture must match: 32-bit and 64-bit code cannot be freely mixed in one ordinary process.

DLL selection/search policy is part of security. Modifying an executable's import table or bytes changes the file identity and generally invalidates its digital signature.

## 10.3 Remote memory and remote threads: conceptual chain

The classic simplified injection pattern illustrates several previously learned concepts:

1. identify a target process instance;
2. open a process handle with specific rights;
3. allocate memory in that process;
4. write controlled data into that address space;
5. arrange executable code or reference a loader routine under matching architecture/ABI assumptions;
6. start execution in the target;
7. wait/check outcome and release handles/memory under correct lifetime rules.

This chain connects:

- process-object access checks;
- process-relative virtual addresses;
- allocation/protection rules;
- thread creation and scheduling;
- module loading;
- handle/reference cleanup.

An address valid in the injector is not automatically valid in the target. ASLR, architecture, mitigations, process protection, Control Flow Guard, code-integrity policy, and API restrictions make the real situation more complex than the textbook sequence.

Executable private memory is not automatically malicious—JIT runtimes generate code—but its provenance, protection transitions, start addresses, and surrounding events can make it suspicious.

## 10.4 Position-independent code

Ordinary compiled code often depends on:

- a known image layout/base;
- relocations;
- imports/IAT;
- initialized globals and runtime libraries;
- TLS and exception-unwind metadata;
- ABI-correct stack/calling convention.

Copying arbitrary function bytes elsewhere usually breaks those assumptions.

**Position-independent code** computes references without a fixed absolute load address. It remains architecture- and ABI-specific and must still solve external calls, data access, alignment, stack discipline, and runtime metadata.

A supported PE module lets the Windows loader handle imports, relocations, section protections, TLS, and other metadata. Raw instruction bytes shift all those responsibilities to the author.

## 10.5 Windows hooks

Documented Windows hook APIs can insert a callback into specific GUI/message/event paths.

Important properties:

- hook type and thread/desktop scope determine what is observed;
- some cross-process hooks require the callback code in a compatible DLL;
- the callback must use the exact ABI;
- it should do little work and avoid blocking/reentrancy problems;
- it normally calls the next hook as required by the contract;
- it must be unhooked before unloading its code.

A callback can run in sensitive/reentrant contexts. Calling complex code or unloading while callbacks remain possible creates races and crashes.

## 10.6 IAT hooking

An **IAT hook** changes an imported function-address slot so indirect calls from a selected module go to a replacement function.

Conceptually:

1. parse the target module's import structures;
2. identify an exact IAT slot;
3. make the containing page writable temporarily;
4. atomically replace the pointer with an ABI-compatible callback;
5. restore page protection;
6. preserve the original pointer for forwarding/restoration;
7. restore the slot before callback code unloads.

Scope is limited: it affects calls routed through that IAT slot, not every possible call to the function. Direct calls, cached pointers, delay-load paths, or other modules' IATs may bypass it.

The replacement must preserve parameters, return type, calling convention, error behavior, thread safety, and reentrancy expectations. Pointer replacement should be atomic for other threads.

This is different from unsupported kernel table patching, which crosses a much more dangerous protection boundary.

## 10.7 Detection as evidence correlation

No single indicator proves injection. Build a timeline and correlate independent evidence.

Start with stable identity:

`PID + creation time + full path + signer/hash + architecture + user/integrity`

Then inspect:

- loaded module list and unexpected paths;
- unsigned or unexpectedly signed modules;
- module architecture and mapped image type;
- thread start addresses and stacks;
- private executable memory or unusual `RW -> RX` / `RWX` regions;
- cross-process handle access;
- image-load, process/thread, and memory-related telemetry if available;
- network/file/Registry activity around the same time;
- differences from a known-good baseline.

Interpretations must stay qualified:

- signed does not mean harmless;
- unsigned does not automatically mean malicious;
- private executable memory may belong to a JIT;
- module enumeration can miss manually mapped or transient code;
- tools observe different layers and can have access/coverage gaps.

Useful tools:

- **Process Explorer:** live processes, tokens, threads/stacks, modules, handles;
- **ListDLLs:** module inventory;
- **VMMap:** image/mapped/private regions and protections;
- **Process Monitor:** time-ordered file/Registry/process/thread/image-load evidence;
- **Sigcheck:** hashes, version, signatures, certificate information.

Preserve evidence before taking disruptive action. Terminating a process may destroy the timeline, memory, handles, and connection state needed to understand what happened.


---

<a id="section-11-high-value-distinctions"></a>
# [11. High-value distinctions](#toc-section-11-high-value-distinctions)

These pairs are common sources of confusion. Rewrite each contrast from memory.

| Concept A | Concept B | Essential difference |
|---|---|---|
| program | process | stored image versus live OS container |
| process | thread | resource/isolation container versus execution path |
| PID | process handle | current numeric identifier versus checked reference carrying rights |
| object name | handle | discoverability versus process-local authorized reference |
| close handle | terminate object execution | release one reference versus stop running code |
| concurrency | parallelism | overlapping progress versus simultaneous execution |
| ready | waiting | wants CPU versus blocked on a condition |
| join | cancellation | wait for finish versus request/force finish |
| priority | affinity | order among ready work versus allowed processors |
| virtual address | physical frame | process-relative name versus RAM storage unit |
| reserve | commit | claim address range versus promise backing |
| commit | resident | backing guarantee versus currently in RAM |
| page fault | hard fault | translation needs OS handling versus storage read required |
| CPU cache | file cache | hardware memory cache versus OS file-data cache |
| page size | allocation granularity | mapping/protection unit versus reservation alignment |
| RVA | VA | offset from image base versus actual process address |
| raw file offset | RVA | location on disk versus position in mapped image |
| static linking | dynamic linking | incorporate code at link time versus resolve external module at load/runtime |
| import table | export table | required external symbols versus published symbols |
| configuration | service status | durable SCM record versus current runtime transition/state |
| WoW64 redirection | UAC virtualization | architecture compatibility view versus legacy unelevated-write compatibility |
| authentication | authorization | establish identity versus decide permitted action |
| SID | account name | authorization identity versus human-readable translation |
| access right | privilege | authority on an object handle versus token capability for special operations |
| DACL | SACL | discretionary permissions versus audit/mandatory policy information |
| null DACL | empty DACL | unrestricted discretionary access versus no discretionary grants |
| elevation | kernel mode | stronger user-mode token versus processor privilege mode |
| mutex | semaphore | owned mutual exclusion versus unowned count of permits |
| event | message | condition signal versus application data |
| release lock | close handle | give up ownership/permit versus release object reference |
| deadlock | starvation | dependency cycle stops group versus one participant never wins |
| anonymous pipe | named pipe | inherited related-process channel versus discoverable server endpoint |
| shared backing | shared address | same underlying bytes versus same pointer value (not guaranteed) |
| injection | hooking | add code/data to context versus redirect behavior path |
| signature | trust verdict | verified publisher/integrity evidence versus full behavioral safety |

