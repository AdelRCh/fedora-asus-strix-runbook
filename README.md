# Linux Setup Runbook on Asus ROG Strix G53QR (AMD/NVIDIA hybrid architecture)
This repository contains the kernel parameters, environment flags, and bootloader configurations needed to stabilize Fedora Linux over a non-standard laptop architecture.

This Runbook is under construction as I document the commands typed to solve the issues encountered while installing Fedora and setting things up within the OS.

### System Configuration Profile
* **Host Hardware:** ASUS ROG Strix G53QR (AMD Ryzen + NVIDIA RTX 3070 Mobile)
* **Target Kernel Layer:** Fedora Workstation 44 (with CUDA Pipeline Enabled for future projects)

### Kernel-Level Issues and Fixes

#### 0. Inability to create user on workstation due to the issues below
* **Issue:** Cannot login due to non-existing user credentials, inability to create user credentials using GUI (edge-case)
* **Root Cause:** All of the issues below
* **Fix:** On GRUB launch (picking the boot option and pressing `e`), used all the flags listed below, removed `rhgb quiet` to track every event and crash, and added `rw init=/bin/bash` to skip the login screen and launch Bash (pressed F10 after the changes)
  
  ```bash
  nomodeset 3 nvme_core.default_ps_max_latency_us=0 processor.max_cstate=1 rw init=/bin/bash
  ```

  From there, the commands went as follows:

  ```bash
  # Setting root password to avoid being locked out of root access on launch when necessary
  passwd root
  # Adding a user account and appending it to the sudo group: (your_username) can change to fit
  useradd -m -G wheel -s /bin/bash your_username
  # Supplying a password for your new username session
  passwd your_username
  # Informing Fedora's security layer to relabel the filesystem on next boot, preventing permission
  # errors from locking us out after forcing password changes
  touch /.autorelabel
  # Flushing storage bus cache to disk, then rebooting
  sync
  reboot -f
  ```

#### 1. The Boot-Loop Issue (`nomodeset 3`)
* **Issue:** Black-screen freezes and kernel panics during boot, and occasional absence of Fedora ASUS ROG launch screen.
* **Root Cause:** Conflict between the open-source Nouveau driver and the hardwired USB-C DisplayPort line routing straight to the dGPU.
* **Fix:** Dropped the kernel into a headless console state by appending `nomodeset 3` directly to GRUB on launch, bypassing display server initialization entirely to force a clean CLI environment for driver injection. Once NVIDIA's proprietary drivers were installed, this specific issue was neutralized, and direct CUDA usage became possible.

  On boot, we selected our option, pressed `e`, removed `rhgb quiet` and changed the flags as follows (then pressed F10):
  ```bash
  nomodeset 3 nvme_core.default_ps_max_latency_us=0 processor.max_cstate=1
  ```
  
  If needed, we connect to WiFi (for those not connecting directly via Ethernet):

  ```bash
  # If connecting through WiFi, the following is needed:
  # Listing all wireless networks and their MAC addresses
  nmcli device wifi list
  
  # Forcing a connection directly to the specific MAC address if the name has odd characters
  # Otherwise, replace "XX:XX:XX:XX:XX" by the network's name
  sudo nmcli device wifi connect "XX:XX:XX:XX:XX:XX" password "YOUR_WIFI_PASSWORD"
  
  # Verifying local IP routing block just in case
  ip route show
  ```

  From there, we proceed with installing the proprietary drivers and ASUS's utility to manage the DisplayPort connection:
  ```bash
  # Uninstalling the Nouveau driver packages to prevent future driver conflicts
  sudo dnf remove xorg-x11-drv-nouveau -y
  
  # Enabling RPM Fusion repository and install official NVIDIA kernel modules
  sudo dnf install akmod-nvidia xorg-x11-drv-nvidia-cuda -y

  # Installing the ASUS ROG hardware's power routing and profile switching utility manager
  sudo dnf install supergfxctl -y
  sudo systemctl enable --now supergfxd
  
  # Keeping external DisplayPort lines active using Hybrid mode
  supergfxctl -m Hybrid

  # We aren't done with this session yet. Check below for the last instructions.

  ```

#### 2. Storage Bus Stability (NVMe Minimum Latency)
* **Issue:** Random I/O stalls at random on Linux runtime.
* **Fix:** Configured the NVMe internal drive's controller sleep states via the kernel command line parameters (`nvme_core.default_ps_max_latency_us=0`), disabling deep PCIe power states to guarantee zero-latency storage disk-write access.

#### 3. Power-State Freezes (AMD C-States Throttling)
* **Issue:** Hard system locks when idling or shifting at launch and out of high-compute PySpark processing loads.
* **Root Cause:** Hardware-level freeze during transition into deep CPU execution sleep states (C6/C7 states) on mobile Ryzen processors.
* **Fix:** Set `processor.max_cstate=1` in the bootloader parameter set, forcing the CPU to remain inside working voltage boundaries.

  ```bash
  # Setting GRUB parameters to all Linux kernels after successful reboot, after creating users:
  sudo grubby --update-kernel=ALL --args="nvme_core.default_ps_max_latency_us=0 processor.max_cstate=1"

  # Rebooting cleanly, now that we are done
  sync
  sudo reboot
  ```

#### 4. The Bootloader Crash (GRUB Restoration)
* **Situation:** Wishing to ensure that the bootloader remembers our last chosen OS option in case of updates or reboots, added/modified two parameters: `GRUB_SAVEDEFAULT=true` and `GRUB_ENABLE_BLSCFG=true` (the latter against my intuition...)
* **Incident:** Messing with the dynamic state variable `GRUB_ENABLE_BLSCFG=true` caused the bootloader configuration to drop into a corrupted state, forcing Linux to default to Ramdisk boot (NOT GOOD)
* **Fix:** Booted via Fedora Live USB sandbox, mounted the host system via `chroot` environment pipelines, re-established the EFI system path mappings, and manually compiled a fresh GRUB binary image directly to the physical boot sectors.

  First, the boot flags and the removal of `rhgb quiet` (F10 afterwards):
  ```bash
  nomodeset 3 nvme_core.default_ps_max_latency_us=0 processor.max_cstate=1
  ```

  From there, we get to fixing the bootloader (good thing to remember as Windows will sometimes break GRUB upon updating):
  ```bash
  # Finding out what our hard drive ID is (the laptop has an NVMe drive)
  # In my case, it showed as nvme0n1p6
  sudo fdisk -l | grep -E "Disk /dev/nvme|/dev/nvme"

  # Mounting Linux partition and BOOT EFI partition to /mnt
  sudo mount /dev/nvme0n1p6 /mnt
  sudo mount /dev/nvme0n1p1 /mnt/boot/efi

  # Bind-mounting the host OS's active hardware communication nodes
  for dir in /dev /proc /sys /run; do sudo mount --bind $dir /mnt$dir; done

  # Time for the good part: we execute commands from the host OS's perspective
  sudo chroot /mnt

  # Purging old GRUB instance
  cd /boot/efi/EFI/fedora/
  sudo rm grub.cfg

  # Fixed the underlying tag that introduced corruption
  sudo sed -i 's/GRUB_ENABLE_BLSCFG=true/GRUB_ENABLE_BLSCFG=false/g' /etc/default/grub

  # Regenerating GRUB
  grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg

  # Ensure future logins are NOT blocked by resynchronizing security tokens
  touch /.autorelabel

  # And we're done with our chroot from here
  sync
  exit

  # Outside of chroot, we reboot
  sudo reboot -f
  
  ```

  
