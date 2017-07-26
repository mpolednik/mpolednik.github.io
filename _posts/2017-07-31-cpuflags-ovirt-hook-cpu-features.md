---
layout: post
title: cpuflags, oVirt and vCPU features
comments: true
---

## Introduction

A new hook is in the oVirt ecosystem: cpuflags. The cpuflags hook is a small but handy addition that allows fine tuning of vCPU characteristics. More specifically, oVirt users and administrators are now able to select which CPU flags are exposed to guest operating systems.

<!--more-->

## libvirt and CPU models

Before digging deeper into what the hook does, it is important to understand what libvirt CPU models and features are. If you are already educated in this subject matter, feel free to skip the chapter.

When creating a libvirt domain, which is just an XML representation of qemu command line, we usually select a CPU model. Libvirt CPU models are just sets of features -- the CPU flags as found in CPUID instruction. Interesting bit is that CPUID really is x86_64 specific and provides a huge amount of data on various CPU capabilities. Other architectures, such as POWER8, do not provide such granular information. This post is therefore focused on x86_64. So what can we find out about a CPU? To spare us the traversing of all CPUID leaves, libvirt provides EAX and EDX indexes for common features in the `/usr/share/libvirt/cpu_map.xml` file.

```
$ cat /usr/share/libvirt/cpu_map.xml
<cpus>
  <arch name='x86'>
    <!-- vendor definitions -->
    <vendor name='Intel' string='GenuineIntel'/>
    <vendor name='AMD' string='AuthenticAMD'/>

    <!-- standard features, EDX -->
    <feature name='fpu'>
      <cpuid eax_in='0x01' edx='0x00000001'/>
    </feature>
    <feature name='vme'>
      <cpuid eax_in='0x01' edx='0x00000002'/>
    </feature>
    <feature name='de'>
      <cpuid eax_in='0x01' edx='0x00000004'/>
    </feature>
    <feature name='pse'>
      <cpuid eax_in='0x01' edx='0x00000008'/>
    </feature>
    <feature name='tsc'>
      <cpuid eax_in='0x01' edx='0x00000010'/>
    </feature>
    <feature name='msr'>
      <cpuid eax_in='0x01' edx='0x00000020'/>
    </feature>
    ...
```

Requesting a specific feature in libvirt XML, done by following snippet, verifies that the host's (physical) CPUID EAX/EDX leaves contain desired value and instructs QEMU to include it in guest's CPUID.

```xml
<cpu match='exact'>
  ...
  <feature policy='disable' name='mmx'/>
  ...
</cpu>
```

That gives us an overview of the basic vCPU capability building block. Moving further, there is something called CPU model. It is important to understand what that is: CPU model is just a set of features. Specifying `skylake-client` CPU model does not magically make the vCPU a skylake processor. Something way less fancy happens: such statement ensures the vCPU supports skylake-defining features: 

```
3dnowprefetch, abm, adx, aes, apic, arat, avx, avx2, bmi1, bmi2, clflush, cmov,
cx16, cx8, de, erms, f16c, fma, fpu, fsgsbase, fxsr, hle, invpcid, lahf_lm, lm,
mca, mce, mmx, movbe, mpx, msr, mtrr, nx, pae, pat, pcid, pclmuldq, pge, pni,
popcnt, pse, pse36, rdrand, rdseed, rdtscp, rtm, sep, smap, smep, sse, sse2,
sse4.1, sse4.2, ssse3, syscall, tsc, tsc-deadline, vme, x2apic, xgetbv1, xsave,
xsavec, xsaveopt
```

Other CPU models act in a similar fashion, usually exposing a subset of higher model's capabilities.

### host-model, host-passthrough

The special cpu models -- host-model and host-passthrough allow for a very specific vCPU setting. First, host-model tries to match all the features supported by physical CPU. The second option goes even further and attempts to copy every detail of host's CPU (not only features) to the vCPU. Unfortunately, that's not really true. Certain flags, such as invtsc, come with special treatment and are ignored in these special cases. That is where oVirt `cpuflags` hook becomes important.

## cpuflags hook

The hook RPM is called vdsm-hook-cpuflags. To configure which features should be required or avoided, custom property called `cpuflags` is used. The property uses a specific syntax:

1. +feature includes a feature
2. -feature excludes a feature
3. uppercase GROUP includes a group of features
4. features and groups are separated by comma (`,`)

For better understanding, consider following example:

```
cpuflags='+svm,+invpcid,SAP,-ssse3'
```

In that case, we

1. include svm, invpcid features
2. include SAP feature group, which is a special group
3. exclude ssse3 flag

The groups are just sets of features named for convenience. The hook automatically removes duplicate feature inclusion or exclusion, but it prevents creation of VM if conflicting choice or invalid group is specified. Conflicting choice roughly looks like this: `+flag,-flag`. In that case, the hook is unable to determine whether the feature should be present or not, and it avoids guessing. Similarly, undefined groups or lowercase feature names without +/- operator are not permitted and will lead to the VM not starting.

As a remainder, the hook must be installed on virtualization hosts that want to take advantage of it, and ovirt-engine must be configured to support the custom property:

```
$ engine-config -s UserDefinedVMProperties='hugepages=^.*$;mdev_uuid=^.*$;cpuflags=^.*$'
Please select a version:
1. 3.6
2. 4.0
3. 4.1
4. 4.2
4
```

That is the new addition that enables fine grained tuning of vCPU capabilities to choose a trade-off between convenience and performance.
