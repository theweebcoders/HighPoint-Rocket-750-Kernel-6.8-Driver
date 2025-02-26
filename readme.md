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

## Codebase Structure

```
├── inc
│   ├── linux_64mpa
│   │   ├── Makefile.def   # Kernel compatibility restrictions (to be updated)
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
│       ├── install.sh
│       ├── os_linux.c
│       ├── osm_linux.c
│       ├── osm_linux.h
│       └── patch.sh
├── product
│   └── r750
│       └── linux
│           ├── config.c  # Driver initialization and configuration logic
│           └── Makefile
└── install.sh           # Installer script
```

---

## 🔧 Modernization Roadmap

The following plan outlines the detailed steps required to modernize the driver for Linux Kernel 6.8.

### 1. Update Kernel-Dependent Code

#### A. Replace or Stub `virt_to_bus()`
- **Issue:** `virt_to_bus()` was removed in favor of modern DMA mapping.
- **Plan:**  
  - Replace calls with proper DMA mapping functions.  
  - For a temporary stub, you may define:
    ```c
    #if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,0)
    #include <linux/dma-mapping.h>
    #define virt_to_bus(x) virt_to_phys(x)  /* Stub: ideally use dma_map_single() */
    #endif
    ```
- **Rationale:** Ensures correct DMA address calculation under modern kernels.

#### B. Remove Outdated SCSI Macros & Fields
- **Issue:** Legacy macros such as `SCSI_DISKx_MAJOR`, `CHECK_CONDITION`, `DRIVER_SENSE`, etc., and fields like `SCp` in `struct scsi_cmnd` no longer exist.
- **Plan:**  
  - Remove or #ifdef–out code that references these macros.  
  - Replace error reporting with modern methods, for example:
    ```c
    #include <scsi/scsi_cmnd.h>
    scsi_build_sense_buffer(SCpnt->sense_buffer, SCSI_SENSE_BUFFERSIZE,
                            /* sense key */ 0x3, /* ASC */ 0x11, /* ASCQ */ 0x04);
    SCpnt->result = (DID_ERROR << 16);
    ```
- **Rationale:** Aligns error handling with current SCSI mid-layer expectations.

#### C. Modernize the SCSI Host Template
- **Issue:** Deprecated fields in `scsi_host_template` (e.g. `detect`, `release`, `unchecked_isa_dma`, `ioctl`) and legacy command completion methods.
- **Plan:**  
  - Remove these fields and update the template as follows:
    ```c
    static struct scsi_host_template r750_template = {
        .module                  = THIS_MODULE,
        .name                    = "Rocket750",
        .queuecommand            = r750_queuecommand,  // Updated routine
        .eh_device_reset_handler = r750_eh_devreset,   // Optional
        .eh_bus_reset_handler    = r750_eh_busreset,   // Optional
        .can_queue               = 32,
        .cmd_per_lun             = 1,
        .max_sectors             = 128,
        .this_id                 = -1,
        .slave_configure         = r750_slave_configure,
        .max_lun                 = 1,
        .proc_name               = "r750",
        // Other fields as required...
    };
    ```
- **Rationale:** Ensures proper integration with the modern SCSI subsystem.

#### D. Update PCI & Interrupt Handling
- **Issue:** Legacy PCI API calls and interrupt flags (e.g., `SA_SHIRQ`) must be updated.
- **Plan:**  
  - Use the current functions such as `pci_set_dma_mask()`, ensuring error handling follows modern conventions.
  - Replace legacy interrupt flags with modern ones (e.g. use `IRQF_SHARED`).
  - Update timer initialization to use `timer_setup()`.
- **Rationale:** Guarantees correct device handling and IRQ management on kernel 6.8.

#### E. Refactor DMA and SG List Code
- **Issue:** The driver’s scatter–gather (SG) list building and DMA unmapping use outdated methods.
- **Plan:**  
  - Replace custom SG list handling with the standard Linux scatterlist API:
    ```c
    static int r750_build_sglist(struct scsi_cmnd *SCpnt, struct scatterlist **psg, int *nents)
    {
         *psg = scsi_sglist(SCpnt);
         *nents = scsi_sg_count(SCpnt);
         return 0;
    }
    ```
  - Replace legacy unmap calls with `dma_unmap_single()` or `pci_unmap_sg()`.
- **Rationale:** Uses standardized APIs that are maintained in modern kernels.

#### F. Remove Legacy Block Device & Revalidation Code
- **Issue:** Old functions such as `bdget()`, `blkdev_get()`, and direct usage of `bd_openers` no longer work.
- **Plan:**  
  - Remove or stub out code that attempts to directly access these fields.
  - Rely on the SCSI and block subsystems to handle revalidation automatically.
- **Rationale:** Modern kernel subsystems handle device revalidation internally.

---

### 2. Update the Build System

#### A. Clean Up Makefiles and Scripts
- **Issue:** Build scripts and makefiles reference kernel versions (2.4/2.6) and use deprecated flags.
- **Plan:**  
  - Update `Makefile.def` and `product/r750/linux/Makefile` to remove version checks and adapt paths for Kernel 6.8 (e.g., set `KERNELDIR` to `/lib/modules/6.8.0-40-generic/build`).
  - Remove unused macros such as `-DMODVERSIONS` if not needed.
  - Update installation scripts (e.g. `install.sh` and `patch.sh`) to reflect modern paths and dependencies.
