# HighPoint R750 Driver Makefile
# Copyright (C) 2012 HighPoint Technologies, Inc.
# All Rights Reserved.

KERNELDIR := /lib/modules/6.8.0-40-generic/build
PWD := /home/ubuntu/Documents/r750_source/product/r750/linux

# Driver module name
obj-m := r750.o

# Corrected source file paths (relative to M=$(PWD))
r750-objs := ../../../osm/linux/os_linux.o \
             ../../../osm/linux/osm_linux.o \
             ../../../osm/linux/div64.o \
             ../../../osm/linux/hptinfo.o \
             ./config.o \
             ../../../lib/linux_64mpa/free-x86_64-regparm0/him_r750.o \
             ../../../lib/linux_64mpa/free-x86_64-regparm0/ldm_la.o

# Compiler flags (FULL ABSOLUTE PATHS)
EXTRA_CFLAGS += -I/home/ubuntu/Documents/r750_source/osm/linux \
                -I/home/ubuntu/Documents/r750_source/product/r750/linux \
                -I/home/ubuntu/Documents/r750_source/inc \
                -I/home/ubuntu/Documents/r750_source/inc/linux_64mpa

all:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) clean
	rm -f $(PWD)/*.ko $(PWD)/*.o $(PWD)/*.mod.o $(PWD)/*.mod.c $(PWD)/Module.symvers $(PWD)/modules.order

install:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) modules_install
	depmod -a

uninstall:
	rm -f /lib/modules/$(shell uname -r)/extra/r750.ko
	depmod -a