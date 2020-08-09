[Home](index.md)

# Introduction
---

## Kernel vs. Distribution
---
The Linux **kernel** is the core interface between a computer’s hardware and its processes. 
* A full Linux **distribution** consists of the **kernel plus a number of other software tools** for file-related operations, user management, and software package management. Each of these tools provides a part of the complete system. Each tool is often its own separate project, with its own developers working to perfect that piece of the system. Linux distributions may be based on different kernel versions. For example, the very popular RHEL 7 distribution is based on the 3.10 kernel. 

Linux kernels are identified by a set of four numbers:
	- Kernel Version
	- Major Revision
	- Minor Revision
	- Security Patches & Bug Fixes

```bash
# Check  version of running kernel
uname -srm
hostnamectl
cat /proc/version
```

```bash
# Check all installed kernel versions
rpm -q kernel
yum list kernel
```

```bash
# Check OS version
cat /etc/os-release
hostnamectl
```
