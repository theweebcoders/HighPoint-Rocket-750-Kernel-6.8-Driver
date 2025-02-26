# HighPoint Rocket 750 Kernel 6.8 Driver Patching

## Overview
This repository contains the **open-source driver for the HighPoint Rocket 750 HBA**, a 40-port SATA 6Gb/s PCI-Express 2.0 x8 Host Bus Adapter (HBA). This driver is officially provided by HighPoint but only supports **Linux kernel versions up to 5.x**.

### **Purpose of This Project**
The goal of this project is to **patch the existing Rocket 750 driver** so that it compiles and functions correctly under **Linux Kernel 6.8**.

## **Official Specifications**

### **📌 HighPoint Rocket 750 Specifications**
- **HBA Type:** 40-Port SATA 6Gb/s PCI-Express 2.0 x8 HBA
- **Connector Type:** SFF-8087 (10 Mini-SAS Ports)
- **Maximum Drives Supported:** 40 SATA Devices
- **Max Storage Capacity:** 320TB
- **Power Consumption:** 4W max (12V), 1W max (3.3V)
- **Management Features:**  
  - SGPIO, Active/Fail LED, I2C interface  
  - Hot-Plug & Hot-Swap Support  
  - Staggered Drive Spin-Up

### **📌 Official Operating System Support (From 2015 Manual)**
- **Windows:** 8 / 2012 / 7 / 2008R2  
- **Linux Distributions:**  
  - ✅ **Ubuntu (Older releases)**
  - ✅ **SUSE Linux Enterprise Server (SLES)**
  - ✅ **Red Hat Enterprise Linux (RHEL)**
  - ✅ **OpenSUSE**
  - ✅ **Fedora**
  - ✅ **FreeBSD**

## **Current Limitations**

### **🚨 Why the Driver Fails on Kernel 6.8**

1. **Kernel Version Check Blocks 6.x:**  
   - The **Makefile.def** explicitly disallows any kernel above **5.x**.
   - We must **modify `Makefile.def`** to allow building under **6.8**.

2. **Deprecated or Removed Kernel Functions:**  
   Several functions used in the driver source code were **removed or replaced** in later kernel versions:
   - `revalidate_disk()` → Removed in **Kernel 5.9**
   - `blkdev_get()` → Removed in **Kernel 5.8**
   - `bdget()` → Removed in **Kernel 5.15**
   - `scsi_host_template` → Changed in **Kernel 5.13**

3. **Changes to PCI and SCSI Handling in the Kernel**  
   - Struct changes affect how the driver interacts with **PCIe devices** and the **SCSI subsystem**.

## **📁 Codebase Structure**

The repository consists of the following key directories and files:

```
├── inc
│   ├── linux_64mpa
│   │   ├── Makefile.def  # Kernel compatibility restrictions
│   │   └── osm.h  # Kernel interface definitions
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
│           ├── config.c  # Driver initialization logic
│           └── Makefile
└── install.sh  # Installer script
```

## **🔧 Roadmap for Patching**

### **Phase 1: Modify Build System**
✅ Allow `Makefile.def` to recognize Kernel 6.x  
🔲 Identify and replace deprecated kernel APIs  
🔲 Patch `scsi_host_template` struct  

### **Phase 2: Code Updates & Compilation**
🔲 Replace removed kernel functions with their modern equivalents  
🔲 Compile `hptnr.ko` under Kernel 6.8  
🔲 Attempt to load `modprobe hptnr`  
🔲 Verify card and disk detection  

### **Phase 3: Automate & Publish**
🔲 Document all changes (`PATCH_NOTES.md`)  
🔲 Create an automated installer for modern distros  
🔲 Publish a final working version  

## **🔗 References**
- **Original Driver:** `R750_Linux_Src_v1.2.14-19_07_30`
- **HighPoint Rocket 750 Manual (2015)**
- **Linux Kernel Changelogs:** [https://www.kernel.org/](https://www.kernel.org/)

---

## **⚠️ Disclaimer**
This project is **not affiliated** with HighPoint Technologies.  
All patches are community-driven efforts to keep legacy hardware functional.

---

### **🚀 Next Steps**
Now that the README is updated with all relevant information, we can begin **patching the driver for Kernel 6.8**. The first step will be modifying `Makefile.def` to allow compilation on **Linux Kernel 6.x**.

Let’s proceed!