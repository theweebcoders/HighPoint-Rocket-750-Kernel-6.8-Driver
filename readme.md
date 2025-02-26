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

---

## Current Limitations

### 🚨 Why the Driver Fails on Kernel 6.8

1. **Kernel Version Check Blocks 6.x:**  
   - The **Makefile.def** explicitly disallows any kernel above **5.x**.
   - It must be modified to allow building under **6.8**.

2. **Deprecated or Removed Kernel Functions:**  
   - Several functions (e.g. `revalidate_disk()`, `blkdev_get()`, `bdget()`) and legacy fields in structures (e.g. in `scsi_cmnd` and `scsi_host_template`) were removed or significantly changed.

3. **Changes to PCI and SCSI Handling:**  
   - The driver relies on legacy PCI and SCSI interfaces that have been replaced by modern APIs.

---

## Compatibility Considerations

Community reports indicate that while the original driver (v1.2.14) was patched to support up to kernel 5.2.x, several changes within the 4.x and 5.x series have introduced challenges:

- **SCSI Subsystem:**  
  - Legacy SCSI codes (e.g. `CHECK_CONDITION`, `GOOD`, etc.) were replaced with SAM codes somewhere in the 4.x series, leading to compile-time issues on newer kernels.

- **Kernel 5.x and Beyond:**  
  - Changes in build mechanisms (e.g. switching from `SUBDIR` to `M` in 5.3).
  - Modifications to block device management (e.g. merging of `bdget()` and `blkdev_get()` into `blkdev_get_by_dev()` and removal of `revalidate_disk()` in 5.8).
  - Updates in the `scsi_host_template` structure (e.g. removal of `unchecked_isa_dma` in 5.13) and legacy status codes (removed in 5.14).

- **Practical Recommendation:**  
  - Although some community patches exist for kernel 5.x (e.g. a patch for 5.15), these may require further testing.
  - For maximum compatibility without extensive patching, a Linux distribution with a 4.x kernel (for example, **Ubuntu 16.04 LTS** with kernel 4.4) is recommended.
  - **Security Note:** Using an older distribution requires additional security measures (network isolation, strict firewall rules, minimal services) due to the end-of-life status of such distributions.

---

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

---

## 🔧 Modernization Roadmap

The following steps outline the process to modernize the driver for Linux Kernel 6.8:

### 1. Update Kernel-Dependent Code

- **Replace or Stub `virt_to_bus()`:**  
  Replace legacy DMA conversion calls with modern DMA mapping functions or temporary stubs.

- **Remove Outdated SCSI Macros & Fields:**  
  Eliminate legacy macros such as `SCSI_DISKx_MAJOR` and fields like `SCp` from `struct scsi_cmnd`.  
  Update error reporting to use functions like `scsi_build_sense_buffer()`.

- **Modernize the SCSI Host Template:**  
  Remove deprecated fields (`detect`, `release`, etc.) and implement a modern `queuecommand()` routine.

- **Update PCI & Interrupt Handling:**  
  Replace legacy PCI API calls and interrupt flags (e.g. `SA_SHIRQ`) with modern equivalents such as `IRQF_SHARED` and use updated timer APIs (`timer_setup()`).

- **Refactor DMA and SG List Code:**  
  Use the standard Linux scatterlist API and update block device get/put operations through helper functions.

- **Remove Legacy Block Device & Revalidation Code:**  
  Rely on the SCSI and block subsystems for device revalidation.

### 2. Update the Build System

- **Clean Up Makefiles and Scripts:**  
  Modify `Makefile.def` to allow kernel 6.x and update paths (e.g. `KERNELDIR`), remove obsolete flags, and adjust the installer scripts.

- **Update Module Parameters & Sysfs Interfaces:**  
  Use modern macros (`module_param()`, `MODULE_PARM_DESC()`) and current sysfs APIs.

### 3. Refactor and Simplify the Driver Architecture

- **Rework the LDM and Array Layers:**  
  Evaluate and simplify RAID/array support (or stub if only JBOD is required).

- **Modernize Error Handling and Sense Data Reporting:**  
  Replace manual sense-buffer handling with modern helper functions.

- **Remove Legacy Private Data Storage:**  
  Eliminate reliance on removed fields like `SCp` and use alternative storage (e.g., hostdata pointers).

### 4. Documentation & Changelog

