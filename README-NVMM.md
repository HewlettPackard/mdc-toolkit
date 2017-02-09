# Non-Volatile Memory Manager (NVMM)

Author: Yupu Zhang (yupu.zhang@hpe.com)

## Description

Non-Volatile Memory Manager (NVMM) is a library written in C++ that
provides simple abstractions for accessing and allocating
Non-Volatile Memory (NVM) from Fabric-Attached Memory (FAM).  Built
on top of the Librarian File System (LFS), NVMM manages shelves from
LFS and assigns them unique shelf IDs so that it exposes a persistent
address space (in the form of shelf ID and offset) that enables
universal access and global sharing.  Furthermore, NVMM presents one
or more shelves as a NVM pool and builds abstractions on top of pools
to support various NVM use styles.  Currently NVMM supports direct
memory mapping through **Region** (mmap/unmap) and finer-grained
allocation through **Heap** (alloc/free).

A key feature of NVMM is its *multi-process* and *multi-node*
support.  NVMM is specifically designed to work in cache-incoherent
multi-node environment like The Machine.  The NVM pool layer permits
concurrent shelf creation and deletion (within a pool) from different
processes on different nodes.  The built-in heap implementation
enables any process from any node to allocate and free globally
shared NVM transparently.

Currently, NVMM provides two types of heaps.  The default is a
single-shelf, zone-based allocator that is implemented with FAM
atomics from scratch and is truly shared across multiple
processes/nodes.  The other is a hierarchical allocator that is built
on top of multiple single-shelf heaps (e.g., simple bump allocators)
and appears to be globally shared.  Its purpose is to make
single-process/node heap implementation work across multiple
processes/nodes.

## Master Source

https://github.hpe.com/labs/nvmm (internal)

https://github.com/HewlettPackard/gull (external)

## Maturity

NVMM is still in alpha state. The basic functionalities are working, but performance is not
optimized and crash recovery is still work in progress.

NVMM runs on both NUMA and FAM systems, but the current release is for NUMA only.

## Dependencies

NVMM depends on cmake, libboost, libfam-atomic (https://github.com/FabricAttachedMemory/libfam-atomic), and nvml (https://github.com/FabricAttachedMemory/nvml). 

For internal users who have access to an l4tm (Linux for The Machine) system, e.g.,
build-l4tm-X.u.labs.hpecorp.net (X=1,2,3,4). Please install the following dependencies
```
$ sudo apt-get install cmake libboost-all-dev
$ sudo apt-get install libfam-atomic2 libfam-atomic2-dbg libfam-atomic2-dev libpmem libpmem-dev
```
Please email Robert Chapman (robert.chapman@hpe.com) for getting access to build-l4tm-X.u.labs.hpecorp.net.

## Build & Test

1. Download the source code:

 Internal:
 ```
 $ git clone git@github.hpe.com:labs/nvmm.git
 ```

 External:
 ```
 $ git clone https://github.com/HewlettPackard/gull
 ```

2. Change into the source directory (assuming the code is at directory $NVMM):

 ```
 $ cd $NVMM
 ```

3. Build

 ```
 $ mkdir build
 $ cd build
 $ cmake ..
 $ make
 ```

 The default build type is Release. To switch between Release and Debug:
 ```
 $ cmake .. -DCMAKE_BUILD_TYPE=Release
 $ cmake .. -DCMAKE_BUILD_TYPE=Debug
 ```

 The default heap implementation is zone-based. To switch between the hierarchical heap (ZONE=OFF)
 and the zone heap (ZONE=ON):
 ```
 $ cmake .. -DZONE=ON
 $ cmake .. -DZONE=OFF
 ```

4. Test

 ```
 $ make test
 ```
 All tests should pass.


## Usage

The following code snippets illustrate how to use the Region and Heap APIs. For more details and examples, please refer to the master source.

**Region** (direct memory mapping, see example/region_example.cc)

``` C++
#include "nvmm/error_code.h"
#include "nvmm/log.h"
#include "nvmm/memory_manager.h"

#include "nvmm/heap.h"

using namespace nvmm;

int main(int argc, char **argv)
{
    init_log(off); // turn off logging

    MemoryManager *mm = MemoryManager::GetInstance();

    // create a new 128MB NVM region with pool id 1
    PoolId pool_id = 1;
    size_t size = 128*1024*1024; // 128MB
    ErrorCode ret = mm->CreateRegion(pool_id, size);
    assert(ret == NO_ERROR);

    // acquire the region
    Region *region = NULL;
    ret = mm->FindRegion(pool_id, &region);
    assert(ret == NO_ERROR);

    // open and map the region
    ret = region->Open(O_RDWR);
    assert(ret == NO_ERROR);
    int64_t* address = NULL;
    ret = region->Map(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED, 0, (void**)&address);
    assert(ret == NO_ERROR);

    // use the region
    *address = 123;
    assert(*address == 123);

    // unmap and close the region
    ret = region->Unmap(address, size);
    assert(ret == NO_ERROR);
    ret = region->Close();
    assert(ret == NO_ERROR);

    // release the region
    delete region;

    // delete the region
    ret = mm->DestroyRegion(pool_id);
    assert(ret == NO_ERROR);    
}
```

**Heap** (finer-grained allocation, see example/heap_example.cc)

``` C++
#include "nvmm/error_code.h"
#include "nvmm/log.h"
#include "nvmm/memory_manager.h"

#include "nvmm/heap.h"

using namespace nvmm;

int main(int argc, char **argv)
{
    init_log(off); // turn off logging

    MemoryManager *mm = MemoryManager::GetInstance();

    // create a new 128MB NVM region with pool id 2
    PoolId pool_id = 2;
    size_t size = 128*1024*1024; // 128MB
    ErrorCode ret = mm->CreateHeap(pool_id, size);
    assert(ret == NO_ERROR);

    // acquire the heap
    Heap *heap = NULL;
    ret = mm->FindHeap(pool_id, &heap);
    assert(ret == NO_ERROR);

    // open the heap
    ret =  heap->Open();
    assert(ret == NO_ERROR);

    // use the heap
    GlobalPtr ptr = heap->Alloc(sizeof(int));  // Alloc returns a GlobalPtr consisting of a shelf ID
                                               // and offset
    assert(ptr.IsValid() == true);

    int *int_ptr = (int*)mm->GlobalToLocal(ptr);  // convert the GlobalPtr into a local pointer

    *int_ptr = 123;
    assert(*int_ptr == 123);

    heap->Free(ptr);

    // close the heap
    ret = heap->Close();
    assert(ret == NO_ERROR);

    // release the heap
    delete heap;

    // delete the heap
    ret = mm->DestroyHeap(pool_id);
    assert(ret == NO_ERROR);    
}
```

## Notes

- Crash recovery is still work in progress
- FAM support is still work in progress

## See Also

- [ALPS](README-ALPS.md)
- [MPGC](README-MPGC.md)

