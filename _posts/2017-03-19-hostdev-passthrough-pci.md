---
layout: post
title: hostdev passthrough - PCI
comments: true
---

## Introduction
What does that even mean?!

There are multiple names for the functionality I'll be talking about today -- PCI device assignment, device assignment, PCI passthrough, host device passthrough, hostdev passthrough... There really is no difference between them for the sake of this post. Let's drop the fancy wording for a minute - we are talking about taking a device which happens to be on computer's PCI bus and "passing it" to a virtual machine with ideally no performance loss.

PCI device within host operating system:
{% include image.html name="pci-intro.png" width="100%" %}

PCI device passthrough:
{% include image.html name="pci-pt-intro.png" width="100%" %}

What can we achieve with that?

## Use Cases

Well, one of the easiest examples is gaming. If you are one of many people who happen to dislike proprietary operating systems, it goes without saying that gaming (on a casual level, let's not even get started with professional gaming) is one of the reasons why you may have such OS on a desktop at home. There is a multitude of variables that make gaming fun -- input latency (how long does it take for your key presses and mouse movement to be registered), hardware compatibility, performance (network, graphics) and, of course, the game itself. Trying to play a game on Linux may come with a quirks that reduce the fun. Bad graphics performance isn't fun.

Instead of trying to fix all the issues you have on Linux, or going for a proprietary software, it's possible to compromise. Imagine running a Linux on your bare metal desktop and your gaming "PC" as a virtual machine. See where I'm going?

KVM and QEMU (and libvirt, and oVirt if you happen to game in your data center) has got you covered! It's possible to accomplish the above with few QEMU features, namely [PCI passthrough](https://docs.fedoraproject.org/en-US/Fedora/13/html/Virtualization_Guide/chap-Virtualization-PCI_passthrough.html), [USB passthrough](http://www.linux-kvm.org/page/USB_Host_Device_Assigned_to_Guest).

Let's slow down for a minute though. We are talking about PCI passthrough, not just GPU. Most of the expansion cards in today's computers (desktops, laptops, enterprise pizza boxes) happen to be connected by PCI express. Graphics cards (rendering, deep learning), network interface cards (10 GiB, 40 GiB, specialized ones), HBAs (storage adapters), USB expansion slots, serial console ports, even FPGAs and many other specialized cards! Got exotic hardware that doesn't have drivers you need? Run it in a VM with different OS!

Unfortunately, getting everything right for PCI passthrough is hard. In theory, this whole features works by taking a PCI card, "decomposing" it's OS-level components (using vfio-pci instead of any other driver - Alex Williamson had a GREAT [talk](http://events.linuxfoundation.org/sites/events/files/slides/An%20Introduction%20to%20PCI%20Device%20Assignment%20with%20VFIO%20-%20Williamson%20-%202016-08-30_0.pdf) about that in Toronto) and giving QEMU access these components to re-create the device inside the guest.

## Requirements

In practice, there is a bit more going on. Let's start with summary what kind of hardware/firmware support is needed:

- support for Intel VT-d or AMD-Vi within the CPU, enabled in BIOS;
- IOMMU enabled within the host operating system (intel_iommu=on, amd_iommu=on on kernel cmdline),
- kernel version that knows how to work with IOMMU,
- a bit of luck.

If you have all that, you're halfway there! Why only halfway? You can see the word [IOMMU](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit) mentioned. Practically, IOMMU groups your devices by the level of isolation there is between them on the PCI level. Great example is yet again GPU -- consumer grade GPUs usually come as two devices, one being the actual GPU whereas the second one is a sound card for HDMI output. These two devices (as far as I have seen) are not isolated on the hardware level. That means they are within the same IOMMU group, and that means they can only be passed to a VM as a group. More specifically, the whole group has to be bound to vfio-pci driver, and any of the devices within that group can be passed to a VM. Small caveat is that if there is a PCI bridge within that group, you shouldn't unbind or pass it through. [Clever software](https://www.ovirt.org/) should handle that for you.

Additionally, if you happen to have a hardware that you want to dedicate to a VM, it is safe not to bind it to it's original driver when system boots up as you could be risking host crash or similar - due to poor driver code.

To do everything mentioned above on QEMU/libvirt, there are tons of great resources from incredibly clever people: Alex Williamson's [VFIO blog](vfio.blogspot.com), [Gentoo Wiki](https://wiki.installgentoo.com/index.php/PCI_passthrough), my friend's blog about [gaming in QEMU](http://www.zveleba.cz/?p=52), [my feature page](http://www.ovirt.org/develop/release-management/features/engine/hostdev-passthrough/), and many more that can be discovered using your favorite search engine.

## PCI passthrough in oVirt

If you want to use hostdev passthrough in your data center, it's a good idea to use software created for data centers such as [oVirt](http://ovirt.org/). So how to do that?

First, the steps above apply, so unfortunately the first step is "get the right hardware". When you have that, we are good to move forward. To configure your host for IOMMU, look into kernel tab when adding a host. The important parts are labeled by a red box .

{% include image.html name="kernel-tab.png" width="100%" %}

After pressing OK and waiting for a host to be installed, make sure that the host is in maintenance mode and reboot it. That is to make sure we are booted into IOMMU capable kernel. When the host is booted up, activate it in oVirt and check it's Device Passthrough status.

{% include image.html name="host-passthrough.png" width="100%" %}

You can then look at the devices available on the host and their IOMMU group.

{% include image.html name="host-devices.png" width="100%" %}

Got this far? AWESOME! You're almost ready to go. Now, in oVirt, when creating a VM with a passthrough device it has to be pinned to a specific host. Luckily that is handled by the software when choosing a device.

{% include image.html name="vm-devices-pre.png" width="100%" %}
{% include image.html name="vm-devices-mid.png" width="100%" %}
{% include image.html name="vm-devices-post.png" width="100%" %}

If you have chosen a device which is in IOMMU group with other devices that you have not chosen, oVirt will automatically add the other devices as kind of a place holders. That is indicated by slightly faded text.
{% include image.html name="vm-devices-iommu.png" width="100%" %}

What does it look like in the guest OS? For the sake of this demo, I've passed through a network card to a Windows VM.
{% include image.html name="win-nic.png" width="100%" %}

There we go. Using the above steps, you can pass through pretty much any PCI device to a VM. Do you feel like you need USB or SCSI devices? SR-IOV, NPIV, GPU? Stay tuned!