- **Rationale:** Ensures that the module builds cleanly using the modern kernel build system.

#### B. Update Module Parameters & Sysfs Interfaces
- **Issue:** Legacy module parameter declarations and sysfs attribute handling may be outdated.
- **Plan:**  
  - Use `module_param()` and `MODULE_PARM_DESC()` for all module parameters.
  - Update any sysfs attribute definitions to use the current kernel API.
- **Rationale:** Provides a modern interface for configuration and runtime control.

---

### 3. Refactor and Simplify the Driver Architecture

#### A. Rework the LDM and Array Layers
- **Issue:** The driver’s internal “Logical Device Manager (LDM)” and RAID array support use very old logic.
- **Plan:**  
  - Evaluate whether full RAID/array support is required. If only basic JBOD functionality is needed, consider removing or stubbing out the array and partition logic.
  - Otherwise, update the union in the VDEV structure (e.g. `pVDev->u.array`) to match modern conventions.
- **Rationale:** Simplifies maintenance and reduces the risk of legacy bugs.

#### B. Modernize Error Handling and Sense Data Reporting
- **Issue:** The driver sets sense data manually using outdated macros.
- **Plan:**  
  - Replace manual sense-buffer code with modern helper functions such as `scsi_build_sense_buffer()`.
  - Update result codes to use modern SCSI result fields.
- **Example:**
    ```c
    scsi_build_sense_buffer(SCpnt->sense_buffer, SCSI_SENSE_BUFFERSIZE,
                            /* sense key */ 0x3, /* ASC */ 0x11, /* ASCQ */ 0x04);
    SCpnt->result = (DID_ERROR << 16);
    ```
- **Rationale:** Aligns error reporting with the modern SCSI mid-layer.

#### C. Remove Legacy Private Data Storage
- **Issue:** The driver references the now-removed `SCp` field in `struct scsi_cmnd` to store driver-private data.
- **Plan:**  
  - Remove these references and use alternative storage methods (such as using the hostdata pointer allocated with `scsi_host_alloc()` or dynamic memory).
- **Rationale:** Ensures that the driver is compatible with current SCSI command structures.

---

### 4. Documentation & Changelog

#### A. Inline Documentation
- **Action:** Add inline comments in each file explaining major changes. For example:
  ```c
  /* Replaced virt_to_bus() with dma_map_single() because virt_to_bus() is deprecated.
   * This is a temporary stub; a full DMA mapping will be implemented in later revisions.
   */
  ```
  
#### B. Changelog
Create a `CHANGELOG.md` with entries such as:

- **Entry 1:**  
  - *Date:* [Insert Date]  
  - *Change:* Replaced all `virt_to_bus()` calls with a stub using `virt_to_phys()`.  
  - *Reason:* `virt_to_bus()` is no longer available; this is a temporary measure pending a full DMA API integration.

- **Entry 2:**  
  - *Date:* [Insert Date]  
  - *Change:* Removed legacy SCSI macros (`SCSI_DISKx_MAJOR`, `CHECK_CONDITION`, etc.) and stubbed out the associated logic.  
  - *Reason:* These macros are obsolete; modern SCSI drivers handle these aspects internally.

- **Entry 3:**  
  - *Date:* [Insert Date]  
  - *Change:* Updated the SCSI host template by removing deprecated fields and implementing a new `queuecommand()` function.  
  - *Reason:* Kernel 6.8 uses a different SCSI mid-layer structure.

- **Entry 4:**  
  - *Date:* [Insert Date]  
  - *Change:* Updated PCI, DMA, and interrupt handling code to use modern APIs.  
  - *Reason:* To ensure proper device operation and compatibility with Linux 6.8.

- **Entry 5:**  
  - *Date:* [Insert Date]  
  - *Change:* Updated the build system (Makefiles and install scripts) to remove obsolete version checks and compiler flags.  
  - *Reason:* To support the new kernel build environment.

---

## Next Steps

1. **Phase 1 – Build System Update:**  
   - Modify `Makefile.def` and the main Makefile to remove version limits and update paths for Kernel 6.8.
   - Ensure that the installer scripts work with modern distributions.

2. **Phase 2 – Code Modernization:**  
   - Replace all deprecated calls (e.g., `virt_to_bus()`, legacy SCSI fields, PCI API differences).
   - Refactor the SCSI host template and `queuecommand()` implementation.
   - Update DMA and SG list handling using the modern Linux API.

3. **Phase 3 – Testing & Documentation:**  
   - Compile the driver against the 6.8 headers.
   - Document every change in a detailed changelog and inline comments.
   - Prepare a final working version for publication.

---

## References

- **Original Driver Source:** R750_Linux_Src_v1.2.14-19_07_30  
- **HighPoint Rocket 750 Manual (2015)**  
- **Linux Kernel Documentation & Changelogs:** [https://www.kernel.org/](https://www.kernel.org/)

---

## Disclaimer

This project is **not affiliated** with HighPoint Technologies. All patches and modifications are community-driven efforts to ensure continued functionality of legacy hardware under modern kernels.

---

By following this roadmap, we aim to fully modernize the Rocket 750 driver so that it compiles and runs correctly on Linux Kernel 6.8 while maintaining as much of the original functionality as possible. Let's begin by updating the build system and progressively modernizing each subsystem.