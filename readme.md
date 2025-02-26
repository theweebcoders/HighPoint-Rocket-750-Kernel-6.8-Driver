# HighPoint Rocket 750 Kernel 6.8 Driver Patching

## Overview
This repository contains the **open-source driver for the HighPoint Rocket 750 HBA**, a 40-port SATA 6Gb/s PCI-Express 2.0 x8 Host Bus Adapter (HBA). Although an official driver exists, it only supports Linux kernel versions up to 5.x. The goal of this project is to **patch the existing Rocket 750 driver** so that it compiles and functions correctly under **Linux Kernel 6.8**.

### Purpose of This Project
- **Modernization:** Update the driver codebase to replace outdated kernel APIs, data structures, and build system components.
- **Compatibility:** Ensure full support for Linux Kernel 6.8 by eliminating deprecated calls and obsolete SCSI, PCI, DMA, and interrupt handling methods.
- **Documentation:** Maintain detailed notes on every change for future maintainability.

---

## Official Specifications

### 📌 HighPoint Rocket 750 Specifications
- **HBA Type:** 40-Port SATA 6Gb/s PCI-Express 2.0 x8 HBA
- **Connector Type:** SFF-8087 (10 Mini-SAS Ports)
- **Maximum Drives Supported:** 40 SATA Devices
- **Max Storage Capacity:** 320TB
- **Power Consumption:** 4W max (12V), 1W max (3.3V)
- **Management Features:**  
  - SGPIO, Active/Fail LED, I2C interface  
  - Hot-Plug & Hot-Swap Support  
  - Staggered Drive Spin-Up

### 📌 Official Operating System Support (From 2015 Manual)
- **Windows:** 8 / 2012 / 7 / 2008R2  
- **Linux Distributions:**  
  - ✅ **Ubuntu (Older releases)**
  - ✅ **SUSE Linux Enterprise Server (SLES)**
  - ✅ **Red Hat Enterprise Linux (RHEL)**
  - ✅ **OpenSUSE**
  - ✅ **Fedora**
  - ✅ **FreeBSD**

## Codebase Structure

```
├── inc
│   ├── linux_64mpa
│   │   ├── Makefile.def   # Kernel compatibility restrictions (updated to support 6.x)
│   │   └── osm.h          # Kernel interface definitions
│   ├── array.h
│   ├── him.h
│   ├── himfuncs.h
│   ├── hptintf.h
│   ├── ldm.h
│   └── list.h
├── osm
│   └── linux
│       ├── div64.c
│       ├── hptinfo.c
│       ├── install.sh     # Updated installation script for modern package management and systemd support
│       ├── os_linux.c     # Updated disk revalidation and block device handling
│       ├── osm_linux.c
│       ├── osm_linux.h
│       └── patch.sh
├── product
│   └── r750
│       └── linux
│           ├── config.c  # Driver initialization and configuration logic
│           └── Makefile  # Updated for kernel 6.8 build paths and source file adjustments
└── install.sh           # Top-level installer script

```

## Current Issues & Compatibility Challenges

The HighPoint Rocket 750 driver was originally designed for much older Linux kernels and has only seen minimal updates over the years. Unfortunately, modern kernels—including Linux 6.8—have introduced major changes that break compatibility.

Some of the biggest challenges include:

-	The driver relies on outdated kernel functions that no longer exist or work differently in modern versions.
-	Changes in how Linux handles PCI devices, storage, and memory management cause build failures and runtime issues.
-	The block device and RAID handling in the driver are based on legacy systems that may not even be necessary anymore.
-	Core SCSI and DMA operations have been updated, making the driver’s current implementation incompatible.
-	There’s little to no documentation on how this driver was originally meant to function, making troubleshooting difficult.

At this point, the driver does not compile or load properly on Linux 6.8, and even if it did, there’s no guarantee that it would function correctly. The goal of this project is to reverse-engineer what’s broken, update what’s needed, and get it working again.

For a more detailed breakdown of specific issues and planned fixes, see the Modernization Roadmap below.

## **Modernization Roadmap** (What Needs to Be Done)

The following steps outline the process to modernize the Rocket 750 driver for **Linux Kernel 6.8**.  

Each step is a **standalone task** that can be worked on independently. The goal is to **systematically replace outdated code** while keeping track of completed work.  

