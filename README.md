# um790pro_7900xtx_setup

(hopefully) Exhaustive description and docs to setup a Radeon RX 7900 XTX 24G eGPU with a Minisforum UM790 Pro, via Minisforum DEG1 Oculink dock

# AMD RX 7900 XTX eGPU via OcuLink on Minisforum UM790 Pro + Proxmox VE

A complete guide for connecting a Minisforum DEG1 eGPU dock with an AMD Radeon RX 7900 XTX to a Minisforum UM790 Pro mini PC running Proxmox VE, and passing the GPU through to a Linux virtual machine for AI/compute workloads.

It should be clear that I can only vouch for the hardware listed and described here, as it's the exact one I bought and set up and found to be working.

---

## Hardware Used

"Hardware used" shall be read as "hardware validated to work" - this includes the config of a Proxmox host (community edition, in my case), PCIe-Passthrough from the host to a VM and setup of a genAI stack inside the VM.

### Host Computer
- **Minisforum UM790 Pro** mini PC
- **CPU:** AMD Ryzen 9 7940HS (8 cores / 16 threads, Zen 4 "Phoenix")
- **RAM:** 64 GB DDR5-5600
- **Internal SSD:** Kingston 1 TB NVMe (in M.2 slot J91)
- **OS:** Proxmox VE 9.1.6 on Debian 13 (trixie), kernel 6.17.13-2-pve
- **BIOS:** AMI 1.09 (November 2023) — no newer version available

The UM790 Pro has **two internal M.2 slots**:
- `J91` — occupied by the internal NVMe SSD
- `U93` — used for the OcuLink adapter in this setup

There is **no external OcuLink port** on the UM790 Pro. The connection is made
via an M.2 adapter card installed inside the machine in the U93 slot.

### M.2-to-OcuLink Adapter
- **RIITOP M.2 NVMe PCIe 4.0 x4 to OCuLink SFF-8611 4i Host Adapter**
- Installed in the **U93 M.2 slot** inside the UM790 Pro
- Converts the PCIe x4 M.2 slot into an external SFF-8611 OcuLink port
- Connected to the DEG1 dock via the included OcuLink SFF-8611 cable
- Choice supported and triggered be a post found on Reddit (see "References")

### eGPU Dock
- **Minisforum DEG1** (OcuLink 4i variant)
- Connects to the host via an **OcuLink SFF-8611 4i cable** (PCIe 4.0 x4, up
  to 64 Gbps, cable used is the one distributed by Minisforum with the DEG1 by default)
- Accepts a standard PCIe x16 graphics card and an ATX or SFX power supply
- Has a **ForcePowerOn button with LED** and **three hidden DIP switches**
  under the bottom backplate (see critical section below)
- Does **not** include a power supply — you must provide one separately
- Choice supported and triggered be a post found on Reddit (see "References")

### External GPU
- **AMD Radeon RX 7900 XTX**, 24 GB GDDR6
- The card contains an internal PCIe switch (Navi 10 XL) that bridges the x4
  OcuLink uplink to a full x16 slot connection on the card itself

### PSU
- be quiet! Pure Power 11 700W (80+ Gold)

---

## How the Connection Works

```
UM790 Pro (U93 M.2 slot)
        │
        │  PCIe 4.0 x4 (M.2 edge connector)
        │
  RIITOP M.2 adapter
        │
        │  OcuLink SFF-8611 cable (PCIe 4.0 x4, 64 Gbps)
        │
   Minisforum DEG1 dock
        │
        │  PCIe x16 slot (electrically x4 from OcuLink)
        │
  RX 7900 XTX
  (internal PCIe switch on card negotiates x16 to x4)
```

The bandwidth limitation is the OcuLink cable: PCIe 4.0 x4 = ~64 Gbps. The
GPU card itself has an internal PCIe switch that bridges this to a full x16
slot, but the actual data throughput is still capped at x4. For GPU compute
workloads (AI inference, rendering) this is negligible. For gaming there is a
small performance penalty compared to a native x16 slot.

---

## PCI Device IDs (What the Numbers Mean)

After successful boot, these devices appear in `lspci` (note: yours may vary, so take care to check and validate):

