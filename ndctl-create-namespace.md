---
title: ndctl
layout: pmdk
---

NAME
====

ndctl-create-namespace - provision or reconfigure a namespace

SYNOPSIS
========

>     ndctl create-namespace [<options>]

DESCRIPTION
===========

A REGION, after resolving DPA aliasing and LABEL specified boundaries, surfaces one or more "namespace" devices. The arrival of a "namespace" device currently triggers either the nd\_blk or nd\_pmem driver to load and register a disk/block device.

EXAMPLES
========

Create a maximally sized pmem namespace in *fsdax* mode (the default)

>     ndctl create-namespace

Convert namespace0.0 to *sector* mode

>     ndctl create-namespace -f -e namespace0.0 --mode=sector

OPTIONS
=======

`-t; --type=`  
Create a *pmem* or *blk* namespace (subject to available capacity). A pmem namespace supports the dax (direct access) capability to mmap2 persistent memory directly into a process address space. A blk namespace access persistent memory through a block-window-aperture. Compared to pmem it supports a traditional storage error model (EIO on error rather than a cpu exception on a bad memory access), but it does not support dax.

`-m; --mode=`  
-   "raw": expose the namespace capacity directly with limitations. Neither a raw pmem namepace nor raw blk namespace support sector atomicity by default (see "sector" mode below). A raw pmem namespace may have limited to no dax support depending the kernel. In other words operations like direct-I/O targeting a dax buffer may fail for a pmem namespace in raw mode or indirect through a page-cache buffer. See "fsdax" and "devdax" mode for dax operation.

-   "sector": persistent memory, given that it is byte addressable, does not support sector atomicity. The problematic aspect of sector tearing is that most applications do not know they have a atomic sector update dependency. At least a disk rarely ever tears sectors and if it does it almost certainly returns a checksum error on access. Persistent memory devices will always tear and always silently. Until an application is audited to be robust in the presence of sector-tearing "safe" mode is recommended. This imposes some performance overhead and disables the dax capability. (also known as "safe" or "btt" mode)

-   "fsdax": A pmem namespace in this mode supports dax operation with a block-device based filesystem (in previous ndctl releases this mode was named "memory" mode). This mode comes at the cost of allocating per-page metadata. The capacity can be allocated from "System RAM", or from a reserved portion of "Persistent Memory" (see the --map= option). Note that a filesystem is required for dax operation, the resulting raw block device (/dev/pmemX) will use the page cache. See "devdax" mode for raw device access that supports dax.

-   "devdax": The device-dax character device interface is a statically allocated / raw access analogue of filesystem-dax (in previous ndctl releases this mode was named "dax" mode). It allows memory ranges to be mapped without need of an intervening filesystem. The device-dax is interface strict, precise and predictable. Specifically the interface:

    -   Guarantees fault granularity with respect to a given page size (4K, 2M, or 1G on x86) set at configuration time.

    -   Enforces deterministic behavior by being strict about what fault scenarios are supported. I.e. if a device is configured with a 2M alignment an attempt to fault a 4K aligned offset will result in SIGBUS.

`-s; --size=`  
For NVDIMM devices that support namespace labels, set the namespace size in bytes. Otherwise it defaults to the maximum size specified by platform firmware. This option supports the suffixes "k" or "K" for KiB, "m" or "M" for MiB, "g" or "G" for GiB and "t" or "T" for TiB.

    For pmem namepsaces the size must be a multiple of the
    interleave-width and the namespace alignment (see
    below).

`-a; --align`  
Applications that want to establish dax memory mappings with page table entries greater than system base page size (4K on x86) need a persistent memory namespace that is sufficiently aligned. For "fsdax" and "devdax" mode this defaults to 2M. Note that "devdax" mode enforces all mappings to be aligned to this value, i.e. it fails unaligned mapping attempts. The "fsdax" alignment setting determines the starting alignment of filesystem extents and may limit the possible granularities, if a large mapping is not possible it will silently fall back to a smaller page size.

