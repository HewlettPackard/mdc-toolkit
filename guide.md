# Newcomers' Guide to the Memory Driven Computing Toolkit

<!---Author:  terence.p.kelly@hpe.com-->

This document describes the opportunities and challenges of [Memory
Driven Computing](https://www.labs.hpe.com/next-next/mdc) (MDC); introduces the Hewlett Packard Labs MDC
Toolkit, which helps programmers to realize the full benefits of this
emerging computing paradigm; and discusses persistent memory and the
new programming style that it invites.

## Memory Driven Computing

[Memory Driven Computing (MDC)](https://www.hpe.com/us/en/newsroom/news-archive/press-release/2016/11/1287610-hewlett-packard-enterprise-demonstrates-worlds-first-memory-driven-computing-architecture.html) takes advantage of emerging technologies such as universal
memory, photonics, and system-on-chip (SoC) processors, to differentiate itself from 
conventional computing architectures.  Conventional
computing centers on processors.  Memory driven computing shifts
the emphasis to data in memory: instead of bringing data to processors,
MDC brings processing to data.  MDC allows us to ingest, store, and
manipulate massive data sets while simultaneously reducing energy/bit
by orders of magnitude.

## Fabric-Attached Memory

Fabric-Attached Memory (FAM) promises to scale shared
byte-addressable memory far beyond today's cache-coherent shared
memories.  FAM offers shared access to *non*-cache-coherent memory, 
for massive numbers of system-on-chip (SoC) processors, accessing the memory
over a memory fabric.  FAM also provides the necessary atomic primitives
and cache flushing operations to enable software to share
access to FAM *safely*.

## Persistent Memory

*Persistent memory* refers to memory whose contents outlive
the processes that allocate, populate, and manipulate the memory. 
Persistence must be defined with respect to
events that terminate processes.  For example, memory that is
persistent with respect to mere process termination, normal or
otherwise, is readily available on today's conventional computers
running conventional operating systems (via file-backed memory
mappings).  By contrast, persistence with respect to operating system
crashes or power outages requires additional hardware and/or software
support.

Persistent memory is not the same as *non-volatile memory* (NVM); the
latter term denotes memory device technologies (e.g., Memristor) that
retain data in the absence of continuously supplied power.  NVM can
facilitate the implementation of memory that is persistent with
respect to certain failures (e.g., power outages), but persistent memory can be
implemented without NVM.  For example, current [HPE ProLiant Persistent Memory 
Servers](https://www.hpe.com/us/en/servers/persistent-memory.html) 
employ NVDIMMs based on conventional volatile DRAM with
flash storage and sufficient standby power to copy the contents of
the DRAM to flash in the event that utility power is lost. While this
is an implementation of persistent memory, it is not based on emerging NVM
technologies like Memristors. The MDC Toolkit is concerned with persistence
rather than the specific memory technologies that implement persistence.

## Persistent Memory Programming

Today's conventional applications manipulate persistent data
indirectly, via file system interfaces (open/close/read/write) or
database interfaces (SQL or put/get) that invoke complex layers of
intermediate software between the application and the persistent data
on block-addressable storage (hard disks or solid-state drives).

Legacy systems can still use filesystem and database system APIs to access data
as storage in persistent memory, but they also still pay the overheads of invoking those 
complex layers of intermediate software.  The alternative is to use persistent memory 
directly as byte-addressable memory.

In "persistent memory programming", applications manipulate
persistent data directly, in-place, using CPU instructions (LOAD and
STORE).  Persistent memory programming offers two main advantages over the
conventional alternative: greatly simplified application software due
to the elimination of separate data formats for memory and storage;
and greatly improved performance due to the elimination of large,
complex intermediate software layers such as file systems and
relational database management systems.

Hewlett Packard Labs has developed a range of technologies to help
our customers realize the full value of MDC and persistent memory.  Our
technologies address the major challenges surrounding MDC and persistent memory:
data organization, messaging, protecting the integrity of data from
failures, allocating and managing memory, and predicting application
performance.

One major concern surrounding persistent memory style programming involves the
vulnerability of application data in persistent memory to corruption
or loss due to failure.  Application data in memory is deemed
*consistent* if it satisfies whatever application-level invariants or
integrity constraints are required by the applications that access
it.  Consistency is orthogonal to persistence - even if memory is
*persistent* its contents need not be *consistent*.  The main worry
about persistent memory programming is that a failure (e.g., process crash,
OS kernel panic, or power outage) that occurs while an application is
updating its persistent data may corrupt or destroy the data.  To
realize the full benefits of persistent memory programming, application
developers require mechanisms to update data in persistent memory from one
consistent state to the next even in the presence of failures.

Another requirement for persistent memory programming is mechanisms that
allow applications, administrators, and end users to allocate and
manage persistent memory.

## Hewlett Packard Labs Technologies

![](Layer_cake.PNG?raw=true)

### Persistent Memory Programming

[Atlas](README-Atlas.md) enables conventional multi-threaded C/Pthreads software to employ persistent memory with crash resilience: Atlas guarantees that failures due to causes such as power outages, OS kernel panics, and application process crashes do not corrupt or destroy application data in persistent memory.

[NVthreads](README-NVthreads.md) is a drop-in replacement for the popular pthreads library to make existing multi-threaded programs crash tolerant. The NVthreads library tracks data at memory page granularity to achieve good performance and allows applications to resume execution from a crash point.

### Data Organization

The [Radix Tree](README-Radix-Tree.md) is a user-space library that
implements a radix tree based on FAM atomic instructions.  The Radix
Tree is a space-optimized trie suitable for efficiently storing 
and searching text and intrinsically hierarchical data; it supports
transactional updates by multiple non-coherent SoCs in a shared FAM
environment.

<!---
### Messaging

[FAM-Messaging](README-fam-messaging.md) is an efficient
implementation for inter-process communication via a shared memory
ringbuffer.  On the sender side, the implementation provides a simple
asynchronous send interface; receivers block until a message arrives.
The implementation supports concurrent reads and writes.
-->

### Log, Checkpoint, and Concurrent Transactions

Technologies that enable applications to log, checkpoint, and manipulate data with persistent memory include
[libnvwal](README-libnvwal.md), [CRIU-PMEM](README-CRIU-PMEM.md), [Managed Data Structures (MDS)](README-MDS.md), and [FOEDUS](README-FOEDUS.md).

libnvwal is a user-space library for applications to manage write-ahead log on persistent memory
(e.g., NVDIMM). CRIU-PMEM is an application-transparent, system level checkpointing tool using persistent memory. 
<!--- Atlas is for both new and existing multi-threaded applications
written in C/Pthreads.  FOEDUS is a database engine roughly similar in operation to Berkeley DB but optimized for
large-memory manycore machines. Ken is a platform for fault-tolerant
distributed computing; it provides a persistent heap that supports
Pmem-style programming even on conventional hardware with disk-based
durability.  libnvwal uses Pmem to accelerate performance for
write-ahead logging in database engines such as MySQL.-->
The Managed Data Structures (MDS) library delivers a simple, high-level programming model, 
which supports multi-threaded, multi-process creation, use and sharing of data structures in persistent memory, 
via APIs in multiple programming languages, Java and C++. FOEDUS is a database engine roughly similar in operation to Berkeley DB but optimized for large-memory manycore machines.
 

| Tool      | Targeted hardware | Intended use                                             | Next release features |
| ---       | ---                   | ---                                                      | ---                   |
| libnvwal  | ProLiant Gen10 persistent memory | efficient write-ahead-logging in persistent memory | None |
| CRIU-PMEM | persistent memory, DRAM, HDD or SSD  | application-transparent system level checkpointing       | remote checkpointing |
| MDS       | ProLiant Gen10 persistent memory, mmapped file on HDD/SSD  | convenient concurrent transactions on data structures in persistent memory | more data structures, larger scale |
| FOEDUS      | large persistent memory and manycore machines   | embeded database engine | None | 



### Memory Allocation and Management

Technologies for memory management/allocation include
[ALPS](README-ALPS.md),
[MPGC](README-MPGC.md),
[Shoveller](README-Shoveller.md), and
[NVMM](README-NVMM.md).
ALPS provides a low-level abstraction layer that relieves the user
from the details of mapping, addressing, and allocating shared PMEM.
MPGC is a multi-process garbage collector for use
with MDS or standalone; it addresses the special challenges of garbage collection
of memory shared among independently developed processes. Shoveller is
a scalable-parallel-log-structued memory
allocator for key-value
stores.  NVMM is a library built atop the Librarian File System
(LFS) that provides simple abstractions for accessing and allocating
PMEM.

| Features             | ALPS                  | MPGC                 | Shoveller     | NVMM       |
| ---                  | ---                   | ---                  | ---         | ---        |
| Single/multi-shelf   | Single                | Single               | N/A         | Multi      |
| Single/multi-process | Multi                 | Multi                | Multi       | Multi      |
| Single/multi-node    | see Note 1            | Single               | Single      | Multi      |
| Crash resilient?     | Yes                   | Yes                  | No          | WIP        |
| Allocation size      | Variable              | Variable             | Variable    | Variable   |
| Data structures      | Volatile + persistent | Persistent + lock-free | Volatile    | Persistent |
| APIs                 | Alloc/Free            | Alloc                | Get/Put/Del | Alloc/Free |
| Garbage collector    | No                    | Yes, online          | NO          | No         |
| De-fragmentation     | No                    | No                   | Yes         | No         |
| Grow capacity        | No                    | Yes                  | No          | Yes        |
| Targeted hardware    | CC-NUMA, FAM          | persistent memory    |large scale machines (e.g, SuperDomeX)| FAM, persistent memory |
| Intended use         | memory allocation     | memory management for persistent memory |key-value store|memory allocation for FAM| 
| Next release features | none                 | performance, scale   |none | crash recovery|

Note 1:  Multi for allocation, no remote free

<!---### Performance Emulation

[Quartz](README-Quartz.md) is a performance emulator for NVM.  Quartz
enables an efficient emulation of a wide range of NVM latencies and
bandwidth characteristics for performance evaluation of emerging
byte-addressable NVMs and their impact on application performance
(without modifying or instrumenting their source code) by leveraging
features available in commodity hardware.
-->

## Learn more

The MDC Toolkit is part of [an initiative by HPE to open source software](https://community.hpe.com/t5/Behind-the-scenes-Labs/Discover-2016-The-Machine-is-an-open-source-project/ba-p/6865943#.WJrIjPL57Lj) 
so that [developers can start exploring what it means to program for the Memory-Driven Computing architecture](https://www.labs.hpe.com/the-machine/the-machine-distribution), 
with massive pools of non-volatile memory, a fast memory fabric and task-specific processing.

To learn more, read about [First Steps In The Program Model For Persistent Memory](https://www.nextplatform.com/2016/04/25/first-steps-program-model-persistent-memory/) and other related topics in this series of articles; and try out some of the tools introduced above in this Memory Driven Computing Toolkit.
