# ALPS:  Allocator Layer for Persistent Shared Memory

Author:  Haris Volos (haris.volos@hpe.com)

## Description

ALPS provides a low-level abstraction layer that relieves the user from
the details of mapping, addressing, and allocating shared persistent memory.
Shared persistent memory refers to non-volatile memory shared among
multiple compute nodes and can take different forms, such as
disaggregated non-volatile memory accessible over a fabric (also
known as fabric-attached memory or FAM) or multi-socket non-volatile
memory.

The main abstraction provided by ALPS is a Global Symmetric Heap that
lets users allocate variable-size chunks of persistent memory through
the familiar `malloc()/free()` interface that can be shared among
multiple concurrent processes.  For example, we extended Spark with
an optimized in-memory shuffle implementation where workers produce
and consume data directly from the shared heap instead of exchanging
data through network I/O.

## Master Source

Public: https://github.com/hvolos/alps

HPE internal: https://github.hpe.com/labs/alps

## Maturity

Alpha release

## Quick Start Guide

This section provides a quick introduction to setting up and testing
ALPS on a CC-NUMA machine.
We assume the ALPS source code is already deployed in directory $ALPS.

1. Change into the ALPS source directory:

 ```
 $ cd $ALPS
 ```

2. Install dependencies:

 ```
 $ ./install-dep
 ```

3. Build ALPS

 ```
 $ mkdir build
 $ cd build
 $ cmake .. -DTARGET_ARCH_MEM=CC-NUMA
 $ make
 ```

4. Run unit tests against tmpfs (located at: /dev/shm):

 ```
 ctest -R tmpfs
 ```

## Dependencies

Dependencies for different platforms and environments is available on a platform
by platform basis:

* [CC-NUMA](https://github.hpe.com/labs/alps/blob/master/INSTALL-NUMA.md): Linux platform on Cache-Coherent Non-Uniform Memory
Access (CC-NUMA) hardware architecture
* [FAM](https://github.hpe.com/labs/alps/blob/master/INSTALL-FAM.md): Linux for The Machine (L4TM) on Fabric-Attached Memory (FAM)
hardware architecture


## Installation

Instructions for building and installing ALPS on different platforms and
environments is available on a platform by platform basis:

* [CC-NUMA](https://github.hpe.com/labs/alps/blob/master/INSTALL-NUMA.md): Linux platform on Cache-Coherent Non-Uniform Memory
Access (CC-NUMA) hardware architecture
* [FAM](https://github.hpe.com/labs/alps/blob/master/INSTALL-FAM.md): Linux for The Machine (L4TM) on Fabric-Attached Memory (FAM)
hardware architecture

## Usage

The interface exposes a _shared heap_ abstraction that provides users
with a shared heap memory allocator.  The allocator lets users
allocate variable-size chunks of physically shared memory through the
familiar `malloc()/free()` interface.  We extend the `malloc`
interface to take an extra memory-attributes argument that can be
used to provide to the allocator a hint about the desirable
properties of the allocated memory.  For example, a user can use this
hint to express locality requirements by stating the NUMA node to
allocate memory from:

```
void* malloc(size_t size, int numa_node_hint);
```

The allocator interface uses relocatable base-relative C++ smart
pointers for logically addressing (naming) allocated blocks.  In
contrast to virtual addresses, our pointers are relocatable meaning
that the shared heap does not have to be memory mapped into the same
virtual address range in each process using the heap.  Instead, a
logical address represents an offset relative to the base of the
virtual address region that maps the shared heap so that the heap can
be memory mapped to different virtual address regions in each
process.

The interface also provides support for fault tolerance necessary to
big-data analytic frameworks.  When opening a heap, a user can ask
for a generation number identifying the current instance of the heap.
Blocks associated with a generation number can be released in two
ways: (1) explicit call to free the memory block, or (2) a bulk-free
call that frees all the blocks associated with a given generation.
Deallocating memory through a bulk-free is useful for releasing
memory when recovering from a crash.

For example code snippets please refer to:

https://github.com/hvolos/alps/blob/master/examples/globalheap

## Notes:

No support for remote free: i.e., a process cannot free memory
allocated by another concurrently active process.

## See Also:

- [Managed Data Structures (MDS)](README-MDS.md)
- [Multi-Process Garbage Collector (MPGC)](README-MPGC.md)
