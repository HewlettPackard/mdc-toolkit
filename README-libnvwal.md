# libnvwal : Write-Ahead-Logging Library for Non-Volatile DIMMs

Author: Haris Volos (haris.volos@hpe.com)

## Description

Write-Ahead-Logging (WAL) is the central component in various software that require 
atomicity and durability, such as data management systems (DBMS).
The durability and low-latency of non-volatile DIMM (NVDIMM) is a great fit for WAL.
This library, _libnvwal_, is a handy user-space library for such software to manage
WAL on NVDIMM with little effort.

The library offers the following benefits:
* Without libnvwal, individual applications must implement their own layering
logics on NVDIMM and Disk (HDD/SSD) to store WAL files larger than NVDIMM.
* This would result in either 1) significant code changes on WAL,
which is usually one of the most complex and critical modules,
or 2) inefficiency via filesystem-level abstraction, which cannot
exploit the unique access pattern of WAL.
* libnvwal solves the issue for a wide variety of DBMS.


## Master Source

https://github.com/HewlettPackard/libnvwal

## Maturity

libnvwal is in beta state. 

## Dependencies

NVML: http://github.com/pmem/nvml

## Usage

The main abstraction provided by the library is a write-ahead log
(WAL).  User writer threads perform writes to the log through an
intermediate volatile circular buffer they register with the library.
They inform the library about these new writes by invoking the
`nvwal_on_wal_write()` API that receives an epoch of the written
logs.  An epoch represents a user-defined duration of time and it is
a central concept in libnvwal.  Depending on the type of client
applications, it might be just one transaction, 10s of milliseconds,
or something else.  An epoch contains some number of log entries,
ranging from 1 to millions or more.  Each log entry always belongs to
exactly one epoch, which represents when the log is written and
becomes durable.

The library relies on another two threads to collect and propagate
the writes to the persistent log: (1) a flusher thread that is
responsible for monitoring log activities of each log writer, writing
them out to NVDIMM, and letting the writers know when they can
reclaim their buffers, and (2) a syncer thread that is responsible
for synchronizing (via `fsync()`) the log in parallel to flusher so
that the flusher can maximize its use of time.

For an example usage of the library API, please refer to the example
folder in the master repository.

For instructions on compiling the library and viewing the
documentation, please refer to the main README file available in the
master repository.

## Notes

N/A

## See also

N/A

