# KVM
Madlab Nexus KVM configs

# Complete Arch Linux VFIO Gaming & Hackintosh Setup Guide
## Lenovo P16v - Multi-VM Gaming & macOS Setup

**Hardware:**
- Lenovo P16v
- Intel Core i9 (13th/14th gen)
- Intel Arc integrated GPU
- NVIDIA RTX 3000 (Ada) dedicated GPU
- Multiple NVMe drives

**Goal:** Create a versatile Arch Linux host capable of running:
1. Windows VM with Vanguard anti-cheat support (GPU passthrough) - **Undetectable**
2. macOS Sequoia VM with Intel Arc GPU passthrough
3. Optional KDE Plasma for bare metal Linux desktop

**What This Guide Covers:**
- ✅ Complete Arch Linux installation from scratch
- ✅ BIOS configuration for virtualization
- ✅ KVM/QEMU/Libvirt setup
- ✅ Hardware passthrough (GPU, USB, NVMe, Audio)
- ✅ Windows VM with maximum performance tuning
- ✅ Complete VM stealth/undetectability configuration
- ✅ Anti-cheat testing procedures
- ✅ macOS VM setup
- ✅ Troubleshooting guide

---

## Table of Contents

1. [Pre-Installation Prerequisites](#1-pre-installation-prerequisites)
2. [BIOS Configuration](#2-bios-configuration)
3. [Arch Linux Base Installation](#3-arch-linux-base-installation)
4. [Bootloader & IOMMU Configuration](#4-bootloader--iommu-configuration)
5. [KDE Plasma Installation (Optional)](#5-kde-plasma-installation-optional)
6. [QEMU/KVM/Libvirt Setup](#6-qemukvm-libvirt-setup)
7. [Hardware Passthrough Configuration](#7-hardware-passthrough-configuration)
8. [Windows VM - Maximum Performance & Undetectable](#8-windows-vm---maximum-performance--undetectable)
   - 8.1 [Advanced Kernel Configuration](#81-advanced-kernel-configuration)
   - 8.2 [Host SMBIOS Information Collection](#82-host-smbios-information-collection)
   - 8.3 [Huge Pages Configuration](#83-huge-pages-configuration)
   - 8.4 [ACPI Table Patching](#84-acpi-table-patching)
   - 8.5 [Complete VM XML Configuration](#85-complete-vm-xml-configuration)
   - 8.6 [Windows Guest VM Hiding](#86-windows-guest-vm-hiding)
   - 8.7 [Performance Tuning in Windows](#87-performance-tuning-in-windows)
   - 8.8 [CPU Isolation & Automation](#88-cpu-isolation--automation)
   - 8.9 [Additional Performance Features](#89-additional-performance-features)
9. [Testing & Verification](#9-testing--verification)
10. [macOS Sequoia VM - Intel Arc Passthrough](#10-macos-sequoia-vm---intel-arc-passthrough)
11. [Troubleshooting](#11-troubleshooting)
12. [Useful Commands Reference](#12-useful-commands-reference)

---

## 1. Pre-Installation Prerequisites

### Required Items
- Arch Linux ISO (latest)
- USB drive (8GB+)
- Ethernet connection (WiFi setup comes later)
- Windows already installed on separate NVMe drive
- Backup of important data

### Important Notes
- **Vanguard Anti-Cheat**: This guide configures the VM for maximum stealth. For game development testing purposes.
- **Laptop Considerations**: Hardware passthrough on laptops is more challenging than desktops due to shared resources.
- **Intel Arc macOS**: Intel Arc is not natively supported by macOS. We'll use experimental approaches.

---

## 2. BIOS Configuration

Access BIOS (usually F1 or F2 during boot) and configure:

### Essential Settings

```
Virtualization Technology (VT-x): ENABLED
VT-d (Intel Virtualization for Directed I/O): ENABLED
IOMMU: ENABLED

For Vanguard Support:
Secure Boot: ENABLED (we'll sign our bootloader)
TPM 2.0: ENABLED

Graphics Settings:
Primary Display: Auto or iGPU (allows both GPUs to be active)
```

### Optional but Recommended
```
ASPM (Active State Power Management): DISABLED (can cause GPU passthrough issues)
Resize BAR: ENABLED (better GPU performance)
Above 4G Decoding: ENABLED (required for large GPU BAR)
```

Save and exit BIOS.

---

## 3. Arch Linux Base Installation

Boot from Arch ISO.

### Set Keyboard Layout (if needed)
```bash
loadkeys br-abnt2  # Brazilian keyboard, adjust as needed
```

### Verify Boot Mode (should show UEFI)
```bash
cat /sys/firmware/efi/fw_platform_size
# Should return 64
```

### Connect to Internet
```bash
# For Ethernet (usually automatic)
ping -c 3 archlinux.org

# For WiFi (if needed now)
iwctl
# In iwctl prompt:
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect "YOUR_SSID"
exit
```

### Update System Clock
```bash
timedatectl set-ntp true
timedatectl status
```

### Partition Disks

Assuming your system has:
- `/dev/nvme0n1` - Primary drive for Arch Linux
- `/dev/nvme1n1` - Windows installation (DO NOT PARTITION)

```bash
# List all disks
lsblk

# Partition the Arch Linux disk
fdisk /dev/nvme0n1
```

**Partition scheme for UEFI:**

```
/dev/nvme0n1p1 - 1GB   - EFI System (type: EFI System)
/dev/nvme0n1p2 - 32GB  - Swap (type: Linux swap)
/dev/nvme0n1p3 - Rest  - Root (type: Linux filesystem)
```

**fdisk commands:**
```
g          # Create GPT partition table
n          # New partition
1          # Partition 1
[Enter]    # Default start
+1G        # 1GB size
t          # Change type
1          # EFI System

n          # Partition 2
2
[Enter]
+32G
t
2
19         # Linux swap

n          # Partition 3
3
[Enter]
[Enter]    # Use remaining space
# Type is already Linux filesystem

w          # Write changes
```

### Format Partitions

```bash
# Format EFI partition
mkfs.fat -F32 /dev/nvme0n1p1

# Setup swap
mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2

# Format root partition (ext4)
mkfs.ext4 /dev/nvme0n1p3
```

### Mount Filesystems

```bash
# Mount root
mount /dev/nvme0n1p3 /mnt

# Create and mount EFI directory
mkdir -p /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

### Install Base System

```bash
# Install essential packages
pacstrap -K /mnt base linux linux-firmware intel-ucode nano vim sudo networkmanager
```

### Generate fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab

# Verify it looks correct
cat /mnt/etc/fstab
```

### Chroot into New System

```bash
arch-chroot /mnt
```

---

## 4. Bootloader & IOMMU Configuration

### Set Timezone

```bash
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime  # Adjust for your timezone
hwclock --systohc
```

### Localization

```bash
# Edit locale.gen
nano /etc/locale.gen

# Uncomment your locale(s), e.g.:
# en_US.UTF-8 UTF-8
# pt_BR.UTF-8 UTF-8

# Generate locales
locale-gen

# Set system locale
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# If you changed keyboard layout
echo "KEYMAP=br-abnt2" > /etc/vconsole.conf
```

### Network Configuration

```bash
# Set hostname
echo "archlinux-vfio" > /etc/hostname

# Configure hosts
nano /etc/hosts
```

Add:
```
127.0.0.1    localhost
::1          localhost
127.0.1.1    archlinux-vfio.localdomain archlinux-vfio
```

### Set Root Password

```bash
passwd
```

### Install Bootloader (systemd-boot)

```bash
bootctl install
```

### Finding Your GPU IDs

**CRITICAL STEP - Do this now:**

```bash
# Find your NVIDIA GPU and Audio device IDs
lspci -nn | grep NVIDIA

# Example output:
# 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation [10de:25e0] (rev a1)
# 01:00.1 Audio device [0403]: NVIDIA Corporation [10de:2291] (rev a1)

# Note down the IDs: 10de:25e0,10de:2291 (yours will be different)
```

Write down your IDs - you'll need them in the next step.

### Configure Bootloader Entry

```bash
nano /boot/loader/entries/arch.conf
```

**Replace XXXX and YYYY with YOUR actual GPU IDs from above:**

```
title   Arch Linux VFIO
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root=/dev/nvme0n1p3 rw intel_iommu=on iommu=pt vfio-pci.ids=10de:XXXX,10de:YYYY video=efifb:off kvm.ignore_msrs=1 kvm.report_ignored_msrs=0 kvm_intel.nested=1 kvm_intel.ept=1 kvm_intel.emulate_invalid_guest_state=0 kvm_intel.enable_shadow_vmcs=1 kvm_intel.enable_apicv=1
```

**Explanation of kernel parameters:**
- `intel_iommu=on` - Enables IOMMU
- `iommu=pt` - Passthrough mode (better performance)
- `vfio-pci.ids=` - Binds NVIDIA GPU to vfio-pci driver at boot
- `video=efifb:off` - Prevents efifb from grabbing the GPU
- `kvm.ignore_msrs=1` - Ignore unknown MSR accesses (some games check these)
- `kvm.report_ignored_msrs=0` - Don't log MSR access attempts
- `kvm_intel.nested=1` - Enable nested virtualization (hides hypervisor better)
- `kvm_intel.ept=1` - Extended Page Tables (performance)
- `kvm_intel.enable_shadow_vmcs=1` - Shadow VMCS (helps hide hypervisor)
- `kvm_intel.enable_apicv=1` - APIC virtualization (performance)

### Configure Loader

```bash
nano /boot/loader/loader.conf
```

```
default  arch.conf
timeout  3
console-mode max
editor   no
```

### Configure mkinitcpio for VFIO

```bash
nano /etc/mkinitcpio.conf
```

Find the `MODULES=` line and modify:
```
MODULES=(vfio_pci vfio vfio_iommu_type1)
```

Regenerate initramfs:
```bash
mkinitcpio -P
```

### Enable NetworkManager

```bash
systemctl enable NetworkManager
```

### Create User Account

```bash
useradd -m -G wheel -s /bin/bash yourusername
passwd yourusername

# Enable sudo for wheel group
EDITOR=nano visudo

# Uncomment this line:
%wheel ALL=(ALL:ALL) ALL
```

### Lenovo Keyboard Port Code Fix (CRITICAL for P16v)

Lenovo laptops, including the P16v, often have keyboard detection issues that can cause:
- Wrong keyboard layout detection
- Non-functioning function keys
- Incorrect key mappings

**Fix this now before rebooting:**

```bash
# Add Lenovo keyboard quirks to modprobe
nano /etc/modprobe.d/lenovo-keyboard.conf
```

Add these lines:
```
# Lenovo keyboard fixes
options atkbd reset=1
options i8042 direct
options i8042 dumbkbd=1
```

**For Portuguese Brazil keyboard specifically:**

```bash
# Set console keymap permanently
echo "KEYMAP=br-abnt2" > /etc/vconsole.conf

# Also set X11 keyboard layout for KDE/Plasma
mkdir -p /etc/X11/xorg.conf.d
nano /etc/X11/xorg.conf.d/00-keyboard.conf
```

Add:
```
Section "InputClass"
    Identifier "system-keyboard"
    MatchIsKeyboard "on"
    Option "XkbLayout" "br"
    Option "XkbVariant" "abnt2"
EndSection
```

**Additional fix for Lenovo ThinkPad keyboards:**

```bash
# Some ThinkPad keyboards need this
nano /etc/modprobe.d/thinkpad-acpi.conf
```

Add:
```
options thinkpad_acpi fan_control=1
```

These fixes ensure:
- ✅ Correct keyboard layout (BR-ABNT2) at boot
- ✅ Proper function key behavior
- ✅ Trackpoint/touchpad work correctly
- ✅ Keyboard recognized on all boots

### Exit and Reboot

```bash
exit           # Exit chroot
umount -R /mnt
reboot
```

Remove USB drive during reboot.

---

## 5. KDE Plasma Installation (Optional)

### First Boot Verification

After rebooting into your new Arch installation, verify keyboard is working correctly:

```bash
# Test if your keyboard layout is correct
# Type these special BR-ABNT2 characters:
# ç Ç ´ ` ^ ~ ª º

# Check loaded keymap
localectl status
# Should show: VC Keymap: br-abnt2

# If keyboard doesn't work at all, you may need to reload the module:
sudo modprobe -r atkbd
sudo modprobe atkbd reset=1

# Verify keyboard device is detected
cat /proc/bus/input/devices | grep -A 5 keyboard
```

**If keyboard still has issues after reboot:**

```bash
# Check kernel messages for keyboard errors
dmesg | grep -i "keyboard\|i8042\|atkbd"

# Try alternative fix
sudo nano /etc/modprobe.d/lenovo-keyboard.conf
```

Add this alternative configuration:
```
options atkbd reset=1 softrepeat=1
options i8042 nomux=1 reset=1 nopnp=1
```

Then:
```bash
sudo mkinitcpio -P
sudo reboot
```

### Install KDE Plasma

After logging in as your user:

```bash
# Install Xorg and KDE Plasma
sudo pacman -S xorg plasma plasma-wayland-session kde-applications

# Enable display manager
sudo systemctl enable sddm
sudo systemctl start sddm
```

**Configure keyboard in KDE Plasma (after first login):**

1. Open System Settings
2. Go to **Input Devices** → **Keyboard**
3. Click **Layouts** tab
4. Check "Configure layouts"
5. Add: **Portuguese (Brazil) - ABNT2**
6. Remove any other layouts
7. Apply

**Alternative: Command line configuration for KDE:**

```bash
# Set keyboard layout for KDE/X11
setxkbmap -model abnt2 -layout br -variant abnt2

# Make it permanent (add to ~/.xprofile)
echo 'setxkbmap -model abnt2 -layout br -variant abnt2' >> ~/.xprofile
```

If you want to stay in CLI mode by default:
```bash
# Don't enable sddm, start it manually when needed:
sudo systemctl start sddm
```

---

## 6. QEMU/KVM/Libvirt Setup

### Install Virtualization Packages

```bash
sudo pacman -S qemu-full libvirt virt-manager virt-viewer dnsmasq bridge-utils ovmf edk2-ovmf iptables-nft swtpm
```

### Enable and Start Libvirt

```bash
sudo systemctl enable libvirtd
sudo systemctl start libvirtd
```

### Add User to Required Groups

```bash
sudo usermod -a -G libvirt $(whoami)
sudo usermod -a -G kvm $(whoami)

# Log out and back in, or:
newgrp libvirt
```

### Configure Libvirt Network

```bash
sudo virsh net-autostart default
sudo virsh net-start default
```

### Create IOMMU Group Checking Script

```bash
nano ~/iommu-groups.sh
```

Add this content:
```bash
#!/bin/bash
for d in /sys/kernel/iommu_groups/*/devices/*; do
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done
```

Make executable and run:
```bash
chmod +x ~/iommu-groups.sh
./iommu-groups.sh | grep -i nvidia
./iommu-groups.sh | grep -i arc
```

**Verify your NVIDIA GPU is in its own IOMMU group or with only its audio device.**

---

## 7. Hardware Passthrough Configuration

### Verify GPU Bound to vfio-pci

```bash
lspci -nnk | grep -A 3 NVIDIA
```

Should show `Kernel driver in use: vfio-pci` for both GPU and Audio.

If not, reboot and check again. If still not working, check your kernel parameters from Section 4.

### Find PCI Addresses

```bash
# Find exact PCI addresses (needed for VM XML)
lspci -D | grep -i nvidia
# Example: 0000:01:00.0 and 0000:01:00.1

# Find USB Controller
lspci -D | grep -i usb
# Example: 0000:00:14.0

# Save these addresses - you'll need them later
```

### Verify IOMMU is Working

```bash
dmesg | grep -i iommu
# Should show: DMAR: IOMMU enabled
```

---

## 8. Windows VM - Maximum Performance & Undetectable

### Context

This configuration is for **game developers who need to test their games in VM environments** with anti-cheat systems. Modern anti-cheat like Vanguard is designed to detect VMs, so thorough testing requires aggressive VM hiding techniques.

### What Vanguard Detects

Riot's Vanguard (and similar anti-cheat systems) check for:
- Hypervisor presence (CPUID leaves)
- VMware/QEMU device signatures
- PCI vendor IDs (Red Hat, VMware, etc.)
- ACPI table strings
- Timing discrepancies
- MAC address OUI patterns
- Registry keys and artifacts
- SMBIOS/DMI mismatches
- CPU topology anomalies
- Memory balloon drivers
- Virtual device drivers

**Our Strategy:**
1. Complete hypervisor signature hiding
2. SMBIOS/DMI spoofing (exact host match)
3. Maximum performance optimization
4. Hardware-level features (TPM, Secure Boot)
5. Registry and file system cleanup
6. Network MAC randomization

---

## 8.1 Advanced Kernel Configuration

We already added advanced parameters in Section 4. Verify they're active:

```bash
cat /proc/cmdline
```

Should include: `kvm.ignore_msrs=1`, `kvm_intel.nested=1`, etc.

---

## 8.2 Host SMBIOS Information Collection

**CRITICAL STEP - This makes your VM undetectable**

```bash
# Save ALL SMBIOS information - we need exact matches
sudo dmidecode -t 0 > ~/smbios-bios.txt
sudo dmidecode -t 1 > ~/smbios-system.txt
sudo dmidecode -t 2 > ~/smbios-baseboard.txt
sudo dmidecode -t 3 > ~/smbios-chassis.txt
sudo dmidecode -t 4 > ~/smbios-processor.txt
sudo dmidecode -t 17 > ~/smbios-memory.txt

# Consolidate everything
sudo dmidecode > ~/smbios-complete.txt

# View your system info
cat ~/smbios-system.txt
```

**Write down these values - you'll enter them in the VM XML:**
- Manufacturer
- Product Name
- Version
- Serial Number
- UUID
- SKU Number
- Family

---

## 8.3 Huge Pages Configuration

For 16GB VM with overhead, we need ~8500 pages:

```bash
# Edit sysctl configuration
sudo nano /etc/sysctl.d/99-hugepages.conf
```

Add:
```
vm.nr_hugepages = 8500
vm.hugetlb_shm_group = 78
```

Find your kvm group ID:
```bash
getent group kvm
# Output: kvm:x:78:yourusername
# Use the number (78) in the config above
```

Apply immediately:
```bash
sudo sysctl -p /etc/sysctl.d/99-hugepages.conf
```

Verify:
```bash
cat /proc/meminfo | grep -i huge
# Should show ~17GB huge pages total
# HugePages_Total: 8500
```

---

## 8.4 ACPI Table Patching

Anti-cheat scans ACPI tables for VM signatures. We'll dump and patch them:

```bash
# Install ACPI tools
sudo pacman -S acpica

# Dump host ACPI DSDT table
sudo cat /sys/firmware/acpi/tables/DSDT > ~/host_dsdt.dat

# Decompile DSDT
iasl -d ~/host_dsdt.dat

# Edit the .dsl file to remove QEMU/BOCHS strings
nano ~/host_dsdt.dsl
```

**In the file, search and replace:**
- `"QEMU"` → `"LENOVO"`
- `"BOCHS"` → `"LENOVO"`
- Any VMware or Red Hat references

**Recompile:**
```bash
iasl -tc ~/host_dsdt.dsl
# This creates host_dsdt.aml

# Copy to a permanent location
mkdir -p ~/.config/vfio
cp ~/host_dsdt.aml ~/.config/vfio/acpi_override.bin
```

---

## 8.5 Complete VM XML Configuration

### Randomize Network MAC Address

Generate a MAC with Lenovo OUI:

```bash
# Lenovo OUI: 28:f1:0e
printf '28:f1:0e:%02x:%02x:%02x\n' $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256))

# Example output: 28:f1:0e:a3:7f:2c
# Write this down for the XML
```

### Create VM XML File

Create a new file for the VM definition:

```bash
nano ~/windows11-gaming.xml
```

**IMPORTANT: You MUST customize these sections with YOUR values:**
1. PCI addresses (from `lspci -D`)
2. SMBIOS information (from `dmidecode`)
3. MAC address (generated above)
4. CPU core numbers (from `lscpu -e`)
5. Input devices (from `ls -l /dev/input/by-id/`)

**Complete XML configuration:**

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <name>windows11-gaming</name>
  <uuid>GENERATE-YOUR-OWN-UUID</uuid>
  <memory unit='GiB'>16</memory>
  <currentMemory unit='GiB'>16</currentMemory>
  <memoryBacking>
    <hugepages/>
    <nosharepages/>
    <locked/>
    <allocation mode='immediate'/>
  </memoryBacking>
  
  <vcpu placement='static'>8</vcpu>
  
  <cputune>
    <!-- ADJUST THESE: Pin to YOUR physical P-cores (check with: lscpu -e) -->
    <!-- Avoid E-cores and hyperthreading -->
    <vcpupin vcpu='0' cpuset='2'/>
    <vcpupin vcpu='1' cpuset='3'/>
    <vcpupin vcpu='2' cpuset='4'/>
    <vcpupin vcpu='3' cpuset='5'/>
    <vcpupin vcpu='4' cpuset='6'/>
    <vcpupin vcpu='5' cpuset='7'/>
    <vcpupin vcpu='6' cpuset='8'/>
    <vcpupin vcpu='7' cpuset='9'/>
    
    <!-- Emulator threads on separate cores -->
    <emulatorpin cpuset='0-1'/>
    
    <!-- I/O threads for disk performance -->
    <iothreadpin iothread='1' cpuset='0'/>
    <iothreadpin iothread='2' cpuset='1'/>
  </cputune>
  
  <iothreads>2</iothreads>
  
  <os>
    <type arch='x86_64' machine='pc-q35-8.2'>hvm</type>
    <loader readonly='yes' secure='yes' type='pflash'>/usr/share/edk2/x64/OVMF_CODE.secboot.fd</loader>
    <nvram template='/usr/share/edk2/x64/OVMF_VARS.secboot.fd'>/var/lib/libvirt/qemu/nvram/windows11-gaming_VARS.fd</nvram>
    <boot dev='hd'/>
    <bootmenu enable='no'/>
  </os>
  
  <features>
    <acpi/>
    <apic/>
    <pae/>
    <hyperv mode='custom'>
      <relaxed state='on'/>
      <vapic state='on'/>
      <spinlocks state='on' retries='8191'/>
      <vpindex state='on'/>
      <runtime state='on'/>
      <synic state='on'/>
      <stimer state='on'>
        <direct state='on'/>
      </stimer>
      <reset state='on'/>
      <vendor_id state='on' value='GenuineIntel'/>
      <frequencies state='on'/>
      <reenlightenment state='on'/>
      <tlbflush state='on'/>
      <ipi state='on'/>
      <evmcs state='off'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
      <hint-dedicated state='on'/>
      <poll-control state='on'/>
    </kvm>
    <vmport state='off'/>
    <ioapic driver='kvm'/>
    <pmu state='off'/>
  </features>
  
  <cpu mode='host-passthrough' check='none' migratable='off'>
    <topology sockets='1' dies='1' cores='8' threads='1'/>
    <cache level='3' mode='emulate'/>
    <feature policy='require' name='invtsc'/>
    <feature policy='require' name='topoext'/>
    <feature policy='disable' name='hypervisor'/>
  </cpu>
  
  <clock offset='localtime'>
    <timer name='rtc' tickpolicy='catchup' track='guest'/>
    <timer name='pit' tickpolicy='discard'/>
    <timer name='hpet' present='no'/>
    <timer name='kvmclock' present='no'/>
    <timer name='hypervclock' present='yes'/>
    <timer name='tsc' present='yes' mode='passthrough'/>
  </clock>
  
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  
  <pm>
    <suspend-to-mem enabled='no'/>
    <suspend-to-disk enabled='no'/>
  </pm>
  
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    
    <!-- NVMe Passthrough - ADJUST /dev/nvmeXnX to YOUR Windows drive -->
    <disk type='block' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native' discard='unmap' iothread='1' queues='8'/>
      <source dev='/dev/nvme1n1'/>
      <target dev='sda' bus='scsi'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
      <boot order='1'/>
    </disk>
    
    <!-- SCSI Controller -->
    <controller type='scsi' index='0' model='virtio-scsi'>
      <driver queues='8' iothread='1'/>
      <address type='pci' domain='0x0000' bus='0x03' slot='0x00' function='0x0'/>
    </controller>
    
    <!-- PCI Controllers -->
    <controller type='pci' index='0' model='pcie-root'/>
    <controller type='pci' index='1' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='1' port='0x10'/>
    </controller>
    <controller type='pci' index='2' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='2' port='0x11'/>
    </controller>
    <controller type='pci' index='3' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='3' port='0x12'/>
    </controller>
    <controller type='pci' index='4' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='4' port='0x13'/>
    </controller>
    
    <!-- NVIDIA GPU Passthrough - ADJUST addresses from: lspci -D | grep NVIDIA -->
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0' multifunction='on'/>
      <rom bar='off'/>
    </hostdev>
    
    <!-- NVIDIA GPU Audio - ADJUST address -->
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x01' slot='0x00' function='0x1'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x1'/>
    </hostdev>
    
    <!-- Network - USE YOUR generated MAC address here -->
    <interface type='network'>
      <mac address='28:f1:0e:XX:XX:XX'/>
      <source network='default'/>
      <model type='virtio'/>
      <driver name='vhost' queues='8'/>
    </interface>
    
    <!-- USB Host Controller Passthrough - OPTIONAL, ADJUST address -->
    <!-- Find with: lspci -D | grep USB -->
    <!-- Uncomment if you want entire USB controller passed through -->
    <!--
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x00' slot='0x14' function='0x0'/>
      </source>
    </hostdev>
    -->
    
    <!-- Audio: PulseAudio/PipeWire passthrough -->
    <sound model='ich9'>
      <codec type='micro'/>
      <audio id='1'/>
    </sound>
    <audio id='1' type='pulseaudio' serverName='/run/user/1000/pulse/native'>
      <input mixingEngine='no' fixedSettings='yes' voices='1' bufferLength='100'/>
      <output mixingEngine='no' fixedSettings='yes' voices='1' bufferLength='100'/>
    </audio>
    
    <!-- Virtual TPM 2.0 - Required for Vanguard -->
    <tpm model='tpm-tis'>
      <backend type='emulator' version='2.0'/>
    </tpm>
    
    <!-- Input: evdev passthrough for lowest latency -->
    <!-- Find devices: ls -l /dev/input/by-id/ -->
    <!-- ADJUST these paths to YOUR keyboard and mouse -->
    <input type='evdev'>
      <source dev='/dev/input/by-id/usb-YOUR_KEYBOARD_NAME-event-kbd' grab='all' grabToggle='ctrl-ctrl' repeat='on'/>
    </input>
    <input type='evdev'>
      <source dev='/dev/input/by-id/usb-YOUR_MOUSE_NAME-event-mouse' grab='all' grabToggle='ctrl-ctrl'/>
    </input>
    
    <!-- Fallback tablet for mouse (if evdev doesn't work) -->
    <input type='tablet' bus='virtio'/>
    
    <memballoon model='none'/>
    
    <!-- RNG for better entropy -->
    <rng model='virtio'>
      <backend model='random'>/dev/urandom</backend>
    </rng>
  </devices>
  
  <!-- CRITICAL: SMBIOS Spoofing - REPLACE ALL VALUES with YOUR dmidecode output -->
  <qemu:commandline>
    <!-- Hide QEMU/KVM signatures -->
    <qemu:arg value='-cpu'/>
    <qemu:arg value='host,host-cache-info=on,kvm=off,l3-cache=on,-hypervisor,+invtsc,migratable=no,hv-vendor-id=GenuineIntel'/>
    
    <!-- SMBIOS Type 0 (BIOS) - CUSTOMIZE with YOUR values -->
    <qemu:arg value='-smbios'/>
    <qemu:arg value='type=0,vendor=LENOVO,version=N3HET61W (1.47),date=07/15/2024,uefi=on'/>
    
    <!-- SMBIOS Type 1 (System) - CRITICAL - USE YOUR EXACT VALUES -->
    <qemu:arg value='-smbios'/>
    <qemu:arg value='type=1,manufacturer=LENOVO,product=21KV0000US,version=ThinkPad P16v Gen 1,serial=YOUR_SERIAL_HERE,uuid=YOUR_UUID_HERE,sku=LENOVO_MT_21KV,family=ThinkPad P16v Gen 1'/>
    
    <!-- SMBIOS Type 2 (Baseboard) - USE YOUR values -->
    <qemu:arg value='-smbios'/>
    <qemu:arg value='type=2,manufacturer=LENOVO,product=21KV0000US,version=SDK0T76461 WIN,serial=YOUR_MB_SERIAL,asset=NO Asset Tag'/>
    
    <!-- SMBIOS Type 3 (Chassis) - USE YOUR values -->
    <qemu:arg value='-smbios'/>
    <qemu:arg value='type=3,manufacturer=LENOVO,version=None,serial=YOUR_CHASSIS_SERIAL,asset=NO Asset Tag,sku=LENOVO_MT_21KV'/>
    
    <!-- SMBIOS Type 4 (Processor) - Match YOUR CPU -->
    <qemu:arg value='-smbios'/>
    <qemu:arg value='type=4,manufacturer=Intel(R) Corporation,version=13th Gen Intel(R) Core(TM) i9-13900H'/>
    
    <!-- ACPI table override - USE YOUR patched ACPI file -->
    <qemu:arg value='-acpitable'/>
    <qemu:arg value='file=/home/YOUR_USERNAME/.config/vfio/acpi_override.bin'/>
  </qemu:commandline>
</domain>
```

### Define the VM

```bash
# Validate XML syntax first
xmllint --noout ~/windows11-gaming.xml

# If no errors, define the VM
sudo virsh define ~/windows11-gaming.xml

# Verify it was created
sudo virsh list --all
```

---

## 8.6 Windows Guest VM Hiding

### Start the VM

```bash
sudo virsh start windows11-gaming

# If you have a monitor connected to the GPU, it should display Windows
# Otherwise, use virt-viewer:
virt-viewer windows11-gaming
```

Wait for Windows to boot (it should boot from your existing Windows installation on the passed-through NVMe).

### Install NVIDIA Drivers

1. Download latest NVIDIA Game Ready drivers from nvidia.com
2. Install in Windows
3. Reboot

### Clean VM Registry Artifacts

In Windows, create `C:\clean_vm_artifacts.reg`:

```reg
Windows Registry Editor Version 5.00

; Remove VM-related registry keys
[-HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\VBoxGuest]
[-HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\VBoxMouse]
[-HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\VBoxSF]
[-HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\VBoxVideo]
[-HKEY_LOCAL_MACHINE\SOFTWARE\Oracle]
[-HKEY_LOCAL_MACHINE\HARDWARE\ACPI\DSDT\VBOX__]
[-HKEY_LOCAL_MACHINE\HARDWARE\ACPI\FADT\VBOX__]
[-HKEY_LOCAL_MACHINE\HARDWARE\ACPI\RSDT\VBOX__]
[-HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Services\vmicheartbeat]
[-HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Services\vmicvss]
[-HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Services\vmicshutdown]
[-HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Services\vmicexchange]

; Change SystemProductName to match host
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SystemInformation]
"SystemProductName"="ThinkPad P16v Gen 1"
"SystemManufacturer"="LENOVO"

[HKEY_LOCAL_MACHINE\HARDWARE\DESCRIPTION\System\BIOS]
"SystemManufacturer"="LENOVO"
"SystemProductName"="21KV0000US"
"BIOSVendor"="LENOVO"
"BIOSVersion"="N3HET61W (1.47)"
```

Import it (run as Administrator):
```cmd
regedit /s C:\clean_vm_artifacts.reg
```

### Hide Hypervisor via Registry

Create `C:\hide_hypervisor.reg`:

```reg
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management]
"FeatureSettingsOverride"=dword:00000003
"FeatureSettingsOverrideMask"=dword:00000003

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\DeviceGuard]
"EnableVirtualizationBasedSecurity"=dword:00000000

[HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\DeviceGuard]
"EnableVirtualizationBasedSecurity"=dword:00000000
```

Import and reboot:
```cmd
regedit /s C:\hide_hypervisor.reg
shutdown /r /t 0
```

### Disable Windows Hypervisor Platform

Open PowerShell as Administrator:

```powershell
# Disable all virtualization features
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -NoRestart

# Disable hypervisor launch
bcdedit /set hypervisorlaunchtype off

# Reboot
Restart-Computer
```

### Verify Hardware Detection

Open PowerShell:

```powershell
# Check BIOS info (should show LENOVO)
Get-WmiObject -Class Win32_BIOS | Select-Object Manufacturer, Version, SerialNumber

# Check System info (should show LENOVO)
Get-WmiObject -Class Win32_ComputerSystem | Select-Object Manufacturer, Model

# Check for VM signatures (should be empty)
Get-WmiObject -Class Win32_BaseBoard | Select-Object Manufacturer, Product

# Check TPM (should show Present and Ready)
Get-Tpm

# Check Secure Boot (should return True)
Confirm-SecureBootUEFI
```

**Everything should show LENOVO, not QEMU/Red Hat/VMware.**

---

## 8.7 Performance Tuning in Windows

### Disable Unnecessary Services

```powershell
Set-Service -Name "DiagTrack" -StartupType Disabled
Set-Service -Name "dmwappushservice" -StartupType Disabled
Set-Service -Name "WSearch" -StartupType Disabled
```

### Set High Performance Power Plan

```powershell
powercfg /setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c
```

### Disable Game DVR

```powershell
reg add "HKEY_CURRENT_USER\System\GameConfigStore" /v "GameDVR_Enabled" /t REG_DWORD /d 0 /f
reg add "HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\GameDVR" /v "AppCaptureEnabled" /t REG_DWORD /d 0 /f
```

### Enable Hardware-accelerated GPU Scheduling

```powershell
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\GraphicsDrivers" /v "HwSchMode" /t REG_DWORD /d 2 /f
```

### Disable Visual Effects

```powershell
reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\VisualEffects" /v "VisualFXSetting" /t REG_DWORD /d 2 /f
```

### NVIDIA Control Panel Settings

1. Open NVIDIA Control Panel
2. Manage 3D Settings → Global Settings:
   - Power management mode: **Prefer maximum performance**
   - Texture filtering - Quality: **High performance**
   - Threaded optimization: **On**
   - Low Latency Mode: **Ultra**

Reboot Windows.

---

## 8.8 CPU Isolation & Automation

### Create CPU Isolation Script

On the host (Arch Linux):

```bash
sudo nano /usr/local/bin/isolate-cpus.sh
```

Add:
```bash
#!/bin/bash

if [ "$1" == "isolate" ]; then
    # Set performance governor
    echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
    echo "CPU governor set to performance"
    
elif [ "$1" == "restore" ]; then
    # Return to powersave
    echo powersave | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
    echo "CPU governor set to powersave"
    
else
    echo "Usage: $0 {isolate|restore}"
    exit 1
fi
```

Make executable:
```bash
sudo chmod +x /usr/local/bin/isolate-cpus.sh
```

### Automate with Libvirt Hooks

```bash
sudo mkdir -p /etc/libvirt/hooks
sudo nano /etc/libvirt/hooks/qemu
```

Add:
```bash
#!/bin/bash

GUEST_NAME="$1"
OPERATION="$2"

if [ "$GUEST_NAME" == "windows11-gaming" ]; then
    if [ "$OPERATION" == "prepare" ]; then
        /usr/local/bin/isolate-cpus.sh isolate
    elif [ "$OPERATION" == "release" ]; then
        /usr/local/bin/isolate-cpus.sh restore
    fi
fi
```

Make executable:
```bash
sudo chmod +x /etc/libvirt/hooks/qemu
sudo systemctl restart libvirtd
```

---

## 8.9 Additional Performance Features

### Looking Glass (Optional)

Looking Glass provides near-native display without needing a second monitor:

```bash
# Install Looking Glass
sudo pacman -S looking-glass

# Configure shared memory
sudo nano /etc/tmpfiles.d/10-looking-glass.conf
```

Add (replace YOUR_USER):
```
f /dev/shm/looking-glass 0660 YOUR_USER kvm -
```

Apply:
```bash
sudo systemd-tmpfiles --create /etc/tmpfiles.d/10-looking-glass.conf
```

Add to VM XML (inside `<devices>`):
```xml
<shmem name='looking-glass'>
  <model type='ivshmem-plain'/>
  <size unit='M'>64</size>
</shmem>
```

In Windows, download and install Looking Glass Host from: https://looking-glass.io/downloads

### Network Performance

```bash
sudo nano /etc/sysctl.d/98-network.conf
```

Add:
```
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
```

Apply:
```bash
sudo sysctl -p /etc/sysctl.d/98-network.conf
```

---

## 9. Testing & Verification

### Level 1: Basic VM Detection Test

In Windows PowerShell:

```powershell
$detections = @()

# Check hypervisor bit
$hypervisor = (Get-WmiObject Win32_ComputerSystem).HypervisorPresent
if ($hypervisor) { $detections += "Hypervisor bit set" }

# Check BIOS
$bios = Get-WmiObject Win32_BIOS
if ($bios.Manufacturer -match "QEMU|VMware|VBox") {
    $detections += "VM BIOS: $($bios.Manufacturer)"
}

# Check System
$system = Get-WmiObject Win32_ComputerSystem
if ($system.Manufacturer -match "QEMU|VMware|VBox") {
    $detections += "VM System: $($system.Manufacturer)"
}

# Check for virtual devices
$devices = Get-PnpDevice | Where-Object {
    $_.FriendlyName -match "VM|Virtual|QEMU|VBox|VMware|Red Hat"
}
if ($devices) {
    $detections += "Virtual devices: $($devices.Count)"
}

# Results
if ($detections.Count -eq 0) {
    Write-Host "SUCCESS: No VM signatures detected!" -ForegroundColor Green
} else {
    Write-Host "FAILED: VM signatures found:" -ForegroundColor Red
    $detections | ForEach-Object { Write-Host "  - $_" -ForegroundColor Yellow }
}
```

**Target: 0 detections**

### Level 2: Performance Benchmarks

Install and run:
- **3DMark** - GPU benchmark (target: >95% of bare metal)
- **Cinebench R23** - CPU benchmark (target: >95% of bare metal)
- **CrystalDiskMark** - NVMe speed (target: >3000 MB/s read)

### Level 3: Vanguard Testing

```powershell
# Install Riot Vanguard
# Download Valorant or League of Legends

# After installation, check services
Get-Service -Name vgc
Get-Service -Name vgk
# Both should show "Running"

# Launch game and test
# If you can join matches, SUCCESS!
```

### Common Vanguard Error Codes

- **Error 7**: TPM/Secure Boot issue
- **Error 43/44**: Potential VM detection
- **Error 51/52**: Vanguard driver not loaded (often VM detection)

If you get errors 43, 44, 51, or 52, review Section 11 Troubleshooting.

---

## 10. macOS Sequoia VM - Intel Arc Passthrough

### Prerequisites

```bash
sudo pacman -S python-pip
git clone https://github.com/thenickdude/KVM-Opencore.git
cd KVM-Opencore
./make.sh
# This creates OpenCore.iso
```

### Download macOS

```bash
git clone https://github.com/acidanthera/OpenCorePkg
cd OpenCorePkg/Utilities/macrecovery
python3 macrecovery.py -b Mac-827FAC58A8FDFA22 -m 00000000000000000 download

# Convert to raw
qemu-img convert -O raw BaseSystem.dmg ~/macos-sequoia.img
```

### Create macOS Disk

```bash
qemu-img create -f qcow2 ~/macos-sequoia-disk.qcow2 128G
```

### macOS VM XML

Create `~/macos-sequoia.xml`:

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <name>macos-sequoia</name>
  <memory unit='GiB'>8</memory>
  <vcpu placement='static'>6</vcpu>
  
  <os>
    <type arch='x86_64' machine='pc-q35-8.2'>hvm</type>
    <loader readonly='yes' type='pflash'>/usr/share/edk2/x64/OVMF_CODE.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/macos_VARS.fd</nvram>
  </os>
  
  <features>
    <acpi/>
    <apic/>
  </features>
  
  <cpu mode='custom' match='exact' check='none'>
    <model fallback='forbid'>Haswell-noTSX</model>
    <topology sockets='1' cores='6' threads='1'/>
    <feature policy='require' name='invtsc'/>
  </cpu>
  
  <clock offset='utc'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
  </clock>
  
  <devices>
    <disk type='file' device='disk'>
      <driver name='qemu' type='raw'/>
      <source file='/home/YOUR_USER/KVM-Opencore/OpenCore.iso'/>
      <target dev='sda' bus='sata'/>
      <boot order='1'/>
    </disk>
    
    <disk type='file' device='disk'>
      <driver name='qemu' type='raw'/>
      <source file='/home/YOUR_USER/macos-sequoia.img'/>
      <target dev='sdb' bus='sata'/>
    </disk>
    
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/home/YOUR_USER/macos-sequoia-disk.qcow2'/>
      <target dev='sdc' bus='sata'/>
    </disk>
    
    <graphics type='vnc' port='-1' autoport='yes' listen='127.0.0.1'>
      <listen type='address' address='127.0.0.1'/>
    </graphics>
    
    <input type='tablet' bus='usb'/>
    <input type='keyboard' bus='usb'/>
  </devices>
  
  <qemu:commandline>
    <qemu:arg value='-device'/>
    <qemu:arg value='isa-applesmc,osk=ourhardworkbythesewordsguardedpleasedontsteal(c)AppleComputerInc'/>
    <qemu:arg value='-smbios'/>
    <qemu:arg value='type=2'/>
    <qemu:arg value='-cpu'/>
    <qemu:arg value='Haswell-noTSX,vendor=GenuineIntel,+invtsc,vmware-cpuid-freq=on'/>
  </qemu:commandline>
</domain>
```

### Start macOS VM

```bash
sudo virsh define ~/macos-sequoia.xml
sudo virsh start macos-sequoia
virt-viewer macos-sequoia
```

Follow macOS installation prompts.

**Note**: Intel Arc GPU passthrough for macOS is experimental. You may need to use VNC/software rendering initially.

---

## 11. Troubleshooting

### Keyboard Not Working or Wrong Layout (Lenovo Specific)

**Symptom**: Keyboard doesn't respond, or keys produce wrong characters.

**Fix 1 - Reload keyboard module:**
```bash
sudo modprobe -r atkbd
sudo modprobe atkbd reset=1
```

**Fix 2 - Try alternative i8042 parameters:**
```bash
sudo nano /etc/modprobe.d/lenovo-keyboard.conf
```

Try these combinations (one at a time):
```
# Option A (most common)
options i8042 direct dumbkbd=1

# Option B (if A doesn't work)
options i8042 nomux=1 reset=1 nopnp=1

# Option C (for newer ThinkPads)
options i8042 direct dumbkbd=0 noloop=1
```

After each change:
```bash
sudo mkinitcpio -P
sudo reboot
```

**Fix 3 - Verify keyboard is detected:**
```bash
# Check if keyboard device exists
ls -l /dev/input/by-path/ | grep kbd

# Check kernel detection
dmesg | grep -i "keyboard\|i8042"

# If no keyboard found, you may need:
sudo pacman -S linux-firmware
```

**Fix 4 - BR-ABNT2 specific issues:**
```bash
# Make sure vconsole is configured
cat /etc/vconsole.conf
# Should show: KEYMAP=br-abnt2

# If not:
echo "KEYMAP=br-abnt2" | sudo tee /etc/vconsole.conf

# Test loadkeys
sudo loadkeys br-abnt2

# If that works, regenerate initramfs
sudo mkinitcpio -P
```

**For KDE/X11 keyboard issues:**
```bash
# Reset X keyboard
setxkbmap -model abnt2 -layout br -variant abnt2

# Make permanent
echo 'setxkbmap -model abnt2 -layout br -variant abnt2' >> ~/.xprofile
```

### GPU Not Binding to vfio-pci

```bash
# Check driver
lspci -nnk | grep -A 3 NVIDIA

# Force bind
echo "0000:01:00.0" | sudo tee /sys/bus/pci/drivers/nouveau/unbind
echo "0000:01:00.0" | sudo tee /sys/bus/pci/drivers/vfio-pci/bind
```

Add to `/etc/modprobe.d/vfio.conf`:
```
softdep nouveau pre: vfio-pci
softdep nvidia pre: vfio-pci
```

### Black Screen After GPU Passthrough

Extract GPU VBIOS:
```bash
cd /sys/bus/pci/devices/0000:01:00.0/
echo 1 | sudo tee rom
sudo cat rom > ~/nvidia-vbios.rom
echo 0 | sudo tee rom
```

Add to VM XML:
```xml
<hostdev ...>
  <rom file='/home/YOUR_USER/nvidia-vbios.rom'/>
</hostdev>
```

### VM Performance is Poor

**Checklist:**
```bash
# Verify huge pages in use
cat /proc/meminfo | grep Huge

# Check CPU governor
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
# Should all say "performance" when VM is running

# Verify GPU passthrough
lspci -nnk | grep -A 3 vfio-pci

# Check VM is using correct cores
virsh vcpuinfo windows11-gaming
```

### Vanguard Detects VM

Try these in order:

1. **Verify all SMBIOS matches exactly**
```bash
sudo dmidecode -t 1
# Compare with VM XML
```

2. **Check for virtual device drivers**
```powershell
# In Windows
Get-PnpDevice | Where-Object {$_.FriendlyName -match "Red Hat|QEMU|VirtIO"}
# Should return nothing
```

3. **More aggressive CPU masking**

Edit VM XML, change CPU line in qemu:commandline:
```xml
<qemu:arg value='-cpu'/>
<qemu:arg value='host,kvm=off,hv_vendor_id=GenuineIntel,-hypervisor,+invtsc'/>
```

### Audio Crackling

Install PipeWire:
```bash
sudo pacman -S pipewire pipewire-pulse
systemctl --user enable --now pipewire pipewire-pulse
```

Configure low latency:
```bash
mkdir -p ~/.config/pipewire
cp /usr/share/pipewire/pipewire.conf ~/.config/pipewire/
nano ~/.config/pipewire/pipewire.conf
```

Set: `default.clock.min-quantum = 64`

---

## 12. Useful Commands Reference

```bash
# VM Management
sudo virsh list --all              # List all VMs
sudo virsh start windows11-gaming  # Start VM
sudo virsh shutdown windows11-gaming # Graceful shutdown
sudo virsh destroy windows11-gaming  # Force stop
sudo virsh edit windows11-gaming   # Edit XML config

# Monitoring
virsh domstats windows11-gaming    # VM statistics
cat /proc/meminfo | grep Huge      # Huge pages usage
nvidia-smi                          # GPU status (from host)
./iommu-groups.sh                   # Check IOMMU groups

# Performance
lscpu -e                            # CPU topology
lspci -nnk                          # PCI devices and drivers
dmesg | grep -i vfio                # VFIO kernel messages

# Network
sudo virsh net-list --all           # List networks
ip link show                        # Network interfaces
```

---

## Additional Resources

- [Arch Wiki - PCI Passthrough](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)
- [r/VFIO Subreddit](https://reddit.com/r/VFIO)
- [OpenCore Guide](https://dortania.github.io/OpenCore-Install-Guide/)
- [Looking Glass Documentation](https://looking-glass.io/docs)

---

## Disclaimer

This guide is for **game developers testing their own games with anti-cheat systems**.

- Running macOS on non-Apple hardware violates Apple's EULA
- Bypassing anti-cheat in games you don't own/develop violates TOS
- Improper configuration can cause data loss - always backup
- Hardware passthrough can be unstable on some laptops

The author is not responsible for any consequences of following this guide.

---

## Guide Information

**Version**: 1.1 (Complete - Single Comprehensive Guide)  
**Last Updated**: September 29, 2025  
**Covers**:
- ✅ Arch Linux installation (complete)
- ✅ Lenovo keyboard port code fix (BR-ABNT2 support)
- ✅ BIOS & bootloader configuration
- ✅ KVM/QEMU setup
- ✅ Windows VM with GPU passthrough
- ✅ VM performance tuning (huge pages, CPU pinning, I/O threads)
- ✅ VM stealth/undetectability (SMBIOS spoofing, ACPI patching, registry cleaning)
- ✅ Anti-cheat testing procedures
- ✅ macOS VM setup
- ✅ Comprehensive troubleshooting

**Changelog**:
- v1.1: Added Lenovo keyboard port code fix for ThinkPad P16v with BR-ABNT2 keyboard support
- v1.0: Initial complete guide

**This is a single, complete guide containing all steps from installation through tuning and undetectability.**

You're ready to begin your setup journey!
