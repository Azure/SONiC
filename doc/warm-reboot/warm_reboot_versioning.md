# Nomenclature
## From version:
- the version being upgraded out from. 
- the version of the SONiC image running on the device before upgrading.


## To version:
- the version upgraded into.
- the version of SONiC image running on the device after successful upgrading.


# Problem statement
Warm reboot is most commonly intended for none disruptively uprading SONiC from one version to another.

With versioning, we are trying to solve following problems:

1. Control upgrading path, meaning which version can upgrade to which version.
2. Provide post upgrading guidance for states restoration.


# Upgrading path control
## Options:
### Persisted with the 'to' image.
- White list all the version numbers that are allowed to be 'from' version.
- Black list all the version numbers that are not allowed to be 'from' version.
- Regex is allowed.

### Persisted with the upgrading harness


## Current status
- Need decision on which option to take after weighting pros and cons.


# Post upgrading help
## Who/Why do we need this help?
- Warm reboot requires 'from' version save status so that 'to' version could restore states and carry on.
- Some modules restore state by reconstructing states, these modules doesn't need any help. e.g. snmp, pmon, dhcp-relay.
- Some modules retore state by save state by 'from' version with certain format to file(s), 'to' version read the file(s) and restore state. We recommend these modules put a version number in the data file, so the 'to' version knows exactly which version created the data file.
- Database contain is a special case, we need to save some contents from redis database. But we don't have control over the dump file format. More importantly, we might have to act differently with the dump file. For this purpsoe, we are going to add a versioning file to help database container.

## Database container versioning design
- We add a file to sonic-buildimage repo.
  - File name: warm-reboot-versions
  - Contents: DATABASE_VERSION=1.0 (format is bash scriptable format because it is most likely being consumed by a bash script).
  - This file is installed in /etc/sonic/
  - warm-reboot script copy this file from /etc/sonic to /host/warmboot/
  - database.sh can read this file and do things accordingly.
