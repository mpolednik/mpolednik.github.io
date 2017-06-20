---
layout: post
title: Hugepages and Virtualization
comments: true
---

## Introduction
Hugepages are memory pages that happen to be larger than the platform dependent standard size (x86_64 uses 4 KiB, ppc64le 16 KiB). Why do they exist and how are they interesting for (not only) virtualization?

<!--more-->

## Pages, Hugepages and Memory Management

To understand the purpose of hugepages, we have to delve down into the inner workings of computer memory management. Modern CPUs use real (physical) addressing while the software takes advantage of virtual memory. There are mostly advantages to this approach, as virtual memory gives you not only portability, scalabity (swap), security (ASLR) and backwards compatibility. The compiler does not have to rely on specific physical addresses being used for functions, or whole memory regions reserved for devices (ISA, PCI-e). 

There is a downside to all this: there must exist a translation of virtual memory regions to physical addresses. Since the translation mapping can grow beyond the size of CPU cache, it is mostly stored in RAM. To make the translation reasonably fast, we have a somewhat transparent cache called Translation Lookaside Buffer (TLB). The cache is preferred when the processor needs to translate a virtual address to a physical address, cache misses are handled by the hardware (at least for x86_64). Still, cache misses waste unnecessary CPU cycles by fetching the page from real address space, validating the memory access and updating the TLB (as it stores most recent translations).

Fortunately, the number of TLB misses can be minimized -- one possibility is increasing page size. If a single page is 2 MiB large, it replaces 512 page table entries. Although significant, x86_64 goes step further and adds support for 1 GiB pages -- condensing 262 144 page table entries into a single entry. Finally, ppc64le architecture is wild enough to use default page size of 16 KiB and 16 GiB for hugepages. Yes, that is whopping 1 048 576 page table entries saved.

Nothing is ever without a catch: the TLB size within CPUs cache is limited to certain number of pages of each size. For my Xeon E5 2650v3 L1-DTLB it is 64 entries for 4 KiB pages, 32 entries for 2 MiB pages and 4 entries for 1 GiB pages. Higher order (L2) cache is unified, holding up to 1024 4 KiB and 2 MiB pages <sup>[1]</sup>. Using multiple 1 GiB pages with (pseudo)random memory access distribution may result in higher number of TLB misses than if 4 KiB pages were used, although the circumstances are quite specific to hit that <sup>[2]</sup>.

## Transparent Hugepages and libhugetlbfs

There are two methods of operation for hugepages: transparent and process specific. The transparent method of operation uses a logic built within kernel (via khugepagesd) that allocates 2 MiB pages whenever 2 MiB aligned region is requested, given that the hugepage can be allocated. In case the allocation does not succeed, kernel can still fall back to default page allocation (2 KiB in case of x86_64). The main difference from process specific allocation is that such page can be swapped by breaking it down to original default sized pages, whereas explicitly allocated hugepages are unswappable. The transparent approach doesn't look too promising [in the field](https://www.perforce.com/blog/tales-field-taming-transparent-huge-pages-linux) though.

Process specific hugepages are allocated either at boot time via kernel command line options (hugepages=NR_HUGEPAGES, hugepagesz=PAGESIZE) or at the runtime by accessing the memory management sysfs interface at /sys/kernel/mm/hugepages/hugepages-PAGESIZEkB/nr_hugepages. Such pages have their specific mount point per page size, such as /dev/hugepages and /dev/hugepages1G and cannot be used transparently. There is a library called [libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs) that facilitates per-process access to user allocated hugepages. Notable function from the library are `get_hugepage_region()`, `get_huge_pages()` and their corresponding `free` variants. There is a high level overview of the interface in this [LWN article](https://lwn.net/Articles/375096/).

## Hugepages and Virtualization

Virtual machine (VM, NOT virtual memory in this post) is an unusual breed of a process. The host OS contains the whole memory of the guest OS in its RAM, along with some overhead caused by devices (virtio, graphics, ...). That means we have a process that allocates huge chunk of host's memory and runs whole operating system within it, with memory accesses all over it. That, along with various database applications, sounds like perfect opportunity to use hugepages.

QEMU itself is able to use preallocated hugepages to back guest OS' memory by specifying `-m 1024 -mem-prealloc -mem-path /path/to/hugepages` arguments on the qemu-kvm command line. If you are not into fiddling with command line, libvirt is able to translate the following XML snippet into the same command line:

```xml
  <memoryBacking>
    <hugepages/>
  </memoryBacking>
```

The above snippet tries to back the guest memory by hugepages of default size. We can also request backing by specific size:

```xml
  <memoryBacking>
    <hugepages>
        <page size="1048576">
    </hugepages>
  </memoryBacking>
```

The requirement is that you have already allocated hugepages by supplying kernel command line parameters (mentioned above) or using the sysfs api (mentioned above).

I've seen performance improvement in the lower tens of percent (unreferenced unfortunately) by just backing the guest's memory with 1 GiB hugepages instead of regular pages. If you are into experimenting, I'd more than welcome any referencable feedback!

[1]: http://repnop.org/pd/slides/PD_Haswell_Architecture.pdf
[2]: https://www.pvk.ca/Blog/2014/02/18/how-bad-can-1gb-pages-be/
