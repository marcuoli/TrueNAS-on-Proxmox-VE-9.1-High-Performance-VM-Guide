
# TrueNAS SCALE on Proxmox VE 9.1 – Performance & Stability Guide

## About

This repository is a practical, opinionated guide to running **TrueNAS SCALE as a VM on Proxmox VE 9.1**, optimized for **maximum performance**, **ZFS correctness**, and **long-term stability**.

It focuses on host + VM configuration that avoids common virtualization pitfalls for storage workloads, including:

- PCIe HBA passthrough (IT mode) and disk architecture do’s/don’ts
- CPU configuration (`cpu: host`), NUMA locality, and pinning guidance
- Memory settings (no ballooning), caching/snapshot caveats, and stability checks
- virtio networking recommendations and miscellaneous platform notes (e.g., TPM)

### Suggested GitHub “About” (repo description)

Best-practices guide for running TrueNAS SCALE as a high-performance VM on Proxmox VE 9.1 (HBA passthrough, ZFS correctness, stability).

Structured for **GitHub Wiki** usage.

---

## 1. Scope and Design Principles

- ZFS must control disks directly
- Prefer PCIe passthrough over virtual disks
- Avoid double caching
- NUMA-aware CPU and memory layout
- No CPU or RAM overcommit

---

## 2. Host Requirements (Proxmox VE 9.1)

### BIOS / Firmware
- UEFI enabled
- Secure Boot **disabled**
- VT-x / AMD-V enabled
- VT-d / AMD-Vi (IOMMU) enabled
- Disable firmware TPM if hardware TPM is installed

### Kernel Parameters
```bash
intel_iommu=on iommu=pt
# or
amd_iommu=on iommu=pt
```

Validate:
```bash
dmesg | grep -e IOMMU -e DMAR
ls /sys/kernel/iommu_groups/
```

---

## 3. Storage Architecture (Critical)

### Recommended
- PCIe HBA passthrough (LSI SAS2008/2308/3008, IT mode)

### Avoid
- ZFS-on-ZFS
- qcow2
- writeback cache
- snapshots of TrueNAS data disks

---

## 4. PCIe HBA Passthrough

### Identify HBA
```bash
lspci -nn | grep -i sas
```

### Bind to VFIO
```bash
echo "options vfio-pci ids=1000:0087" > /etc/modprobe.d/vfio.conf
update-initramfs -u
```

### Attach to VM
```bash
qm set 100 -hostpci0 0000:04:10.0,pcie=1
```

---

## 5. VM Core Configuration

### Machine
- Type: **q35**
- BIOS: **OVMF (UEFI)**
- Secure Boot: Disabled

### CPU (Corrected and Explicit)

**Use only:**
```
CPU type: host
Extra CPU flags: (empty)
```

**Do NOT manually add:**
- +aes
- +avx / +avx2
- +x2apic
- +invtsc

These features are automatically exposed by `cpu: host`. Forcing them may:
- Break live migration or restores
- Cause TSC instability
- Reduce portability

### Speculative Mitigations
Disable unless compliance requires otherwise:
- ssbd
- md-clear
- spec-ctrl
- ibrs / ibpb

### Memory
- Dedicated RAM only (no ballooning)
- Minimum: 16 GB
- Recommended: 32–128 GB
- Disable KSM on host for ZFS workloads

```bash
echo 0 > /sys/kernel/mm/ksm/run
```

---

## 6. Disk Layout

### Boot Disk
- virtio-scsi
- 16–32 GB
- cache=none

### Data Disks
- HBA passthrough only
- Never add data disks in Proxmox

---

## 7. Networking

- virtio-net
- vhost enabled
- multiqueue enabled
- MTU consistent with bridge

---

## 8. TPM

- TrueNAS SCALE does **not require TPM**
- Do not enable swtpm
- Hardware TPM should remain unused by the VM

---

## 9. Recommended `qm config` Example

```ini
agent: 1
bios: ovmf
boot: order=scsi0
cores: 8
cpu: host
machine: q35
memory: 65536
numa: 1
ostype: l26
scsihw: virtio-scsi-single
hostpci0: 0000:04:10.0,pcie=1
net0: virtio=BC:24:11:D6:5C:FB,bridge=vmbr0,queues=8
scsi0: local-lvm:vm-100-disk-0,cache=none,discard=ignore
```

---

## 10. Appendix A – CPU Pinning & NUMA Locality

### Identify NUMA Topology (Host)
```bash
lscpu
numactl --hardware
```

### Pin VM CPUs
Example: pin VM to NUMA node 0 (CPUs 0–7)
```bash
qm set 100 -cpulist 0-7
qm set 100 -numa 1
```

### Pin Memory (Optional)
```bash
qm set 100 -memory 65536 -numa 1
```

Best practice:
- VM CPUs and memory must belong to the **same NUMA node**
- Do not span NUMA nodes unless VM is very large

---

## 11. Appendix B – CPU Validation Checklist

### On Host
```bash
lscpu
```

Check:
- CPU(s)
- NUMA node(s)
- Core-to-node mapping

### On Proxmox (VM)
```bash
qm monitor 100
info cpus
```

Validate:
- vCPU count matches configuration
- CPU model = host
- No unexpected feature masking

### Inside TrueNAS SCALE
```bash
lscpu
```
Confirm:
- NUMA visibility
- Correct core count
- No ballooning

---

## 12. Validation Checklist

- `zpool status` shows physical disks
- HBA visible in SCALE
- No IOMMU faults in `dmesg`
- ARC size stable
- Swap usage minimal or zero

---

## 13. Anti-Patterns

- ZFS on virtual disks
- qcow2
- Ballooning memory
- CPU overcommit
- Sharing HBAs

---

**License:** MIT