### **Compatibility Layer & Build Fixes**  
**✅ Status: Partially Completed (Makefile.def updated, build process partially working, compat.h planned for macro stubs)**  

**Create a Compatibility Header (`compat.h`)**
   - **Problem:** The driver references missing macros/functions (`virt_to_bus`, `SCSI_DISKx_MAJOR`, `CHECK_CONDITION`, etc.).
   - **Solution:** Create `compat.h` to define stubs for missing macros and deprecated functions.
   - **Tasks:**  
     - **Define a replacement for `virt_to_bus()`** (`dma_map_single()` for buffer mappings).  
     - **Stub out `SCSI_DISKx_MAJOR` macros** (or remove them entirely).  
     - **Define `CHECK_CONDITION`, `GOOD`, `DRIVER_SENSE`, etc.** if missing.  
     - **Stub out `scsi_done()` calls.**  
     - **Include `compat.h` in `osm_linux.h` and other relevant files.**  
     - **Recompile and verify that missing macro errors are resolved.**  

**Update the Build System (Done)**
   - **✅ Status: Completed (Makefile.def updated, build errors partially fixed).**
   - **Remaining Work:**  
     - Ensure Makefile paths are correct (`KERNELDIR` and `PWD` consistency).  
     - Run `make` after all compatibility fixes to check for new errors.  

---

### **Fixing Obsolete DMA & PCI Calls**
**✅ Status: In Progress**  

**Replace `virt_to_bus()` With DMA API**  
   - **Problem:** `virt_to_bus()` was removed from the kernel long ago.  
   - **Solution:** Replace it with `dma_map_single()` and `dma_alloc_coherent()`.  
   - **Tasks:**  
     - Locate all instances of `virt_to_bus()` (e.g., in `os_linux.c`, `osm_linux.c`).  
     - Replace them with:  
       ```c
       dma_addr_t dma_addr = dma_map_single(&pcidev->dev, ptr, size, DMA_BIDIRECTIONAL);
       if (dma_mapping_error(&pcidev->dev, dma_addr)) {
           // Handle error
       }
       ```
     - Ensure `dma_unmap_single()` is properly used to clean up DMA mappings.  
     - Verify all changes compile correctly.  

**Update PCI Handling (`pci_set_dma_mask`)**  
   - **Problem:** `pci_set_dma_mask()` is deprecated; we should use `dma_set_mask_and_coherent()`.  
   - **Solution:** Ensure that `pci_set_dma_mask()` is used correctly with error handling.  
   - **Tasks:**  
     - Locate all calls to `pci_set_dma_mask()`.  
     - Replace with:  
       ```c
       if (dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64))) {
           dev_err(&pdev->dev, "No suitable DMA support found.\n");
           return -EIO;
       }
       ```
     - Verify that the driver correctly initializes PCI DMA.  

**Fix `blkdev_get()`, `blkdev_put()`, and `bdget()`**  
   - **Problem:** Some block device operations have changed in modern kernels.  
   - **Solution:** Ensure correct function replacements:  
     ```c
     struct block_device *b = blkdev_get_by_dev(bdev->bd_dev, FMODE_READ, NULL);
     if (IS_ERR(b)) return PTR_ERR(b);
     ```
     - Replace `blkdev_put()` calls with:  
       ```c
       blkdev_put(bdev, FMODE_READ);
       ```
     - Remove `revalidate_disk()` calls (automatic in modern kernels).  

---

### **SCSI & Block Device Updates**
**✅ Status: Not Started**  

**Fix Deprecated SCSI Macros & Status Codes**  
   - **Problem:** Macros like `CHECK_CONDITION`, `DRIVER_SENSE`, `GOOD`, `SUGGEST_ABORT` are missing.  
   - **Solution:** Replace them with standard SCSI status handling.  
   - **Tasks:**  
     - Define missing macros in `compat.h`.  
     - Replace manual sense data handling with:  
       ```c
       scsi_build_sense_buffer(0, SCpnt->sense_buffer, /* sense key */ 0x3, /* asc */ 0x11, /* ascq */ 0x04);
       SCpnt->result = (DID_OK << 16) | COMMAND_COMPLETE;
       ```
     - Verify that error reporting works correctly.  

