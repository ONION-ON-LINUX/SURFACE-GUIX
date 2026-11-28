# Guix System on a Surface Laptop 2 (nonguix + mwifiex)

Notes from installing GNU Guix System on a Microsoft Surface Laptop 2 (i7-8650U). The stock Guix installer uses linux-libre, which has no firmware for the Marvell mwifiex wifi chip, so the install needs a custom nonguix installer image.

I couldn't find another write-up of Guix on a Surface, so this may be useful to someone. These are one person's notes, not an official guide.

**Result:** installed and booted Guix System (btrfs root, nonguix kernel Linux 7.1.13) with working wifi. **It then froze as soon as I did startx trying to get DWM booted**, so this is a partial success; see [Status](#status-unresolved-freeze-under-x).
**Not covered:** touchpad/SAM/keyboard quirks beyond what worked out of the box, and suspend.

## Hardware
- Surface Laptop 2, i7-8650U, 16 GB RAM, NVMe
- Wifi: Marvell mwifiex (PCIe), needs non-free firmware
- Build host: a VirtualBox VM running Guix (the Surface's old OS was wiped during install)

## Overview
1. Install Guix in a VM.
2. Add the nonguix channel and build an installer ISO that carries the nonfree kernel and linux-firmware.
3. Write the ISO to USB, boot the Surface, get wifi, install.
4. Make sure the installed system's `config.scm` also uses the nonfree kernel and firmware.

## 1. Build the installer ISO (in the VM)

`~/.config/guix/channels.scm`:
```scheme
(cons* (channel
        (name 'nonguix)
        (url "https://gitlab.com/nonguix/nonguix")
        (introduction
         (make-channel-introduction
          "897c1a470da759236cc11798f4e0a5f7d4d59fbc"
          (openpgp-fingerprint
           "2A39 3FFF 68F4 EF7A 3D29  12AF 6F51 20A0 22FB B2D5"))))
       %default-channels)
```
Then:
```
guix pull
git clone https://gitlab.com/nonguix/nonguix.git ~/src/nonguix
guix system image --image-type=iso9660 ~/src/nonguix/nongnu/system/install.scm
```

Alternative: SystemCrafters publishes prebuilt nonguix installer ISOs (github.com/SystemCrafters/guix-installer). I built my own, but that would likely have skipped the next problem.

### Gotcha: the kernel compiles locally and eats disk
In my run there was no substitute for the `linux-7.1.13` derivation, so it compiled from source in the VM (about 3 hours on 4 cores). It died twice with `No space left on device` (log lines from `ld`/`objcopy` on DVB driver modules). The nonguix kernel builds a very large module set.
- Give the VM **60 GB+** of disk before starting, not 32.
- To grow a VirtualBox VDI: shut the VM down, `VBoxManage modifymedium "<disk>.vdi" --resize 61440`, then inside the guest `parted /dev/sda` → `print` → answer `Fix` when asked about the GPT backup header → `resizepart 3 100%`, then `resize2fs /dev/sda3`.
- Check the real build log (`zcat /var/log/guix/drvs/xx/<hash>.drv.gz | tail`) instead of trusting Guix's generic "may be low on disk space" hint. Here it was accurate.

### Gotcha: getting the ISO out of the VM
VirtualBox USB passthrough wouldn't attach the stick even with the extension pack installed. What worked instead:
```
# in the VM
cd /gnu/store && python3 -m http.server 8000

# on the host
VBoxManage controlvm <vmname> natpf1 "isoserver,tcp,,8080,,8000"
wget http://localhost:8080/<hash>-image.iso
sudo dd if=<hash>-image.iso of=/dev/sdX status=progress && sync
```
Double-check `/dev/sdX` with `lsblk` before `dd`.

## 2. Boot the Surface and get wifi
Boot the USB (Surface UEFI boot menu). The wifi interface (`wlp2s0` here) showed up, which confirmed the firmware was loaded. Then:
```
rfkill unblock wifi
ip link set wlp2s0 up
wpa_supplicant -B -i wlp2s0 -c <(wpa_passphrase "SSID" "PASSWORD")
dhclient wlp2s0
echo "nameserver 8.8.8.8" > /etc/resolv.conf   # resolvconf failed in the live env
```

## 3. Install
I used the graphical installer with manual partitioning (EFI partition + btrfs root + swap). After I clicked Finish it sat silently with no installer process running. Partitions were formatted and mounted under `/mnt` and `/mnt/etc/config.scm` had been written. So I finished by hand from a shell:

1. Edit `/mnt/etc/config.scm` to add the nonfree kernel/firmware (see below).
2. `guix system init /mnt/etc/config.scm /mnt`

### Gotcha: `install-bootloader.scm` failure
`guix system init` repeatedly failed at the final step with errors about the bootloader install script. Rebooting the live USB did not fix it. This got past it:
```
guix system init --no-bootloader /mnt/etc/config.scm /mnt
```
(`guix gc --verify=contents,repair` did report modified store items in the live environment, but repairing them didn't get past the error. I don't know the root cause.)

### Installing GRUB by hand
- The default `grub-install` on the live image was the BIOS (`grub-pc`) build. Build and use the EFI one:
  ```
  guix build grub-efi        # prints /gnu/store/...-grub-efi-2.14
  export PATH=<that path>/sbin:$PATH
  grub-install --boot-directory=/mnt/boot --efi-directory=/mnt/boot/efi \
               --target=x86_64-efi --bootloader-id=GUIX --removable
  ```
  With `--removable` the loader landed at `EFI/BOOT/BOOTX64.EFI`, not `EFI/GUIX/`. That's expected and the firmware finds it without a boot entry.
- `guix system build` output has no `grub.cfg`, and `grub-mkconfig` needs the target's real root. I ran it inside a chroot:
  ```
  mkdir -p /mnt/dev /mnt/proc /mnt/sys
  mount --bind /dev /mnt/dev; mount --bind /proc /mnt/proc; mount --bind /sys /mnt/sys
  mount --bind /gnu/store /mnt/gnu/store      # make the live store's tools visible
  chroot /mnt /gnu/store/<bash>/bin/bash
  ```
  Inside the chroot there is no usable PATH, so I had to add store paths for tools `grub-mkconfig` calls (`coreutils` for `basename`, `sed`, and so on) and call `grub-mkconfig` by its full path with `-o /boot/grub/grub.cfg`. Find the store paths from a second TTY with `find /gnu/store -maxdepth 1 -name 'coreutils-9*'`.
- The system then booted into the installed Guix.

## 4. `config.scm` (the important part)
Based on the installer-generated config, with the nonguix additions marked. UUIDs replaced with placeholders.

```scheme
(use-modules (gnu)
             (nongnu packages linux)          ; added
             (nongnu system linux-initrd))    ; added
(use-service-modules cups desktop networking ssh xorg)

(operating-system
  (kernel linux)                              ; added: full Linux kernel
  (initrd microcode-initrd)                   ; added
  (firmware (list linux-firmware))            ; added: includes mwifiex firmware
  (locale "en_US.utf8")
  (timezone "America/Los_Angeles")
  (keyboard-layout (keyboard-layout "us"))
  (host-name "SURFACE-LAPTOP-2-ON-GUIX")

  (users (cons* (user-account
                  (name "onion")
                  (comment "Onion")
                  (group "users")
                  (home-directory "/home/onion")
                  (supplementary-groups '("wheel" "netdev" "audio" "video")))
                %base-user-accounts))

  (services
   (append (list (service openssh-service-type)
                 (service network-manager-service-type)
                 (service wpa-supplicant-service-type)
                 (service ntp-service-type)
                 (service gpm-service-type))
           %base-services))

  (bootloader (bootloader-configuration
                (bootloader grub-efi-bootloader)
                (targets (list "/boot/efi"))
                (keyboard-layout keyboard-layout)))

  (swap-devices (list (swap-space (target (uuid "<SWAP-UUID>")))))

  (file-systems (cons* (file-system
                         (mount-point "/")
                         (device (uuid "<ROOT-UUID>" 'btrfs))
                         (type "btrfs"))
                       (file-system
                         (mount-point "/boot/efi")
                         (device (uuid "<EFI-UUID>" 'fat32))
                         (type "vfat"))
                       %base-file-systems)))
```
After install, add this to `~/.config/guix/channels.scm` too, so `guix pull` and `guix system reconfigure` can see the `nongnu` modules.

## Other gotchas
- The live installer's `/gnu/store` is a read-only squashfs plus a RAM overlay. Rebooting resets it.
- Packages installed with `guix install` aren't on `PATH` until you source `~/.guix-profile/etc/profile`. `sudo` uses root's PATH, so call with a full path if needed.
- There is no FHS layout: `/bin/bash` and `/bin/sh` don't exist (only `/usr/bin/env` is provided), so scripts with `#!/bin/bash` shebangs need `#!/usr/bin/env bash`.
- Store hashes are easy to mistype (`l`/`1`/`i` look alike on the Linux console font). Capture paths with `$(...)` instead of retyping them.

## Status: unresolved freeze under X
Installed, booting, wifi working. I then built dwm from source and got it running briefly under X (needed tty permission and `~/.xinitrc` fixes, and disabling i915 PSR/FBC), but the machine **hard-froze the instant dwm rendered**, with no response to any input. I suspect a Surface-specific ACPI/SAM kernel problem and gave up on Guix on this laptop.

For comparison, NixOS with the linux-surface patched kernel (`nixos-hardware`, `microsoft/surface/common`) boots fine on the same machine.

**Unverified guess:** nonguix's kernel has no linux-surface patches, and a linux-surface kernel package for nonguix might fix the freeze. I haven't tried it.

## Help wanted
- Does anyone have a working Surface kernel package for Guix/nonguix?
- Has anyone run X or Wayland on a Surface under Guix without the freeze?
- Is there a root cause for the `install-bootloader.scm` failure during `guix system init`?

## Installer ISO
A prebuilt copy of the ISO this process produces is attached to the repo's Releases page (compressed with `xz`, with `SHA256SUMS`). It is not committed to the repo itself.
