# Multi Process Garbage Collector (MPGC)

Author: Susan Spence (susan.spence@hpe.com)

Contributors: Evan Kirshenbaum, Lokesh Gidra,
Susan Spence, Sergei Uversky.

## Description

Our state-of-the-art Multi Process Garbage Collector (MPGC) delivers
automatic memory management for applications, enabling multiple
applications to share objects in memory, while avoiding memory
management errors and memory leaks.  Whereas commercial garbage
collectors for memory management today are typically single process
and they work at the scale of only tens to hundreds of Gigabytes,
MPGC is ground-breaking in its support for multi-process,
fault-tolerant garbage collection, and in scaling to heaps greater
than hundreds of Gigabytes.  The current implementation delivers a
fault-tolerant, on-the-fly, concurrent garbage collector, with zero
stop-the-world pauses.  MPGC is used by HPE's Managed Data Structures
(MDS) library and other non-MDS applications, including applications
written in unmanaged languages like C++.

MPGC runs and has been tested on x86 and ARM architectures.  It can
be used for automatic memory management by multiple application
processes using a shared object heap in shared (volatile) memory.
However, MPGC is really designed to deliver the benefits of automatic
memory management on [The Machine](https://www.labs.hpe.com/the-machine) 
and other [persistent memory architectures](https://www.hpe.com/us/en/servers/persistent-memory.html).

## Source

Releases of the MPGC source code are available open source on GitHub:
https://github.com/HewlettPackard/mpgc

Maturity: Alpha.  

MPGC has been developed as part of the Managed Data
Structures research project at Hewlett Packard Labs.  As such,
releases of the MPGC source code made available on GitHub are not
production quality, but rather "alpha" quality.  The MPGC APIs have
been specified and implemented, but MPGC is not functionally
complete.  MPGC has gone through basic testing, but may be unstable
and could cause crashes or data loss.

MPGC is an active research project; we intend to make further
releases with more and/or better features and bug fixes as the
project continues.

## License

The Multi Process Garbage Collector is distributed under the LGPLv3 license 
with an exception.
See license files [COPYING](https://github.com/HewlettPackard/mpgc/blob/master/COPYING) and [COPYING.LESSER](https://github.com/HewlettPackard/mpgc/blob/master/COPYING.LESSER).

## Dependencies

No dependencies.

## Usage

A User's Guide to the Multi Process Garbage Collector is available on
the MPGC GitHub site:<br>
https://github.com/HewlettPackard/mpgc/blob/master/doc/MPGC-User-Guide.pdf

This user guide describes in detail how C++ applications can create
and use data in an MPGC-managed heap, and share data between
processes, without having to determine when it is safe to free up
shared memory, and without risk of corrupting the heap if a process
crashes.

For instructions on installing MPGC, see [INSTALL.md](https://github.com/HewlettPackard/mpgc/blob/master/INSTALL.md).

## Notes

**Limitations**: Note that the fault-tolerant aspect of MPGC assumes
  that it is running in a system with persistent memory where caches
  are flushed on failure; otherwise, we do not guarantee that the
  heap will not be corrupted as a result of failures.

## See Also

The Managed Data Structures software library, which uses MPGC for its
automatic memory management:<br>https://github.com/HewlettPackard/mds