| PCI Address | Device ID | What it is | Location |
|---|---|---|---|
| `01:00.0` | `1002:1478` | Navi 10 XL PCIe Switch — Upstream Port | **Inside the RX 7900 XTX card** |
| `02:00.0` | `1002:1479` | Navi 10 XL PCIe Switch — Downstream Port | **Inside the RX 7900 XTX card** |
| `03:00.0` | `1002:744c` | **RX 7900 XTX GPU** (Navi 31) | External — in DEG1 dock |
| `03:00.1` | `1002:ab30` | RX 7900 XTX HDMI/DisplayPort Audio | External — in DEG1 dock |
| `04:00.0` | `10ec:8125` | Realtek 2.5 GbE network card | Internal — on motherboard |
| `05:00.0` | `2646:501b` | Kingston NVMe SSD | Internal — M.2 slot J91 |
| `c6:00.0` | `1002:15bf` | AMD Radeon 780M **iGPU** (Phoenix1) | Internal — inside the CPU |

The PCIe switch inside the GPU (`1002:1478` / `1002:1479`) is a normal part of
the RX 7900 XTX card design. It acts as a bridge that allows the card to
present itself as x16 even when connected via x4 OcuLink.

The **iGPU** (`1002:15bf`) is the graphics processor built into the Ryzen CPU.
It is **not** passed through to the VM — it stays with the Proxmox host for
console/management access.

---

## A (Potential) Critical Problem: DEG1 Hidden DIP Switches

The Minisforum DEG1 dock has **three undocumented DIP switches** hidden under
the bottom backplate (requires removing screws to access). These switches are
**not mentioned in the product manual** and were discovered by users opening
the dock.

> ⚠️ **Warning:** The switches may be fragile (as reported by some users on ServeTheHome forum). The original discoverer broke one
> trying to flip it. Use a fine-tipped tool and apply minimal force.

The switches control signaling behaviour of the dock's PCIe redriver chip
(P13EQX16612). Their labels and meanings:

### Switch 1: `DEBUG`
- **Positions:** A / B
- **Factory default:** A
- **What it does:** Unknown diagnostic/debug mode. Leave at A unless
  troubleshooting.

### Switch 2: `FLLOW START` (Follow Start)
- **Positions:** ON / OFF
- **Factory default:** OFF
- **What it does:** Controls whether the dock's auto power-on works only with
  native Minisforum OcuLink ports (the proprietary cable signal), or also with
  generic M.2 OcuLink adapters.
  - `OFF` — follow-power only works with native Minisforum OcuLink ports and
    the original Minisforum cable. On non-Minisforum hosts (or via M.2
    adapters), the dock does not signal PCIe presence to the host during POST,
    so the GPU is never detected.
  - `ON` — enables follow-power compatibility with generic M.2-to-OcuLink
    adapters. **Required for the UM790 Pro + RIITOP adapter setup.**

### Switch 3: `TGX`
- **Positions:** ON / OFF
- **Factory default:** ON
- **What it does:** Enables Lenovo TGX (Think GPU Xpansion) OcuLink signaling
  variant. Lenovo ThinkBook laptops use a slightly different OcuLink signaling
  protocol. For non-Lenovo hosts this can interfere with link training.
  - `ON` — Lenovo TGX signaling enabled (factory default, works with
    Lenovo ThinkBooks)
  - `OFF` — Standard OcuLink signaling. **Required for UM790 Pro.**

### Working Switch Configuration for UM790 Pro + RIITOP Adapter

| Switch | Setting | Notes |
|---|---|---|
| `DEBUG` | **A** | Leave at factory default |
| `FLLOW START` | **ON** | Changed from factory OFF |
| `TGX` | **OFF** | Changed from factory ON |

This combination was found by trial and error. The factory defaults (`FLLOW START=ON`, `TGX=ON`) are (potentially) optimised for native Minisforum OcuLink ports on newer Minisforum machines. For a generic M.2 adapter on a UM790 Pro, both switches need to be changed. I have no real clue on the meaning and effect of the TGX switch, but switching it to `OFF` in the end did the trick.

**Symptom of wrong switch config:** The PCIe bridge for the U93 slot
(`00:02.5` / `00:02.6`) does not appear in `lspci` at all. The BIOS reports
U93 as `Available` in DMI instead of `In Use`. The GPU fans **do not spin up at all*
when the DEG1 power button is pressed (no PCIe presence signal reaches the
host).

**Sign of correct config:** When the DEG1 power button is pressed (before
booting the host), the GPU fans spin up briefly. On host boot, the GPU appears
in `lspci` and U93 shows `In Use` in `dmidecode -t 9`.

---

## Required Boot Sequence

OcuLink does **not** support hot-plugging. The GPU must be powered and
presenting a PCIe link **before** the host CPU runs POST (the hardware
self-test at startup). If the GPU is not present during POST, the BIOS will not
allocate PCIe resources for it and it will never appear, even if powered on
later.

