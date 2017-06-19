---
layout: post
title: Hugepages and oVirt
comments: true
---

## Introduction
Previously, we have talked about hugepages and how are they helpful and used in virtualization on the level of QEMU or libvirt. Let's continue our tour de stack by oVirt's shiny and new hugepages implementation.

<!--more-->

## oVirt

oVirt had upstream hook that allowed users to use custom property to request hugepage memory backing. The hook unfortunately has had multiple pitfalls: it only supported 2 MiB pages -- making it x86_64 only (and still without 1 GiB hugepages). The hook is now gone and the functionality is merged within VDSM and engine code base - that makes hugepages a native, almost-first-class citizen in oVirt.

## Setup

To enable hugepages backing, we must first allocate the hugepages. The code can allocate the pages right before VM starts, although It's not exactly recommend that: as time goes, the chance to allocate the requested number of hugepages goes down due to memory fragmentation. The most sane way is therefore using kernel command line, and the most supported option is doing that through ovirt-host-deploy interface.

{% include image.html name="ohd.png" width="100%" %}

The process is as follows: `Hosts` -> `Edit` -> `Kernel` -> (set everything up, OK) -> Host `Maintenance` -> Reboot the physical machine -> Host `Activate`.

## Configuration

When hugepages are available on the host, the only requirement is setting predefined property called `hugepages`, where the format is `hugepages=size`. Setting `hugepages=2048` therefore sets 2 MiB hugepages, `hugepages=1048576` yields 1 GiB hugepages.

{% include image.html name="property.png" width="100%" %}

Additionally, there are several new settings in /etc/vdsm/vdsm.conf:

* use_dynamic_hugepages -- This is the option that enables dynamic allocation of hugepages. If enabled, VDSM can allocate extra hugepages to accommodate the VM without preallocating the pages. The algorithm prefers preallocated pages if those exist, and only allocates additional pages if necessary.
* use_preallocated_hugepages -- If you expect VDSM to prefer preallocated pages, set this to `true`. Actually `true` by default.
* reserved_hugepage_count -- The interesting part. oVirt expects some preallocated pages to be used for different processes that just VMs. Example of such process is DPDK that assumes hugepages backing. If there is such process running on the host, set the `reserved_hugepages_count` and `reserved_hugepage_size` accordingly to accommodate it.
* reserved_hugepages_size -- See above.
