# SONIC ARM Architecture support

[![Marvell Technologies](https://www.marvell.com/content/dam/marvell/en/rebrand/marvell-logo3.svg)](https://www.marvell.com/)

# Description

  - This document describes enhancement in SONIC build script to support ARM32 and ARM64
  
Support for ARM architecture needs changes in the following modules

  - sonic-slave
  - dockers
  - rules
  - Makefile
  - Buildscript
  - Repo list
  - Onie Build



### User Input

Similar to configuring the platform in the Make, architecture should be user driven.

* [SONIC_ARCH] - make configure PLATFORM=[ASIC_VENDOR] PLATFORM_ARCH=[armhf]
* Default is X86_64

### Dockers
Since all the modules and code are compiled inside docker environment, the docker image should be based on multiarch/[distribution]-[arm_arch] 

Below dockers use the debian distribution which will now be based on the CPU Architecture distribution.
```sh
dockers/docker-base
dockers/docker-base-stretch
dockers/docker-ptf
```

### Developer Notes
Following are the variables used in make files
PLATFORM_ARCH : specifies the target architecture, if not set amd64 is chosen
CONFIGURED_ARCH : In Makefiles, no where amd64 should be hardcoded, instead $(CONFIGURED_ARCH) has to be used  
```sh
Example: in place of amd64 in below target CONFIGURED_ARCH is replaced
LINUX_IMAGE = linux-image-$(KVERSION)_$(KERNEL_VERSION)-$(KERNEL_SUBVERSION)_amd64.deb
LINUX_IMAGE = linux-image-$(KVERSION)_$(KERNEL_VERSION)-$(KERNEL_SUBVERSION)_$(CONFIGURED_ARCH).deb
```


### SONIC Slave Docker

sonic-slave docker provides build environment for the rest of the dockers, it should be able to run the different architecture on the host cpu architecture.

To do such cross compilation, we can make use of binfmt-misc to run target arch binary using qemu-static binary to run on the host cpu architecture. 

```sh
sonic-slave-arm64
sonic-slave-armhf
```

qemu static binaries need to be installed and docker for multiarch/qemu-user-static:register is enabled to run.

### Miscellaneous

Architecture specific packages need to installed or ignored.
Like ixgbe and grub are specific to X86 architecture, which need to be excluded.


### Platform

Same platform or board can have variants in CPU vendor. To address this, platform can be made ARCH specific, and customized changes can be added in this platform specific make infra.

```sh
platform/marvell-armhf/docker-syncd-mrvl-rpc.mk
platform/marvell-armhf/docker-syncd-mrvl-rpc/99-syncd.conf
platform/marvell-armhf/docker-syncd-mrvl-rpc/Dockerfile.j2
platform/marvell-armhf/docker-syncd-mrvl-rpc/ptf_nn_agent.conf
platform/marvell-armhf/docker-syncd-mrvl.mk
platform/marvell-armhf/docker-syncd-mrvl/Dockerfile.j2
platform/marvell-armhf/docker-syncd-mrvl/start.sh
platform/marvell-armhf/docker-syncd-mrvl/supervisord.conf
platform/marvell-armhf/docker-syncd-mrvl/syncd.sh
platform/marvell-armhf/libsaithrift-dev.mk
platform/marvell-armhf/linux-kernel-armhf.mk
platform/marvell-armhf/one-image.mk
platform/marvell-armhf/platform.conf
platform/marvell-armhf/rules.mk
platform/marvell-armhf/sai.mk
platform/marvell-armhf/sai/Makefile
```

#### Rule/makefile

Hardcoded "amd64" need to be replaced with Makefile variable which hold the target architecture.
* amd64
* armhf
* arm64

```sh
rules/bash.mk
rules/docker-base-stretch.mk
rules/docker-base.mk
rules/docker-ptf.mk
rules/docker-snmp-sv2.mk
rules/frr.mk
rules/gobgp.mk
rules/hiredis.mk
rules/iproute2.mk
rules/isc-dhcp.mk
rules/libnl3.mk
rules/libteam.mk
rules/libyang.mk
rules/linux-kernel.mk
rules/lldpd.mk
rules/lm-sensors.mk
rules/mpdecimal.mk
rules/python3.mk
rules/quagga.mk
rules/radvd.mk
rules/redis.mk
rules/sairedis.mk
rules/smartmontools.mk
rules/snmpd.mk
rules/socat.mk
rules/swig.mk
rules/swss-common.mk
rules/swss.mk
rules/tacacs.mk
rules/telemetry.mk
rules/thrift.mk
slave.mk
src/bash/Makefile
src/hiredis/Makefile
src/iproute2/Makefile
src/isc-dhcp/Makefile
src/libnl3/Makefile
src/libteam/Makefile
src/lm-sensors/Makefile
src/mpdecimal/Makefile
src/python3/Makefile
src/radvd/Makefile
src/redis/Makefile
src/smartmontools/Makefile
src/snmpd/Makefile
src/socat/Makefile
src/tacacs/nss/Makefile
src/tacacs/pam/Makefile
src/thrift/Makefile

```

### Repo list
Below repo sources list need to updated as the azure debian repo doesn't have arm packages 


```sh
files/apt/sources.list-armhf
files/build_templates/sonic_debian_extension.j2

```

#### Onie Image

Onie image configuration and build script should be updated for the uboot specific environment for ARM.
Update target platform for Onie image platform configuration in onie image conf.
 - onie-image.conf for AMD64
 - onie-image-armhf.conf for ARMHF
 - onie-image-arm64.conf for ARM64
Onie platform config file will chosed based on the target platform 
 - platform/<TARGET_PLATFORM>/platform.conf
 platform.conf will be used by the onie installer script to install the onie image
Onie Installer scripts
 - installer/x86_64/install.sh 
 - installer/arm64/install.sh
 - installer/armhf/install.sh

SONIC Image installation is driven by these onie installer scripts which does
 - Boot loader update with image boot details
 - Partition the primary disk if not formatted/partitioned
 - Extract sonic image in the mounted disk under /host directory

For different platforms, the primary storage device may vary, unlike X86 platforms which mainly use varieant of sata disks,
ARM platform can also use NAND/NOR flash or SD/MMC cards
The platform dependent partition scheme is moved to platform/<TARGET_PLATFORM>/platform.conf, where
selecting primary storage medium, partitioning, formatting, and mounting is taken care.
The mount path is provided to the generic SONIC installer script, which does common functionalities of extracting image, and copying files.

X86 uses grub as its bootloader, where ARM can use Uboot or proprietary bootloaders.
Bootloader configuration for boot image details are also updated in platform.conf

#### Sonic Installer

SONIC upgrade from SONIC uses python scripts to access bootloader configuration to update the boot image details, to support
image upgrade, image deletion, and change boot order.
For ARM Uboot firmware utilities is used to access boot configuration, as in grub for X86.
 - sonic_installer/main.py

### Kernel ARM support

Submodule sonic-linux-kernel Makefile and patch need to be updated to compile for respective ARM architecture. As kernel .config will be generated using debian build infra, dpkg env variables need to properly updated to select the architecture.

 - src/sonic-linux-kernel
 
### Custom Kernel  (Expert Mode)

Based on architecture the linux kernel may vary and need to be changed to custom kernel rather that the SONIC default kernel version.
This can be addressed in platform specific makefiles.

 - platform/marvell-armhf/linux-kernel-armhf.mk


### Usage for ARM Architecture
To build Arm32 bit for (ARMHF) plaform

    # Execute make configure once to configure ASIC and ARCH
    make configure PLATFORM=[ASIC_VENDOR] SONIC_ARCH=armhf
    **example**:
    make configure PLATFORM=marvell-armhf SONIC_ARCH=armhf

To build Arm64 bit for plaform

    # Execute make configure once to configure ASIC and ARCH
    make configure PLATFORM=[ASIC_VENDOR] SONIC_ARCH=arm64
    **example**:
    make configure PLATFORM=marvell-arm64 SONIC_ARCH=arm64

---- 
Author
======
Antony Rheneus [arheneus@marvell.com]
Copyright Marvell Technologies