**Every boot must follow this order:**

1. Ensure the host (UM790 Pro) is fully **powered off**
2. Ensure the OcuLink cable is connected at both ends (click to lock at both
   the RIITOP adapter and the DEG1 dock)
3. Switch on the PSU (rear rocker switch)
4. Press the **DEG1 power button** — the GPU fans will spin at full speed
   (normal, the GPU has no driver yet to control them)
5. Wait approximately **10–30 seconds** for the fans to settle
6. Power on the **UM790 Pro**

To power off: shut down the host OS first, then power off the DEG1.

---

## Proxmox VE Configuration

### Goal

The RX 7900 XTX is passed through entirely to a Linux VM (genaivm, VM 103)
using **VFIO** — a Linux kernel subsystem that lets virtual machines have
direct, exclusive access to a physical PCIe device. The host Proxmox OS does
not use the GPU at all; only the VM does.

### Step 1: Kernel Boot Parameters

Verify these are in `/etc/default/grub` on the
`GRUB_CMDLINE_LINUX_DEFAULT` line:

```
amd_iommu=on    ← enables AMD's IOMMU (required for PCIe passthrough)
iommu=pt        ← passthrough mode (better performance, less overhead)
pcie_aspm=force ← apparently this **can** cause stability problems, but I found none.
```

Apply with `update-grub` if changed, then reboot. For the `pcie_aspm=force` switch: I found it to neither trigger stability issues, but also no power saving at all in comparison to not having this in the GRUB commandline.

### Step 2: VFIO Module Configuration

**File: `/etc/modprobe.d/vfio.conf`**

```
#options vfio-pci ids=1002:744c,1002:ab30
softdep amdgpu pre: vfio-pci
softdep snd_hda_intel pre: vfio-pci
```

**What this means, line by line:**

- `#options vfio-pci ids=1002:744c,1002:ab30`
  — This line is intentionally **commented out** for this setup.

  When active, `vfio-pci` would claim both GPU (`1002:744c`) and audio
  (`1002:ab30`) at early boot via static ID binding. For this hardware
  combination (RX 7900 XTX over OcuLink/M.2), this causes a fatal QEMU crash
  on every VM start: `vfio-pci` claims the GPU before `amdgpu` has initialized
  its PCIe config space, leaving the INTPIN register in an invalid state. QEMU
  then crashes with:
  ```
  kvm: ../hw/pci/pci.c: Assertion `0 <= irq_num && irq_num < PCI_NUM_PINS' failed.
  ```
  With the line commented out, Proxmox handles the `amdgpu` → `vfio-pci`
  rebind dynamically at VM start time, after the GPU config space is fully
  initialized. The `softdep` lines below ensure `vfio-pci` is loaded early
  enough for Proxmox to use it.

  > ⚠️ **Important:** Every time `update-initramfs` runs (e.g. after a kernel
  > upgrade), the current `/etc/modprobe.d/vfio.conf` is baked into the new
  > initramfs. Make sure the `ids=` line remains commented out, or the VM will
  > stop starting after the next kernel update.

- `softdep amdgpu pre: vfio-pci`
  — Tells the kernel: "before loading the `amdgpu` graphics driver, load
  `vfio-pci` first." This ensures `vfio-pci` is available when Proxmox
  performs the dynamic rebind at VM start time.

- `softdep snd_hda_intel pre: vfio-pci`
  — Same concept for the audio driver (`snd_hda_intel`) which would otherwise
  claim the `1002:ab30` audio device.

### Step 3: VFIO Kernel Modules

**File: `/etc/modules`**

```
vfio
vfio_iommu_type1
vfio_pci
```

These three modules must be loaded at boot (in `/etc/modules`) rather than
on-demand, so they are available early enough in the boot process to claim the
GPU before any graphics driver tries to.

- `vfio` — core VFIO framework
- `vfio_iommu_type1` — IOMMU integration (uses the AMD IOMMU to isolate the
  device for the VM)
- `vfio_pci` — the driver that claims PCIe devices for VFIO passthrough

### Step 4: Rebuild initramfs

After editing the above files, run:

```bash
update-initramfs -u -k all
```

This rebuilds the initial RAM disk (the minimal OS image loaded at very early
boot) to include the VFIO modules and configuration. Without this step the
changes take no effect until the next Proxmox package update.

### Step 5: VM Configuration

**File: `/etc/pve/qemu-server/103.conf`** (relevant lines)

```
bios: ovmf                               # UEFI firmware (required for GPU passthrough)
machine: q35                             # PCIe-capable virtual machine chipset
hostpci0: 0000:03:00.0,pcie=1,rombar=0  # RX 7900 XTX GPU
hostpci1: 0000:03:00.1                  # RX 7900 XTX Audio (mandatory — see note)
```

**What this means:**

- `hostpci0: 0000:03:00.0,pcie=1,rombar=0`
  — Pass the GPU (`03:00.0`) directly to the VM.
  - `pcie=1` — expose it as a PCIe device (required for modern GPUs)
  - `rombar=0` — disables the option ROM BAR. Use this instead of `x-vga=1`
    for headless/compute setups with no monitor attached to the eGPU.
    `x-vga=1` enables a VGA compatibility shim that routes legacy IRQs through
    a QEMU path which reads the GPU's INTPIN register — unreliable for this GPU
    over OcuLink. Since the VM uses OVMF/UEFI and no display is connected,
    `x-vga=1` is not needed.

- `hostpci1: 0000:03:00.1`
  — Pass the audio device (`03:00.1`) to the VM alongside the GPU.

  > ⚠️ **This line is mandatory and must not be removed.** The RX 7900 XTX
  > GPU die actively power-manages the `03:00.1` audio chip and sends PCIe
  > transactions to its registers during normal GPU operation. If `hostpci1` is
  > omitted, those transactions hit the host `snd_hda_intel` driver instead,
  > generating cascading PCIe AER errors that crash the guest. Both devices
  > must always be passed through together.

- `bios: ovmf` — UEFI firmware. GPU passthrough requires UEFI, not legacy
  BIOS, because modern AMD GPUs need UEFI GOP (Graphics Output Protocol) to
  initialize.

- `machine: q35` — the Q35 chipset emulates a PCIe bus inside the VM, which
  is required for `hostpci` passthrough to work correctly.

### Step 6: Post-Stop Hook Script

When the guest shuts down, the RX 7900 XTX GPU puts its HDMI/DP audio
companion (`03:00.1`) into D3hot as part of normal GPU power management. If
QEMU exits while `03:00.1` is in D3hot, the next `qm start` will hand the GPU
to the guest while it is still in D3, and the guest will fail to enumerate it
(the GPU will be invisible inside the VM).

The Proxmox hook script handles this by running a `post-stop` phase that
detects the stuck D3hot state and forces a PCIe remove + rescan to return the
device to D0 before the next start.

**File: `/var/lib/vz/snippets/cpu-pinning.pl`**

```perl
#!/usr/bin/perl