- **Inline Documentation:**  
  Add comments in each file explaining the purpose of modifications.

- **Changelog:**  
  Maintain a `CHANGELOG.md` documenting every major change.

---

## Next Steps

1. **Phase 1 – Build System Update:**  
   - Confirm that the updated Makefile and installer scripts work across target distributions.

2. **Phase 2 – Code Modernization:**  
   - Continue replacing deprecated kernel calls and updating interfaces as needed.

3. **Phase 3 – Testing & Documentation:**  
   - Compile the driver against the 6.8 headers.
   - Perform runtime tests and update documentation with any further modifications.
   - Prepare the final working version for release.

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

📢 Note from the Maintainer

Okay, let’s get something out of the way right now: I have no idea what I’m doing.

I don’t want anyone stumbling across this repository and thinking, “Ah, yes, a proper project by someone who knows what they’re doing.” Because this is not that.

This is a complete and utter mess.

I have no experience with kernel drivers. I have no experience with patching old drivers to work on new kernels. I barely know what I’m looking at half the time. I’m learning as I go, and frankly, I’m 99% sure that most of my commits are making things worse, not better.

But here’s the thing:
I am working on this. Every. Single. Day.

I don’t know what I’m doing, but I’m going to keep slamming my head against this code until something, by pure statistical probability, starts working. Chaos theory. The whole “monkey typing Shakespeare” thing. If you make enough commits, eventually one of them has to work, right?

So, if you actually know what you’re doing, please, for the love of everything, submit a pull request. Tell me my code is garbage. Rip apart my changes. Revert my nonsense. I do not care. You cannot hurt my feelings because I already know how bad this is.

If you’re one of those people who sees this and thinks, “I could fix this in a weekend, but I don’t have time,” let me just say this:
	•	If this is a side project for you and you can only contribute every couple of weeks, just PR your changes.
	•	If you want to take over and you’re actually going to be active, by all means, do it.
	•	If you just want to laugh at my suffering, honestly, fair.

But if no one else is going to do it, then I guess it’s down to me.

I don’t care how long it takes. I’m going to get this driver working on Linux 6.8. Even if it’s purely by accident.

---

## FAQ

**Q: Why are you doing this?**

  Because I’m cheap.

**Q: No, really. Why?**

  I bought a 45 Drives 60-bay server from some guy on Facebook Marketplace. It came with HighPoint Rocket 750 HBAs, which are great except for one problem:

    •	The drivers only work up to Linux Kernel 5.x
    •	Unraid is Kernel 6.8
    •	I use consumer-grade hardware with only two PCIe x8 slots.

  If I want modern HBAs, I need three or four of them—which would cost a stupid amount of money and require new cables that don’t exist in the quantity I need.

  So instead of buying new hardware, I’ve decided to make the Rocket 750 drivers work on Linux 6.8 (despite having no experience with kernel drivers).

**Q: How long will this take?**

  Unknown. My strategy is simple: bash my head against the code until it compiles and runs. If I make enough commits, eventually something has to work, right?

**Q: Can I help?**

    •	If you know how to fix it, PR your changes on GitHub.
    •	If you have patches for older kernels, I’d love to see them.
    •	If you want to just watch me suffer, fair enough.

  This is a community effort, so if you want to see Rocket 750s working on modern Linux, let’s make it happen.

**Q: Aren’t you that idiot who tried to put commercials back into commercial-free anime viewing?**

  Why yes, yes I am. Thank you for recognizing my contributions to reverse progress. While others were busy making streaming more convenient, I was deep in the coding mines, slamming my head against Python to make Plex feel like 2004 again.

  And now, instead of learning from my mistakes, I’ve decided to tackle this cursed project. Because clearly, I don’t value my time.

  But hey, if you’re here, that probably means you need this driver to work just as badly as I do. So let’s suffer together.

---

## License (Or Lack Thereof)

	“There is no license.”

This project follows The Weeb Coders’ core rule:
	•	No license.
	•	No restrictions.
	•	No limitations.
	•	No attribution required.
	•	No copyright nonsense.

You can do whatever the hell you want with this code. Modify it, sell it, use it in a commercial project, burn it onto a CD and bury it in the desert—we do not care.

Everything here is free.

---

Join Us on Discord

Want to chat, contribute, or just watch the chaos unfold?
Join The Weeb Coders on Discord: https://discord.com/invite/S7NcUdhKRD