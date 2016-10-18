---
layout: post
title: Linux KSPP - HARDENED_ATOMIC
date: 2016-10-07
---

# Linux KSPP: HARDENED\_ATOMIC
The Linux Kernel Self Protection Project
([KSPP](http://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project)) was
created with a mandate to eliminate classes of kernel bugs.  To date, this work
has largely involved porting features found in
[PaX](https://pax.grsecurity.net/)/[grsecurity](https://grsecurity.net/) to the
mainline Linux kernel.  A few such features, like
[hardened usercopy](https://www.lwn.net/Articles/695991), have already been accepted 
into the mainline kernel.  

Earlier in 2016 I started [working](https://lwn.net/Articles/668724/) on porting
`PAX_REFCOUNT`, a
[feature](https://forums.grsecurity.net/viewtopic.php?f=7&t=4173) of PaX that
protects kernel reference counters against overflow.  A team from
[Intel](http://www.01.org) graciously decided to collaborate with me and
finally we've submitted an [RFC](https://lwn.net/Articles/702640/).  

This page will serve as documentation for `HARDENED_ATOMIC` and will be updated
as the feature goes through the steps for merging into mainline Linux.  

### Why do we need `HARDENED_ATOMIC`?

#### Use-after-free Bugs
Rather than targeting individual bugs, KSPP aims to eliminate entire categories
of vulnerabilities from the Linux kernel.  The class of vulnerabilities
addressed by `HARDENED_ATOMIC` is known as use-after-free vulnerabilities.

Use-after-free vulnerabilities are aptly named: they are classes of bugs in
which an attacker is able to gain control of a piece of memory after it has
already been freed and use this memory for nefarious purposes: introducing 
malicious code into the address space of an existing process, redirecting the
flow of execution, etc.

### Feature Design
`HARDENED_ATOMIC` provides its protections by modifying the data type used
in the Linux kernel to implement reference counters: `atomic_t`.  `atomic_t`
is a type that contains an integer type, used for counting.  `HARDENED_ATOMIC`
modifies `atomic_t` and its associated API so that the integer type contained
inside of `atomic_t` cannot be overflowed.     

A key point to remember about `HARDENED_ATOMIC` is that, once enabled, it protects
all users of `atomic_t` without any additional code changes. The protection
provided by `HARDENED_ATOMIC` is not "opt-in": since `atomic_t` is so widely
misused, it must be protected as-is.  `HARDENED_ATOMIC` protects all users
of `atomic_t` and `atomic_long_t` against overflow.  New users wishing to use
atomic types, but not needing protection against overflows, should use the
new types introduced by this series: `atomic_wrap_t` and `atomic_long_wrap_t`.

#### Detect/Mitigate
The core mechanism of `HARDENED_ATOMIC` can be viewed as a bipartite process:
detection of an overflow and mitigating the effects of the overflow, either by
ignoring it or reversing the operation that caused the overflow.  

Overflow detection is architecture-specific.  Details of the approach
used to detect overflows on each architecture can be found in the `PAX_REFCOUNT`
[documentation](https://forums.grsecurity.net/viewtopic.php?f=7&t=4173#INTERNALS).

Once an overflow has been detected, `HARDENED_ATOMIC` mitigates the overflow by
either reverting the operation or simply not writing the result of the operation
to memory.

### Implementation
As mentioned above, `HARDENED_ATOMIC` modifies the `atomic_t` API to provide its
protections.  Following is a description of the functions that have been
modified.

First, the type `atomic_wrap_t` needs to be defined for those kernel users who want an atomic type that may be allowed to overflow/wrap (e.g. statistical counters).  Otherwise, the built-in protections (and associated costs) for `atomic_t` would erroneously apply to these non-reference counter users of `atomic_t`:

* `include/linux/types.h`: define `atomic_wrap_t` and `atomic64_wrap_t`

Next, we need to define the mechanism for reporting an overflow of a protected atomic type:

* `kernel/panic.c`: `void hardened_atomic_overflow(struct pt_regs)`

The following functions are an extension of the `atomic_t` API, supporting this new "wrappable" type:

- `static inline int atomic_read_wrap()`
- `static inline void atomic_set_wrap()`
- `static inline void atomic_inc_wrap()`
- `static inline void atomic_dec_wrap()`
- `static inline void atomic_add_wrap()`
- `static inline long atomic_inc_return_wrap()`

#### Departures from Original PaX Implementation
While `HARDENED_ATOMIC` is based largely upon the work done by PaX in their
original `PAX_REFCOUNT` patchset, `HARDENED_ATOMIC` does in fact have a few minor
differences.  We will be posting them here as final decisions are made regarding
how certain core protections are implemented.

##### x86 Race Condition
In the original implementation of `PAX_REFCOUNT`, a
[known race condition](https://forums.grsecurity.net/viewtopic.php?f=7&t=4173#APPENDA)
exists when performing atomic add operations.  The crux of the problem lies in
the fact that, on x86, the "detect" portion of `PAX_REFCOUNT`'s detect/mitigate
mechanism actually needs to perform a prospective operation in order to determinine if
that operation produces an overflow.  In other words, the original `PAX_REFCOUNT`
implementation provided no way to determine a priori whether a prospective atomic
operation will result in an overflow.

Therefore, there exists a set of conditions in which, given the correct timing of
threads, an overflowed counter could be visible to a processor.  If multiple
threads execute in such a way so that one thread overflows the counter with an
addition operation, while a second thread executes another addition operation on
the same counter before the first thread is able to revert the previously executed
addition operation (by executing a subtraction operation of the same (or greater)
magnitude), the counter will have been incremented to a value greater than `INT_MAX`.
At this point, the protection provided by `PAX_REFCOUNT` has been bypassed, as further
increments to the counter will not be detected by the processor's overflow detection
mechanism.

The likelihood of an attacker being able to exploit this race was sufficiently
insignificant such that `PAX_REFCOUNT`'s authors thought fixing the race would be
counterproductive.

### Performance Impact
Preliminary benchmarks indicate `HARDENED_ATOMIC` incurs negligible performance
impact.  However, we will be posting definitive benchmarks soon.

### Bugs Prevented
The following vulnerabilities would have been prevented by `HARDENED_ATOMIC`:  

* [CVE-2016-3135](https://www.cve.mitre.org/cgi-bin/cvename.cgi?name=2016-3135) - Netfilter `xt_alloc_table_info` integer overflow
* [CVE-2016-0728](https://www.cve.mitre.org/cgi-bin/cvename.cgi?name=2016-0728) - Keyring refcount overflow
* [CVE-2014-2851](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-2851) - `group_info` refcount overflow
* [CVE-2010-2959](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-2959) - CAN integer overflow vulnerability,

### Future Work
We plan on implementing `HARDENED_ATOMIC` on all applicable architectures.  

Below is a table containing the implementation status of `HARDENED_ATOMIC` on each
architecture. 
 
|--------------------+-----------------|
|  Arch              | Supported       |
|:-------------------+----------------:|
| arm                | In Progress     |
| arm64              | In Progress     |
| MIPS               | No              |
| PowerPC            | No              |
| SPARC              | No              |
| x86                | In Progress     |
|--------------------+-----------------|

