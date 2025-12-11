# Câu 5: Bare-metal vs Hosted Hypervisor

> **Chương:** 3 - Tiến trình và Ảo hóa  
> **Độ khó:** ⭐⭐⭐⭐ (Khó)  
> **Thời gian đọc:** ~25 phút

---

## 📋 Mục lục

- [Đề bài](#đề-bài)
- [Phần 1: Bare-metal Hypervisor (Type 1)](#phần-1-bare-metal-hypervisor-type-1)
- [Phần 2: Hosted Hypervisor (Type 2)](#phần-2-hosted-hypervisor-type-2)
- [Phần 3: Performance & Cost Analysis](#phần-3-performance--cost-analysis)
- [Phần 4: VM vs Container in Cloud](#phần-4-vm-vs-container-in-cloud)
- [Tóm tắt](#tóm-tắt)

---

## 📋 Đề bài

So sánh hai loại hypervisor:

**Bare-metal Hypervisor (Type 1)**
- Examples: VMware ESXi, Xen, KVM
- Chạy trực tiếp trên hardware
- Performance: 95-98% native

**Hosted Hypervisor (Type 2)**
- Examples: VirtualBox, VMware Workstation
- Chạy trên host OS
- Performance: 80-90% native

**Yêu cầu:**

1. So sánh về:
   - Performance overhead
   - Cost và complexity
   - Device driver reuse

2. Phân tích **VM vs Container** cho cloud computing:
   - IaaS (Infrastructure-as-a-Service)
   - PaaS (Platform-as-a-Service)
   - Trade-offs về performance, security, density

---

## 💡 Bài giải

### Phần 1: Bare-metal Hypervisor (Type 1)

#### A. Architecture
```
╔═══════════════════════════════════════════════╗
║      BARE-METAL HYPERVISOR (Type 1)           ║
║         "Runs directly on hardware"           ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  ┌────────┐  ┌────────┐  ┌────────┐          ║
║  │  VM 1  │  │  VM 2  │  │  VM 3  │          ║
║  │ Linux  │  │Windows │  │ Linux  │          ║
║  └───┬────┘  └───┬────┘  └───┬────┘          ║
║      │           │           │               ║
║  ════╪═══════════╪═══════════╪═══════════    ║
║      │  Virtual Hardware     │               ║
║  ┌───▼───────────▼───────────▼──────────┐    ║
║  │      HYPERVISOR (VMM)                │    ║
║  │                                      │    ║
║  │  ┌────────────────────────────────┐ │    ║
║  │  │  VM Management                 │ │    ║
║  │  │  - CPU scheduling              │ │    ║
║  │  │  - Memory management           │ │    ║
║  │  │  - I/O multiplexing            │ │    ║
║  │  └────────────────────────────────┘ │    ║
║  │                                      │    ║
║  │  ┌────────────────────────────────┐ │    ║
║  │  │  Device Drivers                │ │    ║
║  │  │  - Network                     │ │    ║
║  │  │  - Storage                     │ │    ║
║  │  │  - Must write own drivers! ⚠️  │ │    ║
║  │  └────────────────────────────────┘ │    ║
║  └──────────────┬───────────────────────┘    ║
║                 │ Direct access              ║
║  ═══════════════╪════════════════════════    ║
║                 │                            ║
║  ┌──────────────▼──────────────────────┐     ║
║  │      PHYSICAL HARDWARE              │     ║
║  │  - CPU                              │     ║
║  │  - Memory                           │     ║
║  │  - NIC, Disk, GPU, etc.             │     ║
║  └─────────────────────────────────────┘     ║
║                                               ║
╚═══════════════════════════════════════════════╝

Advantages:
✅ Best performance (95-98% native)
✅ Direct hardware access
✅ No host OS overhead
✅ Production-grade reliability

Disadvantages:
❌ Complex to develop/maintain
❌ Must write own device drivers
❌ Higher cost (enterprise licenses)
⚠️ Limited hardware compatibility
```

#### B. Examples

**1. VMware ESXi**
```
ESXi Hypervisor
═══════════════════════════════════════════

Architecture:
- Tiny footprint (~150 MB)
- Direct hardware control
- vSphere management

Performance:
- CPU: 98% native ✅
- Memory: 97% native ✅
- Network (vmxnet3): 95% native ✅
- Storage (paravirtual SCSI): 96% native ✅

Cost:
- Free edition: Limited features
- Standard: $995 per CPU
- Enterprise Plus: $3,595 per CPU ⚠️

Use case: Production datacenters ✅
```

**2. KVM (Linux)**
```
KVM (Kernel-based Virtual Machine)
═══════════════════════════════════════════

Architecture:
- Part of Linux kernel (since 2.6.20)
- Uses standard Linux features
- libvirt for management

Performance:
- CPU: 97% native ✅
- Memory: 98% native ✅ (EPT/NPT)
- Network (virtio): 90% native ✅
- Storage (virtio-blk): 93% native ✅

Cost:
- Free and open-source ✅✅

Use case:
- Public cloud (AWS, GCP use KVM)
- Private cloud (OpenStack)
- Cost-sensitive deployments ✅
```

**3. Xen**
```
Xen Hypervisor
═══════════════════════════════════════════

Architecture:
- Microkernel design (minimal hypervisor)
- Dom0 (privileged VM for management)
- DomU (guest VMs)

Performance:
- Paravirtualization: 95% native ✅
- HVM (full virtualization): 90% native

Cost:
- Free and open-source

Use case:
- AWS (early days, now KVM)
- Security-focused deployments
```

---

### Phần 2: Hosted Hypervisor (Type 2)

#### A. Architecture
```
╔═══════════════════════════════════════════════╗
║        HOSTED HYPERVISOR (Type 2)             ║
║        "Runs on top of host OS"               ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  ┌────────┐  ┌────────┐                      ║
║  │  VM 1  │  │  VM 2  │   (Guest VMs)        ║
║  │ Linux  │  │Windows │                      ║
║  └───┬────┘  └───┬────┘                      ║
║      │           │                           ║
║  ════╪═══════════╪═══════════════════════    ║
║      │  Virtual Hardware                     ║
║  ┌───▼───────────▼─────────────────────────┐ ║
║  │    HYPERVISOR APPLICATION               │ ║
║  │    (e.g., VirtualBox, VMware Workstation)│ ║
║  │                                         │ ║
║  │  - CPU emulation/virtualization        │ ║
║  │  - Memory management                   │ ║
║  │  - I/O handling via host OS ✅         │ ║
║  └──────────────┬──────────────────────────┘ ║
║                 │ System calls               ║
║  ═══════════════╪════════════════════════    ║
║                 │                            ║
║  ┌──────────────▼──────────────────────────┐ ║
║  │         HOST OPERATING SYSTEM           │ ║
║  │         (Windows / Linux / macOS)       │ ║
║  │                                         │ ║
║  │  - Device drivers (reused!) ✅          │ ║
║  │  - File system                          │ ║
║  │  - Networking stack                     │ ║
║  └──────────────┬──────────────────────────┘ ║
║                 │                            ║
║  ═══════════════╪════════════════════════    ║
║                 │                            ║
║  ┌──────────────▼──────────────────────────┐ ║
║  │       PHYSICAL HARDWARE                 │ ║
║  │  - CPU                                  │ ║
║  │  - Memory                               │ ║
║  │  - Devices                              │ ║
║  └─────────────────────────────────────────┘ ║
║                                               ║
╚═══════════════════════════════════════════════╝

Advantages:
✅ Easy to install (just an application)
✅ Reuses host OS device drivers ✅✅
✅ Lower cost (often free)
✅ Good for development/testing

Disadvantages:
❌ Lower performance (80-90% native)
❌ Host OS overhead (extra layer)
⚠️ Not suitable for production
⚠️ Limited scalability
```

#### B. Examples

**1. VirtualBox**
```
Oracle VirtualBox
═══════════════════════════════════════════

Architecture:
- Cross-platform (Windows, Linux, macOS)
- Free and open-source
- Extensions for USB, RDP, etc.

Performance:
- CPU: 85% native ⚠️
- Memory: 90% native ⚠️
- Network: 70% native (NAT) ⚠️
- Storage: 80% native ⚠️

Cost:
- Free ✅✅

Use case:
- Development
- Testing
- Personal use
- Education ✅
```

**2. VMware Workstation**
```
VMware Workstation Pro
═══════════════════════════════════════════

Architecture:
- Windows/Linux host
- Commercial product
- Advanced features (snapshots, cloning)

Performance:
- CPU: 90% native ✅
- Memory: 92% native ✅
- Network (vmxnet3): 85% native
- Storage: 88% native

Cost:
- $199 per license ⚠️

Use case:
- Professional development
- Testing environments
- Training ✅
```

**3. Parallels Desktop (macOS)**
```
Parallels Desktop
═══════════════════════════════════════════

Architecture:
- macOS host only
- Optimized for macOS/Windows integration
- Coherence mode (seamless)

Performance:
- CPU: 88% native
- Graphics: 80% native (DirectX support)
- Good for Windows on Mac ✅

Cost:
- $99/year subscription

Use case:
- Run Windows apps on macOS
- Development on macOS ✅
```

---

### Phần 3: Performance & Cost Analysis

#### A. Performance Comparison
```
╔═══════════════════════════════════════════════════════╗
║        TYPE 1 vs TYPE 2 PERFORMANCE                   ║
╠═══════════════════════════════════════════════════════╣
║                                                       ║
║  Metric            │ Type 1      │ Type 2            ║
║                    │ (Bare-metal)│ (Hosted)          ║
║ ═══════════════════╪═════════════╪══════════════════ ║
║                                                       ║
║  CPU performance   │  95-98% ✅  │  80-90% ⚠️        ║
║                                                       ║
║  Memory overhead   │   2-3%  ✅  │  10-15% ⚠️        ║
║                                                       ║
║  Network I/O       │  95% ✅     │  70-85% ⚠️        ║
║  (throughput)      │             │                   ║
║                                                       ║
║  Disk I/O          │  96% ✅     │  80-88% ⚠️        ║
║  (IOPS)            │             │                   ║
║                                                       ║
║  VM startup time   │  30 sec     │  45 sec           ║
║                                                       ║
║  Max VMs/server    │  50-100 ✅  │  5-10 ⚠️          ║
║  (64 GB RAM)       │             │                   ║
║                                                       ║
║  Latency overhead  │  <1ms ✅    │  2-5ms ⚠️         ║
║                                                       ║
╚═══════════════════════════════════════════════════════╝
```

**Benchmark: Web Server (Nginx)**
```python
# Simulated benchmark results

results = {
    "Native (bare metal)": {
        "throughput": 100000,  # req/s
        "latency": 5.0,        # ms
        "cpu_util": 95,        # %
    },
    "Type 1 (ESXi)": {
        "throughput": 97000,   # 97% ✅
        "latency": 5.2,        # +0.2ms
        "cpu_util": 93,
    },
    "Type 2 (VirtualBox)": {
        "throughput": 85000,   # 85% ⚠️
        "latency": 6.5,        # +1.5ms
        "cpu_util": 88,
    }
}

for system, metrics in results.items():
    print(f"{system}:")
    print(f"  Throughput: {metrics['throughput']:,} req/s")
    print(f"  Latency: {metrics['latency']} ms")
    print(f"  CPU: {metrics['cpu_util']}%")
    print()
```

**Output:**
```
Native (bare metal):
  Throughput: 100,000 req/s
  Latency: 5.0 ms
  CPU: 95%

Type 1 (ESXi):
  Throughput: 97,000 req/s ✅ (3% overhead)
  Latency: 5.2 ms
  CPU: 93%

Type 2 (VirtualBox):
  Throughput: 85,000 req/s ⚠️ (15% overhead)
  Latency: 6.5 ms
  CPU: 88%
```

#### B. Cost Analysis
```
╔═══════════════════════════════════════════════════════╗
║              TOTAL COST OF OWNERSHIP (TCO)            ║
║               (3-year period, 10-server datacenter)   ║
╠═══════════════════════════════════════════════════════╣
║                                                       ║
║  Cost Component      │ Type 1      │ Type 2          ║
║                      │ (VMware ESXi)│ (VirtualBox)   ║
║ ═════════════════════╪═════════════╪════════════════ ║
║                                                       ║
║  Software licenses   │  $35,950    │  $0 ✅          ║
║  (10 CPUs × $3,595)  │             │  (free)         ║
║                                                       ║
║  Support contract    │  $10,000/yr │  $0             ║
║  (3 years)           │  $30,000    │                 ║
║                                                       ║
║  Training            │  $5,000     │  $500           ║
║                                                       ║
║  Hardware            │  $100,000   │  $120,000 ⚠️    ║
║  (servers)           │             │  (need more)    ║
║                                                       ║
║  Power/cooling       │  $15,000    │  $18,000 ⚠️     ║
║  (3 years)           │             │  (lower density)║
║                                                       ║
║  Admin labor         │  $60,000    │  $45,000 ✅     ║
║  (3 years)           │             │  (simpler)      ║
║                                                       ║
║ ─────────────────────┼─────────────┼──────────────── ║
║                                                       ║
║  TOTAL (3 years)     │ $245,950 ⚠️ │ $183,500 ✅     ║
║                                                       ║
║  But consider:                                        ║
║  - Type 1: Production-ready ✅                       ║
║  - Type 2: Not recommended for production ❌         ║
║                                                       ║
╚═══════════════════════════════════════════════════════╝

Conclusion:
- Type 2 cheaper upfront ✅
- Type 1 better TCO for production ✅
- Type 2 good for dev/test only
```

#### C. Device Driver Reuse
```
╔═══════════════════════════════════════════════╗
║         DEVICE DRIVER STRATEGY                ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  Type 1 (Bare-metal):                         ║
║  ┌─────────────────────────────────────────┐ ║
║  │  Must write hypervisor-specific drivers │ ║
║  │                                         │ ║
║  │  Example: ESXi vmxnet3 driver           │ ║
║  │  - Written by VMware                    │ ║
║  │  - Optimized for virtualization ✅      │ ║
║  │  - But: Limited device support ⚠️       │ ║
║  │    (only certified hardware)            │ ║
║  └─────────────────────────────────────────┘ ║
║                                               ║
║  Pros:                                        ║
║  ✅ Optimized (paravirtual drivers)          ║
║  ✅ Best performance                         ║
║                                               ║
║  Cons:                                        ║
║  ❌ Must develop/maintain drivers            ║
║  ❌ Limited hardware compatibility           ║
║                                               ║
║ ────────────────────────────────────────────  ║
║                                               ║
║  Type 2 (Hosted):                             ║
║  ┌─────────────────────────────────────────┐ ║
║  │  Reuses host OS drivers! ✅✅            │ ║
║  │                                         │ ║
║  │  Example: VirtualBox on Windows          │ ║
║  │  - Uses Windows NIC driver              │ ║
║  │  - Uses Windows disk driver             │ ║
║  │  - Works with ANY hardware! ✅          │ ║
║  └─────────────────────────────────────────┘ ║
║                                               ║
║  Pros:                                        ║
║  ✅ Universal hardware support               ║
║  ✅ No driver development needed             ║
║  ✅ Automatic driver updates                 ║
║                                               ║
║  Cons:                                        ║
║  ⚠️ Not optimized (generic drivers)          ║
║  ⚠️ Extra layer (host OS overhead)           ║
║                                               ║
╚═══════════════════════════════════════════════╝
```

---

### Phần 4: VM vs Container in Cloud

#### A. Container Architecture
```
╔═══════════════════════════════════════════════╗
║         CONTAINER vs VM COMPARISON            ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  VIRTUAL MACHINE:                             ║
║  ┌────────────────────────────────────────┐  ║
║  │  App A  │  App B  │  App C             │  ║
║  ├─────────┼─────────┼────────────────────┤  ║
║  │ Bins/Libs│Bins/Libs│Bins/Libs          │  ║
║  ├─────────┼─────────┼────────────────────┤  ║
║  │Guest OS │Guest OS │Guest OS            │  ║
║  │(1.5 GB) │(1.5 GB) │(1.5 GB) ⚠️         │  ║
║  └────────────────────────────────────────┘  ║
║         │         │         │                ║
║  ┌──────▼─────────▼─────────▼────────────┐  ║
║  │         HYPERVISOR                     │  ║
║  └────────────────────────────────────────┘  ║
║                                               ║
║  Total size: 3 × 1.5 GB = 4.5 GB ⚠️          ║
║  Startup time: 30-60 seconds ⚠️              ║
║                                               ║
║ ────────────────────────────────────────────  ║
║                                               ║
║  CONTAINER:                                   ║
║  ┌────────────────────────────────────────┐  ║
║  │  App A  │  App B  │  App C             │  ║
║  │(100 MB) │(100 MB) │(100 MB) ✅         │  ║
║  ├─────────┼─────────┼────────────────────┤  ║
║  │     Bins/Libs (shared)                 │  ║
║  └─────────┬─────────┬────────────────────┘  ║
║            │         │                       ║
║  ┌─────────▼─────────▼────────────────────┐  ║
║  │     CONTAINER RUNTIME (Docker)         │  ║
║  └────────────────────────────────────────┘  ║
║  ┌────────────────────────────────────────┐  ║
║  │       HOST OS (shared kernel) ✅       │  ║
║  └────────────────────────────────────────┘  ║
║                                               ║
║  Total size: 300 MB ✅✅ (15× smaller!)      ║
║  Startup time: <1 second ✅✅                ║
║                                               ║
╚═══════════════════════════════════════════════╝
```

#### B. Cloud Use Cases

**1. IaaS (Infrastructure-as-a-Service)** 
```
╔═══════════════════════════════════════════════╗
║      IaaS: VM-BASED (ONLY OPTION) ✅          ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  Requirements:                                ║
║  ├─ Customer needs full OS control ✅         ║
║  ├─ Different OS (Windows, Linux, BSD)        ║
║  ├─ Kernel customization required             ║
║  ├─ Strong isolation (multi-tenant)           ║
║  └─ Legacy application support                ║
║                                               ║
║  Why VM (not container):                      ║
║  ✅ Full OS control                           ║
║  ✅ Hardware-level isolation                  ║
║  ✅ Any OS (Windows, Linux, custom)           ║
║  ✅ Kernel modules (load eBPF, custom)        ║
║                                               ║
║  Container limitations for IaaS:              ║
║  ❌ Shared kernel (cannot customize)          ║
║  ❌ Same OS only (Linux containers need Linux)║
║  ❌ Weaker isolation                          ║
║                                               ║
║  Examples:                                    ║
║  - AWS EC2 ✅                                 ║
║  - Google Compute Engine ✅                   ║
║  - Azure Virtual Machines ✅                  ║
║                                               ║
╚═══════════════════════════════════════════════╝
```

**2. PaaS (Platform-as-a-Service)**