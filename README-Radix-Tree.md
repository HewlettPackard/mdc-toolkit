# Radix Tree

Contact: Yupu Zhang (yupu.zhang@hpe.com)

## Description
Radix Tree is a user-space library written in C++ that implements a radix tree that relies on
fabric-attached memory (FAM) atomics, which are atomics primitives within a cache-incoherent
memory-semantic fabric environment.

A radix tree (also known as a radix trie) is a space-optimized trie data structure, where the number
of children of every internal node is at least the radix r of the radix tree, where r is a positive
integer and a power of 2. Unlike in regular tries, edges can be labeled with sequences of elements
as well as single elements, which makes radix trees much more efficient for small sets (especially
if the strings are long) and for sets of strings that share long prefixes. As a result, radix trees
are frequently used for efficiently storing and searching text and intrinsically hierarchical data.

FAM atomic operations permit programs to perform operations such as write, compare-and-store
(analogous to compare-and-swap), or fetch-and-add on operands located in fabric-attached memory
atomically. Use of these atomics permits programs running on multiple non-coherent SoCs in a shared
FAM environment to transactionally update the radix tree in a controlled fashion.

Radix tree relies on the FAM-aware Non-Volatile Memory Manager (NVMM) library, which is available
from https://github.com/HewlettPackard/gull.

## Master Source

https://github.com/HewlettPackard/meadowlark

## Maturity
Radix Tree is still in alpha state. The basic functionalities are working, but performance is
not optimized. One limitation of the current implementation is that the key length is fixed and can
only be tuned at compile time.

Radix Tree runs on both NUMA and FAME systems.

## Dependencies

- Install additional packages

  ```
  $ sudo apt-get install build-essential cmake libboost-all-dev
  ```

- Install libpmem

  ```
  $ sudo apt-get install autoconf pkg-config doxygen graphviz
  $ git clone https://github.com/FabricAttachedMemory/nvml.git
  $ cd nvml
  $ make
  $ sudo make install
  ```

- Install libfam-atomic

  ```
  $ cd ~
  $ sudo apt-get install autoconf autoconf-archive libtool
  $ sudo apt-get --no-install-recommends install asciidoc xsltproc xmlto
  $ git clone https://github.com/FabricAttachedMemory/libfam-atomic.git
  $ cd libfam-atomic
  $ bash autogen.sh
  $ ./configure
  $ make
  $ sudo make install
  ```

- Install [NVMM](https://github.com/HewlettPackard/gull/blob/master/README.md)

  Radix Tree uses Non-Volatile Memory Manager (NVMM). Before building the radix tree, please
  download and build NVMM.

- Setup [FAME](https://github.com/HewlettPackard/mdc-toolkit/blob/master/guide-FAME.md) if you want to try Radix Tree on top of FAM

## Build & Test

1. Download the source code:

 ```
 $ git clone https://github.com/HewlettPackard/meadowlark.git
 ```

2. Change into the source directory (assuming the code is at directory $RADIXTREE):

 ```
 $ cd $RADIXTREE
 ```

3. Point build system to NVMM (assuming NVMM is already compiled with or without FAME support)
   
   Set the environment variable ${CMAKE_PREFIX_PATH} to include the 
   paths to the NVMM header and library files. 

   For example, if NVMM is installed at /home/${USER}/projects/nvmm:

   ```
   $ export CMAKE_PREFIX_PATH=/home/${USER}/projects/nvmm/include:/home/${USER}/projects/nvmm/build/src
   ```

4. Build

 On CC-NUMA systems:

 ```
 $ mkdir build
 $ cd build
 $ cmake .. -DFAME=OFF
 $ make
 ```

 On FAME:

 ```
 $ mkdir build
 $ cd build
 $ cmake .. -DFAME=ON
 $ make
 ```

 The default build type is Release. To switch between Release and Debug:
 ```
 $ cmake .. -DCMAKE_BUILD_TYPE=Release
 $ cmake .. -DCMAKE_BUILD_TYPE=Debug
 ```

5. Test

 ```
 $ make test
 ```
 All tests should pass.

## Demo on FAME

There is a demo program that creates and destroys radix trees, and issue put/get/destroy/list
commands to a radix tree, given its root. Below are the steps to run the demo:

1. Setup [FAME](https://github.com/HewlettPackard/mdc-toolkit/blob/master/guide-FAME.md) with at least two nodes (e.g., node01 and node02)

2. Install NVMM with FAME support on all nodes

3. Install Radix Tree on all nodes at $RADIXTREE, with FAME support

4. First log into one node, create a radix tree, and remember its root pointer ($ROOT_PTR):

 ```
 $ cd $RADIXTREE/build/demo
 $ ./demo_radix_tree 0 create_tree
 ```

5. Then log into any node to exercise put/get/destroy/list commands. For example:

 Log into node01, put a pair of key value: a 1

 ```
 $ cd $NVMM/build/demo
 $ ./demo_radix_tree $ROOT_PTR put a 1
 $ ./demo_radix_tree $ROOT_PTR list
 ```

 Log into node02, get the value of key a:
 ```
 $ cd $NVMM/build/demo
 $ ./demo_radix_tree $ROOT_PTR list
 $ ./demo_radix_tree $ROOT_PTR get a
 ```

## Usage

Please see demo/demo_radix_tree.cc

## Notes
- There is another implementation of RadixTree at branch "numa_radixtree" whose FAM-support is still
work in progress