use strict;
use warnings;

my $vmid = shift;
my $phase = shift;

if ($phase eq 'post-start') {
    # Suche die PID der VM
    my $pid = `cat /var/run/qemu-server/$vmid.pid`;
    chomp($pid);

    if ($pid) {
        # Hier definieren wir die Kerne 1 bis 7 (deine physischen Kerne)
        my $mask = "1,2,3,4,5,6,7";
        system("taskset -cp $mask $pid");
        print "CPU Pinning: VM $vmid an Kerne $mask gebunden (PID $pid).\n";
    } else {
        die "CPU Pinning Fehlgeschlagen: PID für VM $vmid nicht gefunden.\n";
    }

} elsif ($phase eq 'post-stop') {
    if ($vmid == 103) {
        my $audio = '0000:03:00.1';
        my $state_file = "/sys/bus/pci/devices/$audio/power_state";

        if (-e $state_file) {
            open(my $fh, '<', $state_file) or die "Cannot read power state: $!";
            my $state = <$fh>; chomp $state;
            close($fh);

            if ($state ne 'D0') {
                print "post-stop: $audio is in $state, forcing re-enumeration...\n";
                system("echo 1 > /sys/bus/pci/devices/$audio/remove");
                sleep(2);
                system("echo 1 > /sys/bus/pci/devices/0000:02:00.0/rescan");
                sleep(2);
                print "post-stop: re-enumeration complete.\n";
            } else {
                print "post-stop: $audio already at D0, no action needed.\n";
            }
        }
    }
}

exit 0;
```

The `post-stop` block is scoped to VM 103 because the PCIe remove/rescan is
specific to the RX 7900 XTX audio companion. The `post-start` CPU pinning
block intentionally has no VM ID check — it applies to any VM that uses this
hook script.

This script is referenced in the VM config as:
```
hookscript: local:snippets/cpu-pinning.pl
```

### Step 7: Verify After Reboot

After rebooting the host:

```bash
# Confirm vfio-pci has claimed the GPU (not amdgpu)
lspci -k | grep -A2 "744c"
# Expected output includes: "Kernel driver in use: vfio-pci"

