# Atlas: Programming for Persistent Memory

## Description

Atlas enables conventional multi-threaded C/Pthreads software to
employ persistent memory with crash resilience: Atlas guarantees that
failures due to causes such as power outages, OS kernel panics, and
application process crashes do not corrupt or destroy application
data in persistent memory.

## Master Source

https://github.com/hewlettpackard/atlas

## Maturity

Beta: Atlas is well-tested open source software; in particular, Atlas
has been tested extensively on HPE ProLiant Gen9 servers with
NVDIMM-based persistent memory.

## Dependencies

- Currently, we support only x86-64 CPUs
- We assume Linux OS.  Linux `tmpfs` must be supported.  Testing has
  been done on RedHat and Ubuntu.
- We assume modern C/C++ compilers in the build environment that must
  support C/C++11.
- The default compiler used in the build is clang.  Testing has been
  done with version 3.6.0 or later.  The instrumentation support is
  currently available only with clang/LLVM.  The runtime should build
  with any compiler supporting C/C++11 though clang is preferred for
  uniformity purposes.
- cmake version 3.1 or later
- boost library
- bash 4.0

For Ubuntu 16.04, these dependencies can be installed with:
```
sudo apt-get install llvm clang cmake libboost-graph-dev
```

- ruby (for certain test scripts)

## Usage

Complete installation instructions and documentation on all Atlas
APIs is readily available via the [Atlas GitHub
site](https://github.com/HewlettPackard/Atlas/blob/master/README.md);
additional [extensive API documentation](https://hewlettpackard.github.io/Atlas/runtime/doc/)
is also available.  Here we provide an overview of how Atlas
operates.

Applications that use Atlas create persistent memory regions using
[persistent region APIs](https://github.com/HewlettPackard/Atlas/blob/master/runtime/include/atlas_alloc.h).
Persistent memory is allocated from a persistent memory region using
interfaces similar to `malloc()/free()`.  An interface also allows
the application to manipulate a root pointer within a persistent
memory region; all in-use application data within the persistent
memory region must be reachable by the application starting from
the region's root pointer.

Atlas also provides [consistency APIs](https://github.com/HewlettPackard/Atlas/blob/master/runtime/include/atlas_api.h)
that enable applications to update data in persistent memory regions
from one consistent state to the next even in the presence of
crashes: Atlas provides durable sections for single-threaded code and
supports failure-atomic critical sections for multi-threaded code.
The latter is the intended mode of use; Atlas automatically infers
and enforces failure-atomicity within critical sections.

The following is a small but complete example program that
manipulates a singly-linked list within critical sections.
Here it has been edited slightly for clarity; the original
is available in the Atlas distribution as
`/runtime/tests/data_structures/sll_nvm.cpp`.

```
#include <alloca.h>
#include <assert.h>
#include <complex>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/time.h>

#include "atlas_alloc.h"
#include "atlas_api.h"

typedef struct Node { char *Data; struct Node *Next; } Node;
typedef struct SearchResult { Node *Match; Node *Prev; } SearchResult;
Node *SLL = NULL;
Node *InsertAfter = NULL;

uint32_t sll_rgn_id; // ID of Atlas persistent region

Node *createSingleNode(int Num, int NumBlanks) {
    Node *node = (Node *)nvm_alloc(sizeof(Node), sll_rgn_id);
    int num_digits = log10(Num) + 1;
    int len = num_digits + NumBlanks;
    char *d = (char *)nvm_alloc((len + 1) * sizeof(char), sll_rgn_id);
    node->Data = d;
    char *tmp_s = (char *)alloca(
        (num_digits > NumBlanks ? num_digits + 1 : NumBlanks + 1) *
        sizeof(char));
    sprintf(tmp_s, "%d", Num);
    memcpy(node->Data, tmp_s, num_digits);
    for (int i = 0; i < NumBlanks; ++i) {
        tmp_s[i] = ' ';
    }
    memcpy(node->Data + num_digits, tmp_s, NumBlanks);
    node->Data[num_digits + NumBlanks] = '\0';
    return node;
}

Node *insertPass1(int Num, int NumBlanks, __attribute__((unused)) int NumFAI) {
    Node *node = createSingleNode(Num, NumBlanks);
    // In pass 1, the new node is the last node in the list
    node->Next = NULL;
    NVM_BEGIN_DURABLE();
    if (!SLL) {
        SLL = node; // write-once
        InsertAfter = node;
    } else {
        InsertAfter->Next = node;
        InsertAfter = node;
    }
    NVM_END_DURABLE();
    return node;
}

Node *insertPass2(int Num, int NumBlanks, int NumFAI) {
    Node **new_nodes = (Node **)malloc(NumFAI * sizeof(Node *));
    Node *list_iter = InsertAfter->Next;
    for (int i = 0; i < NumFAI; i++) {
        new_nodes[i] = createSingleNode(Num + i, NumBlanks);
        if (list_iter != NULL) {
            new_nodes[i]->Next = list_iter;
        } else {
            break;
        }
        list_iter = list_iter->Next;
    }
    NVM_BEGIN_DURABLE();    // Atlas failure-atomic region
    for (int i = 0; i < NumFAI; i++) {
        InsertAfter->Next = new_nodes[i];
        if (new_nodes[i]->Next != NULL) {
            InsertAfter = new_nodes[i]->Next;
        } else {
            InsertAfter = new_nodes[i];
            break;
        }
    }
    NVM_END_DURABLE();
    return InsertAfter;
}

long printSLL() {
    long sum = 0;
    Node *tnode = SLL;
    while (tnode) {
        sum += atoi(tnode->Data);
        tnode = tnode->Next;
    }
    return sum;
}

int main(int argc, char *argv[]) {
    struct timeval tv_start;
    struct timeval tv_end;
    gettimeofday(&tv_start, NULL);
    if (argc != 4) {
        fprintf(stderr, "usage: a.out numInts numBlanks numFAItems\n");
        exit(0);
    }
    int N = atoi(argv[1]);
    int K = atoi(argv[2]);
    int X = atoi(argv[3]);
    fprintf(stderr, "N = %d K = %d X = %d\n", N, K, X);
    assert(!(N % 2) && "N is not even");
    assert(X > 0);
    // Initialize Atlas
    NVM_Initialize();
    // Create an Atlas persistent region
    sll_rgn_id = NVM_FindOrCreateRegion("sll_ll", O_RDWR, NULL);
    int i;
    for (i = 1; i < N / 2 + 1; ++i) {
        insertPass1(i, K, X);
    }
    InsertAfter = SLL;
    for (i = N / 2 + 1; i < N; i += X) {
        insertPass2(i, K, X);
    }
    int total_inserted = i - 1;
    insertPass2(i, K, N - total_inserted);
    fprintf(stderr, "Sum of elements is %ld\n", printSLL());
    // Close the Atlas persistent region
    NVM_CloseRegion(sll_rgn_id);
    // Atlas bookkeeping
    NVM_Finalize();
    gettimeofday(&tv_end, NULL);
    fprintf(stderr, "time elapsed %ld us\n",
            tv_end.tv_usec - tv_start.tv_usec +
                (tv_end.tv_sec - tv_start.tv_sec) * 1000000);
    return 0;
}
```

## Notes

When Atlas is used with Ext4/DAX on NVDIMMs, "lazy allocation"
features of the file system (e.g., sparse files) should not be used.

## See Also

- [OOPSLA 2014 paper on Atlas](http://dl.acm.org/citation.cfm?id=2660224)
- [Managed Data Structures (MDS)](README-MDS.md)