`-e; --reconfig=`  
Reconfigure an existing namespace (change the mode, sector size, etc…). All namespace parameters, save uuid, default to the current attributes of the specified namespace. The namespace is then re-created with the specified modifications. The uuid is refreshed to a new value by default whenever the data layout of a namespace is changed, see --uuid= to set a specific uuid.

`-u; --uuid=`  
This option is not recommended as a new uuid should be generated every time a namespace is (re-)created. For recovery scenarios however the uuid may be specified.

`-n; --name=`  
For NVDIMM devices that support namespace labels, specify a human friendly name for a namespace. This name is available as a device attribute for use in udev rules.

`-l; --sector-size`  
Specify the logical sector size (LBA size) of the Linux block device associated with an namespace.

`-M; --map=`  
A pmem namespace in "fsdax" or "devdax" mode requires allocation of per-page metadata. The allocation can be drawn from either:

-   "mem": typical system memory

-   "dev": persistent memory reserved from the namespace

        Given relative capacities of "Persistent Memory" to "System
        RAM" the allocation defaults to reserving space out of the
        namespace directly ("--map=dev"). The overhead is 64-bytes per
        4K (16GB per 1TB) on x86.

`-f; --force`  
Unless this option is specified the *reconfigure namespace* operation will fail if the namespace is presently active. Specifying --force causes the namespace to be disabled before the operation is attempted. However, if the namespace is mounted then the *disable namespace* and *reconfigure namespace* operations will be aborted. The namespace must be unmounted before being reconfigured.

`-L; --autolabel; --no-autolabel`  
Legacy NVDIMM devices do not support namespace labels. In that case the kernel creates region-sized namespaces that can not be deleted. Their mode can be changed, but they can not be resized smaller than their parent region. This is termed a "label-less namespace". In contrast, NVDIMMs and hypervisors that support the ACPI 6.2 label area definition (ACPI 6.2 Section 6.5.10 NVDIMM Label Methods) support "labelled namespace" operation.

-   There are two cases where the kernel will default to label-less operation:

    -   NVDIMM does not support labels

    -   The NVDIMM supports labels, but the Label Index Block (see UEFI 2.7) is not present and there is no capacity aliasing between *blk* and *pmem* regions.

-   In the latter case the configuration can be upgraded to labelled operation by writing an index block on all DIMMs in a region and re-enabling that region. The *autolabel* capability of *ndctl create-namespace --reconfig* tries to do this by default if it can determine that all DIMM capacity is referenced by the namespace being reconfigured. It will otherwise fail to autolabel and remain in label-less mode if it finds a DIMM contributes capacity to more than one region. This check prevents inadvertent data loss of that other region is in active use. The --autolabel option is implied by default, the --no-autolabel option can be used to disable this behavior. When automatic labeling fails and labelled operation is still desired the safety policy can be bypassed by the following commands, note that all data on all regions is forfeited by running these commands:

        ndctl disable-region all
        ndctl init-labels all
        ndctl enable-region all

`-v; --verbose`  
Emit debug messages for the namespace creation process

`-r; --region=`  
    A 'regionX' device name, or a region id number. The keyword 'all' can
    be specified to carry out the operation on every region in the system,
    optionally filtered by bus id (see --bus= option).

`-b; --bus=`  
Enforce that the operation only be carried on devices that are attached to the given bus. Where *bus* can be a provider name or a bus id number.

COPYRIGHT
=========

Copyright (c) 2016 - 2018, Intel Corporation. License GPLv2: GNU GPL version 2 <http://gnu.org/licenses/gpl.html>. This is free software: you are free to change and redistribute it. There is NO WARRANTY, to the extent permitted by law.

SEE ALSO
========

[ndctl-zero-labels](ndctl-zero-labels.md), [ndctl-init-labels](ndctl-init-labels.md), [ndctl-disable-namespace](ndctl-disable-namespace.md), [ndctl-enable-namespace](ndctl-enable-namespace.md), [UEFI NVDIMM Label Protocol](http://www.uefi.org/sites/default/files/resources/UEFI_Spec_2_7.pdf)