# Confirm U93 slot is active
dmidecode -t 9 | grep -A4 "U93"
# Expected: "Current Usage: In Use"
```

Then start VM 103 and install the AMD ROCm or AMDGPU-PRO drivers inside the
guest OS.

---

## IOMMU Groups

IOMMU groups define which devices must be passed through together (they share
a memory isolation boundary). For this setup:

| IOMMU Group | Device | Description |
|---|---|---|
| 15 | `03:00.0` `[1002:744c]` | RX 7900 XTX GPU |
| 16 | `03:00.1` `[1002:ab30]` | RX 7900 XTX Audio |
| 14 | `c6:00.0` `[1002:15bf]` | iGPU (stays with host) |

The GPU and its audio are in separate IOMMU groups. Despite this, both devices
**must** be passed through together — see the note in Step 5. The separate
groups are useful for isolation purposes but do not mean the audio device can
be omitted from the VM config.

---

## Troubleshooting

### GPU not visible in `lspci` after boot

1. Check the DEG1 DIP switches — `FLLOW START` must be **ON**, `TGX` must be
   **OFF** for the UM790 Pro + RIITOP adapter.
2. Verify boot order: DEG1 must be powered on **before** the UM790 Pro.
3. Check both OcuLink cable connectors are locked (push until click).
4. Check RIITOP adapter is fully seated in the U93 M.2 slot (flat, screw
   tightened).
5. Run `dmidecode -t 9 | grep -A4 "U93"` — if it shows `Available` instead
   of `In Use`, the BIOS did not detect the adapter during POST.

### GPU visible but VFIO not claiming it (amdgpu loads instead)

1. Verify `/etc/modprobe.d/vfio.conf` has the `softdep` lines present.
2. Verify VFIO modules are in `/etc/modules` (not commented out).
3. Re-run `update-initramfs -u -k all`.
4. Check with `lspci -k | grep -A2 "744c"` — should show `vfio-pci`.

### VM fails to start — QEMU assertion crash (pci_irq_handler)

Symptom in Proxmox task log:
```
kvm: ../hw/pci/pci.c: Assertion `0 <= irq_num && irq_num < PCI_NUM_PINS' failed.
start failed: QEMU exited with code 1
```

Cause: The `ids=` line in `/etc/modprobe.d/vfio.conf` is active (not commented
out), causing early static binding before the GPU config space is initialized.

Fix:
1. Comment out the `options vfio-pci ids=...` line in `/etc/modprobe.d/vfio.conf`
2. Run `update-initramfs -u -k all`
3. Reboot

### VM starts but GPU not visible inside guest

Symptom: `lspci` inside the guest shows no AMD GPU, only the emulated VGA
(`1234:1111`).

Cause: The VM was started while `03:00.0` or `03:00.1` was stuck in D3hot/D3cold
from a previous unclean shutdown. QEMU reported `TASK OK` but handed a
powered-down device to the guest.

Fix: A full host reboot is required to power-cycle the OcuLink link and bring
the GPU back to D0. The post-stop hook (Step 6) prevents this from recurring
after normal shutdowns.

### VM fails to start after adding hostpci lines

1. Ensure the VM uses `machine: q35` and `bios: ovmf`.
2. Confirm both `hostpci0` and `hostpci1` are present in the VM config.
3. Confirm the GPU is not in use by the host (`lspci -k` shows `vfio-pci`).
4. Check Proxmox task log for the specific error.

### VM shuts down but QEMU process stays alive / internal-error state

The QEMU process uses `-no-shutdown` (Proxmox default), meaning it stays alive
after the guest powers off waiting for Proxmox to issue a final `quit` command.
If the guest agent is not responding or the ACPI shutdown times out, Proxmox
may leave the process in `internal-error` state.

- **Immediate fix:** `qm stop 103` — forces QEMU to exit cleanly
- **Prevention:** Ensure `qemu-guest-agent` is installed and running inside the
  guest VM. This allows Proxmox to use a clean guest-initiated shutdown path.

---

## Reference

- STH forum thread (switch discovery): https://forums.servethehome.com/index.php?threads/minisforum-deg1-hidden-switches-and-other-observations.47589/
- STH DEG1 review: https://www.servethehome.com/minisforum-deg1-oculink-egpu-dock-quick-look-amd-nvidia/
- Reddit confirmation (identical hardware): https://old.reddit.com/r/eGPU/comments/1j0yzeo/
- Proxmox PCI passthrough documentation: https://pve.proxmox.com/wiki/PCI_Passthrough