**Remove `scsi_done()` Calls (Use Return Codes Instead)**  
   - **Problem:** The driver still uses `scsi_done()`, which is obsolete.  
   - **Solution:** Modify `queuecommand()` to return status codes instead of calling `scsi_done()`.  
   - **Tasks:**  
     - Locate all instances of `SCpnt->scsi_done(SCpnt)`.  
     - Modify `queuecommand()` to return:  
       - `SCSI_MLQUEUE_HOST_BUSY` (if queue is full)  
       - `0` (if the command is successfully queued)  
     - Test to ensure that command completion is handled correctly.  

**Fix Block Device Calls (`blkdev_get()`, `blkdev_put()`)**  
   - **Problem:** The driver uses outdated functions that no longer exist.  
   - **Solution:** Replace them with modern equivalents.  
   - **Tasks:**  
     - Replace `blkdev_get()` calls with:  
       ```c
       struct block_device *b = blkdev_get_by_dev(bdev->bd_dev, FMODE_READ, NULL);
       if (IS_ERR(b)) return PTR_ERR(b);
       ```
     - Replace `blkdev_put()` calls with:  
       ```c
       blkdev_put(bdev, FMODE_READ);
       ```
     - Remove `revalidate_disk()` calls (since revalidation is now automatic).  

---

### **Removing Unnecessary RAID/Array Code**
**✅ Status: In Progress**  

**Remove or Refactor RAID Handling (`pVDev->u.array`)**  
   - **Problem:** The driver references old RAID structures (`pVDev->u.array`), but this may be unnecessary if only JBOD is required.  
   - **Solution:** If only JBOD support is required, remove RAID-related logic entirely.  
   - **Tasks:**  
     - Identify all uses of `pVDev->u.array`, `pVDev->u.partition`, etc.  
     - **If RAID is not needed**, remove references entirely.  
     - **If RAID is needed**, update structures to align with modern SCSI layer RAID management.  

---

### **Testing & Debugging**
**✅ Status: Testing and debugging are continuous processes that should be performed after each fix to ensure the driver functions correctly.**  

**Test the Driver After Each Fix**  
   - **Tasks:**  
     - Compile the driver (`make -j$(nproc)`).  
     - Load the module (`insmod r750.ko`).  
     - Run `dmesg` to check for kernel errors, SCSI issues, or segmentation faults.  
     - Run `lsblk` and `fdisk -l` to verify drive detection.  
     - Test drive initialization using `hdparm -I /dev/sdX` to check ATA commands.  

 **Update Documentation & Changelog**  
   - **Tasks:**  
     - Log each fix in `CHANGELOG.md`.  
     - Add inline comments explaining major changes.  

---

## References

