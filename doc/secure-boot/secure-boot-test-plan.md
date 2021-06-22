# SONiC Secure Boot Test Plan

## Overview
The purpose of this test plan is to validate that SONiC switch supports secure boot.

## Test cases

### 1. Test to install an signed image
Test to install a signed image, expect the signed image can be installed.

**Steps:**
 * Copy a signed image
 * Run the sonic installer command: sonic_installer install <image>
 * Expect the installation successfully

### 2. Test to install an unsigned image
Test to install an unsigned image, expect the unsigned image never be installed when SONiC Secure Boot enabled.

**Steps:**
 * Create a fake unsigned image
 * Run the sonic installer command: sonic_installer install <image>
 * Expect the installation failed with image not signed

### 3. Test fast-reboot/warm-reboot from signed image into another signed image
Test the fast-reboot/warm-reboot into another signed image, make sure the secure boot option still exist.

**Steps:**
 * Copy a signed image
 * Run the sonic installer command: sonic_installer install <image>
 * Run fast-reboot
 * Check for secure_boot_enable=y in /proc/cmdline
 * Run warm-reboot
 * Check for secure_boot_enable=y in /proc/cmdline
 
### 4. Test not able to reboot from unsigned image when running fast-reboot/warm-reboot

**Steps:**
 * Replace the current image to an unsigned dummy image
 * Run fast-reboot
 * Expect the switch is not reboot with verification failure message
 * Run warn-reboot
 * Expect the switch is not reboot with verification failure message

### 6. Test boot from untrusted signed image

**Steps:**
 * Sign image using self-signed certificate not installed in switch
 * Install the image
 * Reboot
 * Expect system hanging on the bootloader
 
### 7. Test CA certificate expired
Test CA rotation, CA expired need to change another CA

**Steps:**
 * Create a test CA certificate that will expired in a short time
 * Sign the sonic image by a certificate signed by the CA
 * Install the CA to the switch
 * Install the sonic image
 * Wait until the CA expired
 * Start to boot the switch
 * Expect system hanging on the bootloader

 
### 8. Test tempered image

**Steps:**
 * Change a bit in the image
 * Reboot the SONiC switch
 * Expect system hanging on the bootloader

### 9. Test no executable files in rw folder after reboot
If there are any files with -x option in rw folder, the option will be removed after the SONiC switch reboot.

**Steps:**
 * Add files: /var/core/test /home/admin/test /home/admin/.test with +x option 
 * Reboot the SONiC switch
 * Expect the +x option has removed from the files

### 10. Test all files not in allowlist will be removed after reboot
If there are any files not in the allowlist, the files will be removed after the SONiC switch reboot.

**Steps:**
 * Add a file in /etc/test /etc/init.d/test /etc/init.d/.test
 * Reboot the SONiC switch
 * Expect the files are removed
 
### 11. Test the file in allowlist will be persisted
All the files in the allowlist will be persisted, the content of the files keeps the same after the SONiC switch reboot.

**Steps:**
 * Add a host config in /etc/hosts
 * Reboot the SONiC switch
 * Expect the change config exists after reboot

 
### 12. Remove unexpected config files
Test if there are any new config files added in test cases. If someone adds a config file for a new feature, the config file should be added in the allowlist file. Expect the build will be broken if not added.

**Steps:**
 * Add a pytest fixture to all test cases
 * The pytest fixture runs before and after every test case, and detect the file change in rw/etc folder
 * Expect all the change should be in the allowlist

### 13. Change the allowlist file
Test the scenario to change the allowlist config file, make sure it works as expected.

**Steps:**
 * Add a new config file in the allowlist
 * Rebuild the signed image and install it
 * Change the config file
 * Reboot the switch
 * Expect the config still existing

### 14. Test to reboot when the disk is full
The disk is full, test the reboot process will not write anything to the disk.

**Steps:**
 * Generate files to use all the disk space in the switch
 * Expect the reboot will be successful
