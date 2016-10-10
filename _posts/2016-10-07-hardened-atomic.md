---
layout: post
title: Linux KSPP: HARDENED_ATOMIC
---
## Linux KSPP: HARDENED_ATOMIC
The Linux Kernel Self Protection Project
([KSPP](http://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project)) was
created with a mandate to eliminate classes of kernel bugs.  To date, this work
has largely involved porting features found in
[PaX](https://pax.grsecurity.net/)/[grsecurity](https://grsecurity.net/) to the
mainline Linux kernel.  A few such features, like
[hardened usercopy](https://www.lwn.net), have already been accepted into the
mainline kernel.  

Earlier in 2016 I started [working](https://lwn.net/Articles/668724/) on porting
PAX_REFCOUNT, a
[feature](https://forums.grsecurity.net/viewtopic.php?f=7&t=4173) of PaX that
protects kernel reference counters against overflow.  A team from
[Intel](http://www.0org.org) graciously decided to collaborate with me and
finally we've submitted an [RFC](https://lwn.net/Articles/702640/).  

This page will serve as documentation for HARDENED_ATOMIC and will be updated
as the feature goes through the steps for merging into mainline Linux.  

---
### Why do we need HARDENED_ATOMIC?

#### Use-after-free Bugs
Rather than targeting individual bugs, KSPP aims to eliminate entire categories
of vulnerabilities from the Linux kernel.  The class of vulnerabilities
addressed by HARDENED_ATOMIC is known as use-after-free vulnerabilities.

Use-after-free vulnerabilities are aptly named: they are classes of bugs in
which an attacker is able to gain control of a piece of memory after it has
already been freed and use this memory for nefarious purposes:
(XX: link to examples here)
introducing malicious code into the address space of an existing process,
redirecting the flow of execution, etc.


---
### Feature Design
HARDENED\_ATOMIC provides its protections by modifying the data type used
in the Linux kernel to implement reference counters: `atomic\_t`.  `atomic\_t`
is a type that contains an integer type, used for counting.  HARDENED_ATOMIC
modifies `atomic\_t` and its associated API so that the integer type contained
inside of `atomic\_t` cannot be overflowed.     

A key point to remember about HARDENED_ATOMIC is that, once enabled, it protects
all users of `atomic\_t` without any additional code changes.  This widespread
coverage leads to another issue: `atomic\_t` is badly misused.  It is used in
many contexts in which an atomic datatype is not necessarily needed: statistical
counters, for example.    

#### Detect/Mitigate
The core mechanism of HARDENED_ATOMIC can be viewed as a bipartite process:
detection of an overflow and mitigating the effects of the overflow, either by
ignoring it or reversing the operation that caused the overflow.  

Overflow detection is architecture-specific.  Details of the approach
used to detect overflows on each architecture can be found in the PAX_REFCOUNT
[documentation](https://forums.grsecurity.net/viewtopic.php?f=7&t=4173#INTERNALS).

Once an overflow has been detected, HARDENED_ATOMIC mitigates the overflow by
either reverting the operation or simply not writing the result of the operation
to memory.

### Implementation
As mentioned above, HARDENED_ATOMIC modifies the `atomic\_t` API to provide its
protections.  Following is a description of the functions that have been
modified.

First, the type `atomic\_wrap\_t` needs to be defined for those kernel users who want an atomic type that may be allowed to overflow/wrap (e.g. statistical counters).  Otherwise, the built-in protections (and associated costs) for `atomic\_t` would erroneously apply to these non-reference counter users of `atomic\_t`:
- `include/linux/types.h`: define `atomic\_wrap\_t` and `atomic64\_wrap\_t`

Next, we need to define the mechanism for reporting an overflow of a protected atomic type:
- `kernel/panic.c`: `void hardened\_atomic\_refcount\_overflow(struct pt\_regs)`

The following functions are an extension of the `atomic\_t` API, supporting this new "wrappable" type:
- `static inline int atomic\_read\_wrap()`
- `static inline void atomic\_set\_wrap()`
- `static inline void atomic\_inc\_wrap()`
- `static inline void atomic\_dec\_wrap()`
- `static inline void atomic\_add\_wrap()`
- `static inline long atomic\_inc\_return\_wrap()`

#### Departures from Original PaX Implementation
XX: Discuss major departures: REFCOUNT_BIAS change, possible change
to use `cmpxchg` in `atomic_add()` for x86 to avoid a race
condition in the overflow case.  

---
### Performance Impact
The following benchmarks were performed in a x86_64 virtual machine with 8 GB
RAM running Ubuntu 16.04.1:

XX: POST BENCHMARK DATA HERE

---
### Bugs Prevented
The following vulnerabilities would have been prevented by HARDENED_ATOMIC:
- [CVE-2016-3135](https://www.cve.mitre.org/cgi-bin/cvename.cgi?name=2016-3135) - Netfilter xt_alloc_table_info integer overflow
- [CVE-2016-0728](https://www.cve.mitre.org/cgi-bin/cvename.cgi?name=2016-0728) - Keyring refcount overflow
- [CVE-2014-2851](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-2851) - Group_info refcount overflow
- [CVE-2010-2959](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2959) - CAN integer overflow vulnerability,

---
### Future Work
We plan on implementing HARDENED_ATOMIC on all applicable architectures.  

Below is a table containing the implementation status of HARDENED_ATOMIC on each
architecture.  

Architecture  | HARDENED_ATOMIC Supported?    
--------------|-----------------------------
ARM           |  No                           
MIPS          |  No                           
PowerPC       |  No                           
SPARC         |  No                           
x86           |  Yes                          