- **Original Driver Source:** R750_Linux_Src_v1.2.14-19_07_30  
- **HighPoint Rocket 750 Manual (2015)**  
- **Linux Kernel Documentation & Changelogs:** [https://www.kernel.org/](https://www.kernel.org/)

---

## Disclaimer

This project is **not affiliated** with HighPoint Technologies. All patches and modifications are community-driven efforts to ensure continued functionality of legacy hardware under modern kernels.

---

By following this roadmap and keeping detailed records in the changelog, we aim to fully modernize the Rocket 750 driver so that it compiles and runs correctly on Linux Kernel 6.8 while preserving as much of the original functionality as possible. Let's begin by updating the build system and progressively modernizing each subsystem.

---

### 📢 Note from the Maintainer

Okay, let’s get something out of the way right now: I have no idea what I’m doing.

I don’t want anyone stumbling across this repository and thinking, “Ah, yes, a proper project by someone who knows what they’re doing.” Because this is not that.

This is a complete and utter mess.

I have no experience with kernel drivers. I have no experience with patching old drivers to work on new kernels. I barely know what I’m looking at half the time. I’m learning as I go, and frankly, I’m 99% sure that most of my commits are making things worse, not better.

But here’s the thing:
I am working on this. Every. Single. Day.

I don’t know what I’m doing, but I’m going to keep slamming my head against this code until something, by pure statistical probability, starts working. Chaos theory. The whole “monkey typing Shakespeare” thing. If you make enough commits, eventually one of them has to work, right?

So, if you actually know what you’re doing, please, for the love of everything, submit a pull request. Tell me my code is garbage. Rip apart my changes. Revert my nonsense. I do not care. You cannot hurt my feelings because I already know how bad this is.

If you’re one of those people who sees this and thinks, “I could fix this in a weekend, but I don’t have time,” let me just say this:
  -	If this is a side project for you and you can only contribute every couple of weeks, just PR your changes.
  -	If you want to take over and you’re actually going to be active, by all means, do it.
  -	If you just want to laugh at my suffering, honestly, fair.

But if no one else is going to do it, then I guess it’s down to me.

I don’t care how long it takes. I’m going to get this driver working on Linux 6.8. Even if it’s purely by accident.

---

## FAQ

**Q: Why are you doing this?**

  Because I’m cheap.

**Q: No, really. Why?**

  I bought a 45 Drives 60-bay server from some guy on Facebook Marketplace. It came with HighPoint Rocket 750 HBAs, which are great except for one problem:

  -	The drivers only work up to Linux Kernel 5.x
  -	Unraid is Kernel 6.8
  -	I use consumer-grade hardware with only two PCIe x8 slots.

  If I want modern HBAs, I need three or four of them—which would cost a stupid amount of money and require new cables that don’t exist in the quantity I need.

  So instead of buying new hardware, I’ve decided to make the Rocket 750 drivers work on Linux 6.8 (despite having no experience with kernel drivers).

**Q: How long will this take?**

  Unknown. My strategy is simple: bash my head against the code until it compiles and runs. If I make enough commits, eventually something has to work, right?

**Q: Can I help?**

  - If you know how to fix it, PR your changes on GitHub.
  - If you have patches for older kernels, I’d love to see them.
  - If you want to just watch me suffer, fair enough.

  This is a community effort, so if you want to see Rocket 750s working on modern Linux, let’s make it happen.

**Q: I tried looking at your changelogs, and half of your commits are just updates to the README. Why do you keep making small incremental updates instead of just updating it all at once? It makes your commit history impossible to read.**

  Great question. There are actually two very good reasons for this:

  1.	I don’t actually know what I’m doing.

  If I change something in the README and later realize it was a terrible idea, I want an easy way to go back and see what it originally said.

  I could be removing vital information and have no way of realizing it at the time. Small commits make it easier to track what I’ve done and undo any mistakes.

  2.	This is my favorite way to procrastinate.

  I could be fixing the driver and doing something productive, but instead, I’m tweaking the README because it makes me feel like I’m making progress without actually doing any hard work.

  If you think this is a bad habit… yeah, it probably is. But here we are.

  So yeah, don’t judge me. Or do, whatever, I’m too busy rewriting the README for the 40th time to care.

**Q: You say you don’t know what you’re doing, but this README has a very clear roadmap. What’s the deal?**

  Great question. The roadmap is almost entirely made up.

  I mean, it’s based on research, AI suggestions, and general problem-solving logic, but I don’t actually know if any of this will work. I could spend months following this roadmap step by step and end up exactly where I started.

  I put this roadmap together because I needed some structure to avoid wandering in circles, but if you look at it and think, “Wow, this is all wrong,” please tell me. Rip it apart. Rewrite it. If you know a better way, do it.

  I fully expect that a good chunk of this roadmap is nonsense. That’s fine. I’ll keep iterating until something works.

**Q: Aren’t you that idiot who tried to put commercials back into commercial-free anime viewing?**

  Why yes, yes I am. Thank you for recognizing my contributions to reverse progress. While others were busy making streaming more convenient, I was deep in the coding mines, slamming my head against Python to make Plex feel like 2004 again.

  And now, instead of learning from my mistakes, I’ve decided to tackle this cursed project. Because clearly, I don’t value my time.

  But hey, if you’re here, that probably means you need this driver to work just as badly as I do. So let’s suffer together.



## License (Or Lack Thereof)

	“There is no license.”

This project follows The Weeb Coders’ core rule:

-	No license.
-	No restrictions.
-	No limitations.
-	No attribution required.
-	No copyright nonsense.

You can do whatever the hell you want with this code. Modify it, sell it, use it in a commercial project, burn it onto a CD and bury it in the desert—we do not care.

Everything here is free.

---

## Join Us on Discord

Want to chat, contribute, or just watch the chaos unfold?
Join The Weeb Coders on Discord: https://discord.com/invite/S7NcUdhKRD