
# FOEDUS: Fast Optimistic Engine for Data Unification Services

## Description

FOEDUS is a transactional key-value store that is optimized for a
large number of CPU cores and persistent memory (or a fast SSD).  It
is a C++ library that you can either include in your source code (by
invoking a CMake script) or dynamically link to.  In a nutshell, it
is something like BerkeleyDB, but it is much more efficient on new
hardware.

## Master Source

https://github.com/hewlettpackard/foedus

## Maturity

Alpha; see [warning](https://github.com/hewlettpackard/foedus_code).

## Dependencies

- Only 64-bits CPUs, specifically, x86_64 and ARMv8.
- Linux/Unix so far; no MacOS, Windows, nor Solaris.
  FOEDUS development was done on Centos 7.2.
- Linux kernel 3.16 or later (for performance reasons).
- FOEDUS assumes that `fsync(2)` works correctly;
  check the write cache in your storage devices.
- Reasonably modern C++ compilers, namely gcc 4.8.2 or later.
  Experimental support for clang is present.
- CMake
- glog
- tinyxml2
- libbacktrace
- gperftools
- papi
- libnuma
- glibc/stdc++
- valgrind
- xxHash
- Sufficient non-transparent hugepages must be allocated.

Additional information on configuration is available
[here](https://github.com/hewlettpackard/foedus_code/tree/master/foedus-core).

## Usage

Applications use FOEDUS by linking with the FOEDUS library.
Extensive documentation on the APIs exposed by the library is
available from the [FOEDUS GitHub software
repository](https://github.com/hewlettpackard/foedus_code).

An instructive lengthy example of FOEDUS usage is the [FOEDUS implementation
of the YCSB
benchmark](https://github.com/hewlettpackard/foedus_code/blob/master/experiments-core/src/foedus/ycsb/ycsb_driver.cpp)

The following simple program illustrates very basic FOEDUS usage:
```
#include <iostream>

#include "foedus/engine.hpp"
#include "foedus/engine_options.hpp"
#include "foedus/epoch.hpp"
#include "foedus/proc/proc_manager.hpp"
#include "foedus/storage/storage_manager.hpp"
#include "foedus/storage/array/array_metadata.hpp"
#include "foedus/storage/array/array_storage.hpp"
#include "foedus/thread/thread.hpp"
#include "foedus/thread/thread_pool.hpp"
#include "foedus/xct/xct_manager.hpp"

const uint16_t kPayload = 16;
const uint32_t kRecords = 1 << 20;
const char* kName = "myarray";
const char* kProc = "myproc";

foedus::ErrorStack my_proc(const foedus::proc::ProcArguments& args) {
  foedus::thread::Thread* context = args.context_;
  foedus::Engine* engine = args.engine_;
  foedus::storage::array::ArrayStorage array(engine, kName);
  foedus::xct::XctManager* xct_manager = engine->get_xct_manager();
  WRAP_ERROR_CODE(xct_manager->begin_xct(context, foedus::xct::kSerializable));
  char buf[kPayload];
  WRAP_ERROR_CODE(array.get_record(context, 123, buf));
  foedus::Epoch commit_epoch;
  WRAP_ERROR_CODE(xct_manager->precommit_xct(context, &commit_epoch));
  WRAP_ERROR_CODE(xct_manager->wait_for_commit(commit_epoch));
  return foedus::kRetOk;
}

int main(int argc, char** argv) {
  foedus::EngineOptions options;
  foedus::Engine engine(options);
  engine.get_proc_manager()->pre_register(kProc, my_proc);
  COERCE_ERROR(engine.initialize());

  foedus::UninitializeGuard guard(&engine);
  foedus::Epoch create_epoch;
  foedus::storage::array::ArrayMetadata meta(kName, kPayload, kRecords);
  COERCE_ERROR(engine.get_storage_manager()->create_storage(&meta, &create_epoch));
  foedus::ErrorStack result = engine.get_thread_pool()->impersonate_synchronous(kProc);
  std::cout << "result=" << result << std::endl;
  COERCE_ERROR(engine.uninitialize());

  return 0;
}
```

## Notes

N/A

## See Also

- [SIGMOD 2015 paper on FOEDUS](https://www.labs.hpe.com/techreports/2015/HPL-2015-37.pdf)
