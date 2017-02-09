# Radix Tree

Author: Huanchen Zhang

## Description

Radix Tree is a user-space library written in C++ that implements a radix tree that relies on fabric-attached memory (FAM) atomics, which are atomics primitives within a cache-incoherent memory-semantic fabric environment.

A radix tree (also known as a radix trie) is a space-optimized trie data structure, where the number of children of every internal node is at least the radix r of the radix tree, where r is a positive integer and a power of 2. Unlike in regular tries, edges can be labeled with sequences of elements as well as single elements, which makes radix trees much more efficient for small sets (especially if the strings are long) and for sets of strings that share long prefixes. As a result, radix trees are frequently used for efficiently storing and searching text and intrinsically hierarchical data.

FAM atomic operations permit programs to perform operations such as write, compare-and-store (analogous to compare-and-swap), or fetch-and-add on operands located in fabric-attached memory atomically. Use of these atomics permits programs running on multiple non-coherent SoCs in a shared FAM environment to transactionally update the radix tree in a controlled fashion.

Radix Tree incorporates a very simple transaction manager that uses a single global lock to provide
concurrency control between competing transactions. The radix tree relies on the FAM-aware
Non-Volatile Memory Manager (NVMM) library, which is available from https://github.hpe.com/labs/nvmm
or https://github.com/HewlettPackard/gull.

## Master Source

https://github.hpe.com/labs/radixtree (internal)

https://github.com/HewlettPackard/meadowlark (external)

## Maturity

Radix Tree is still in alpha state. The basic functionalities are working, but performance is
not optimized. Known limitations: concurrency control is very simple (i.e., a single global lock),
the garbage collection implementation is incomplete (i.e., nodes with no entries won't be garbage
collected) and updateValue currently only correctly supports unique indexes (i.e., it will only
update the first value for non-unique indexes).

While Radix Tree is designed to work on FAM, currently it only runs on NUMA systems.

## Dependencies

Radix Tree depends on cmake, libboost, libfam-atomic (https://github.com/FabricAttachedMemory/libfam-atomic), and nvml (https://github.com/FabricAttachedMemory/nvml). 

For internal users who have access to an l4tm (Linux for The Machine) system, e.g.,
build-l4tm-X.u.labs.hpecorp.net (X=1,2,3,4). Please install the following dependencies
```
$ sudo apt-get install cmake libboost-all-dev
$ sudo apt-get install libfam-atomic2 libfam-atomic2-dbg libfam-atomic2-dev libpmem libpmem-dev
```
Please email Robert Chapman (robert.chapman@hpe.com) for getting access to build-l4tm-X.u.labs.hpecorp.net.

Radix Tree uses Non-Volatile Memory Manager (NVMM). Before building the radix tree, please
download and build NVMM.

## Build & Test

1. Download the source code:

 Internal:
 ```
 $ git clone git@github.hpe.com:labs/radixtree.git 
 ```

 External:
 ```
 $ git clone https://github.com/HewlettPackard/meadowlark.git
 ```

2. Change into the source directory (assuming the code is at directory $RADIXTREE):

 ```
 $ cd $RADIXTREE
 ```

3. Setup NVMM (assuming NVMM is already compiled)

 Please update NVMM_LIB_PATH in CMakeLists.txt with the NVMM path. For example,
 if NVMM is located at /code/nvmm, then the following line in CMakeLists.txt, 

 ```
 set(NVMM_LIB_PATH "nvmm path" "/home/$ENV{USER}/projects/nvmm")
 ```
 should be changed to
 ```
 set(NVMM_LIB_PATH "nvmm path" "/code/nvmm")
 ```

4. Build

 ```
 $ mkdir build
 $ cd build
 $ cmake ..
 $ make
 ```

5. Test

 ```
 $ make test
 ```
 All tests should pass.

## Usage

Please see `test/test_transaction.cc` and `test/test_radix_tree.cc`

