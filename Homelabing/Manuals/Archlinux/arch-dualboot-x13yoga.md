# Arch Linux Dual Boot Install Log

**Machine:** Lenovo ThinkPad X13 Yoga
**Hostname:** VeraX13
**Date started:** 2026-08-25
**Goal:** Arch Linux alongside Windows 11, shared EFI partition, UEFI mode, hibernation working

This log records both what was done and why. The reasoning matters more than the commands,
because the commands are on the wiki and the reasoning is what I will have forgotten in
six months when something breaks.

---

## Key values for this machine


| Thing | Value |
|---|---|
| Root partition | `/dev/nvme0n1p5` |
| Root UUID | `ed0adbb2-c29e-4bc4-bfc1-6b4965e45de3` |
| ESP | `/dev/nvme0n1p1`, mounted at `/boot/efi` |
| ESP UUID | `96CE-F798` |
| Swap file | `/swapfile`, 12 GiB |
| **resume_offset** | **`311296`** |
| CPU | Intel (`intel-ucode`) |
| Sleep states available | `s2idle` only, no `deep`/S3 |

---

## Baseline hardware

| Item | Value |
|---|---|
| Disk | SKHynix HFS256GDE9X081N |
| Capacity | 238.47 GiB (256060514304 bytes, 500118192 sectors) |
| Sector size | 512 B logical / 512 B physical |
| Disk label | GPT |
| Disk GUID | 966C8B33-7AEA-4280-90B5-39F08E5AF017 |
| RAM | 8 GB physical, 7.5 GiB usable |
| Firmware mode | UEFI 64-bit |
| Secure Boot | Disabled for install |
| Install media | archiso, kernel 7.1.5-arch1-2, label `ARCH_202608`, USB at `/dev/sda` |

---

## Phase 0: Windows preparation

### 0.1 Shrink the Windows volume

Done in Windows Disk Management. Shrank C: to free **65.3 GiB**.

The freed space landed **between** the Windows data partition (p3) and the Windows recovery
partition (p4), not at the end of the disk. This is normal. On GPT the partition *number*
and the physical *position* are independent, so the new Linux partition became p5 while
sitting physically before p4. `fdisk` prints a warning about this. The warning is harmless.

### 0.2 Finding: factory BitLocker in clear-key state

`lsblk -f` from the live environment reported BitLocker on C:, even though the Windows UI
showed no BitLocker enabled.

`manage-bde -status` from an admin PowerShell explained the contradiction:

```
Conversion Status:    Used Space Only Encrypted
Percentage Encrypted: 100.0%
Protection Status:    Protection Off
Key Protectors:       None Found
```

**What this state means.** The volume was genuinely encrypted, but with a *clear key*. No
TPM protector, no recovery password, no PIN. The unlock key sat in plaintext on the disk,
so Windows unlocked it automatically without consulting anything.

This is Lenovo/Microsoft pre-provisioning. The factory image encrypts the drive ahead of
time so Device Encryption can be finalized later without waiting for a full-disk pass. On
this machine it was never finalized, because it was never signed into a Microsoft account.

Two consequences followed:

- **Disabling Secure Boot was harmless here.** BitLocker's TPM protector seals the key
  against a set of platform measurements, one of which (PCR 7) covers Secure Boot state.
  Change Secure Boot on a machine where protection is genuinely on, and the TPM refuses to
  release the key, and Windows demands the 48-digit recovery key on next boot. With no TPM
  protector, nothing was sealed, so nothing broke.
- **The Windows data was not actually protected.** A clear key is readable by anyone who
  pulls the SSD.

**The general rule, for next time:** always check `manage-bde -status` before touching
firmware settings on a Windows machine. `Protection Status: On` means get the recovery key
first. `Protection Off` with `Key Protectors: None Found` means you are free to proceed.

### 0.3 Decrypt C:

Decided to fully decrypt rather than leave the half-state.

The reason is that the half-state is unstable in a way that gives no warning. Sign into
Windows with a Microsoft account, or let a Windows Update finalize Device Encryption, and
protectors get created silently and a recovery key gets escrowed to the cloud. From that
moment a firmware or bootloader change can lock the machine, and nothing tells you the
state changed.

```powershell
manage-bde -off C:
manage-bde -status C:     # poll until "Fully Decrypted"
```

Used-space-only decryption on NVMe took a few minutes.

### 0.4 Disable Fast Startup

```powershell
powercfg /h off
```

Fast Startup makes "shut down" actually hibernate the kernel session. The NTFS filesystem
is then left in a dirty state, and mounting it read-write from Linux risks corruption.
Turning it off makes shutdown a real shutdown.

---

## Phase 1: Live environment pre-flight

### 1.1 Confirm UEFI boot mode

```bash
cat /sys/firmware/efi/fw_platform_size
```

Returned `64`.

**Why this is the first check.** This file only exists if the system booted via UEFI. If
the USB boots in legacy/CSM mode, the Arch install ends up BIOS-style and cannot coexist
with a UEFI Windows install. You find out at reboot, after everything else is done. Check
it first, not last.

### 1.2 Partition table as found

```
Device           Start        End    Sectors    Size  Type
/dev/nvme0n1p1    2048     411647     409600    200M  EFI System
/dev/nvme0n1p2  411648     444415      32768     16M  Microsoft reserved
/dev/nvme0n1p3  444416  361474047  361029632  172.2G  Microsoft basic data
   << free       361474048  498421759  136947712   65.3G >>
/dev/nvme0n1p4 498421760  500115455    1693696    827M  Windows recovery environment
```

Gap check: `498421760 - 361474048 = 136947712` sectors = 65.3 GiB. Matches Disk Management,
so nothing was lost or misaligned in the shrink.

### 1.3 Clock

RTC was in UTC, NTP active but unsynced until the network came up.

**The dual-boot clock problem.** Linux assumes the hardware clock is UTC. Windows assumes
it is local time. With both installed, whichever booted last "corrects" the RTC and the
other one is then wrong by the UTC offset (2 hours for CEST).

Fix on the **Windows** side rather than the Linux side, because making Linux use local time
breaks things subtly around DST transitions:

```
reg add "HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /t REG_DWORD /d 1 /f
```

**Not yet done.** Run this after the first successful Windows boot post-install.

### 1.4 Network

```bash
rfkill unblock wifi
iwctl
```

Inside the `iwctl` prompt:

```
device list
station wlan0 get-networks
station wlan0 connect Skynet
exit
```

Verified with `ping -c 3 archlinux.org` and `timedatectl` showing synchronized.

### 1.5 Keyboard layout in the live console

```bash
loadkeys de-latin1
```

Only affects the live environment. The installed system gets its own setting via
`/etc/vconsole.conf` (see 3.3).

**Trap worth knowing:** if you set a root password under one layout and later type it at a
prompt running a different layout, characters like `y`/`z`, `-`, `/` and anything behind
Alt Gr land in different places and you cannot log in. Keep install-time passwords to plain
letters and digits, or make sure the layout is consistent.

### 1.6 SSH into the live environment

Done to make copy-paste of output practical.

archiso runs `sshd` and permits root login already, but root has no password, so password
auth fails until one is set.

```bash
passwd
systemctl status sshd          # start it if not running
ip -brief addr show wlan0
```

Then from the client machine: `ssh root@<ip>`

**Always work inside tmux over SSH.** A dropped connection kills the foreground process. If
that happens during `pacstrap`, you are left with a half-populated root filesystem.

```bash
tmux new -s arch          # first time
tmux attach -t arch       # to get back in, including after a drop
```

`duplicate session: arch` means the session already exists. Attach, do not create.

---

## Phase 2: Partitioning

### 2.1 Decisions and why

**Reuse the existing ESP, do not create a second one.**
`/dev/nvme0n1p1` already exists and Windows boots from it. One EFI System Partition is
designed to serve multiple operating systems. A second ESP is a well-known route to a boot
order that breaks unpredictably, because firmware picks one and you cannot always tell
which.

**Bootloader: GRUB, not systemd-boot.**
The ESP is only 200 MB and Windows already uses part of it. systemd-boot requires the
kernel, initramfs, fallback initramfs and microcode to all live on the ESP. With two
kernels installed that is comfortably over 200 MB, and it grows with every kernel. It fails
mid-upgrade, which is the worst time. GRUB keeps `/boot` on the root partition and puts
roughly 10 MB on the ESP. It also handles the Windows entry via `os-prober`.

So the ESP mounts at `/boot/efi`, not at `/boot`.

**No separate /home.** 65 GiB is too small for the split to be worth the rigidity.

**Swap file, not swap partition.** A file can be resized without touching the partition
table, and when Windows is eventually removed and the root partition grown into the freed
space, a swap file comes along with no work. A swap partition sitting mid-disk would be in
the way.

**Hibernation: yes.** `cat /sys/power/mem_sleep` returned `[s2idle]` with no `deep` option,
meaning this machine only supports Modern Standby and not classic S3 sleep. Modern Standby
drains roughly 1 to 4 percent of battery per hour while "asleep", so a laptop left in a bag
over a weekend arrives flat with the session lost. Hibernation is the fix.

**Swap size: 12 GiB.** Swap is doing two jobs, holding the hibernation image and acting as
real swap. On 8 GB of RAM you will genuinely page out, and sizing swap exactly at RAM means
a hibernate attempt can fail when swap is already partly used. Rule of thumb for
hibernation is RAM plus the square root of RAM, about 11 GiB, rounded up to 12.

### 2.2 Create the partition

```bash
fdisk /dev/nvme0n1
```

- `n`, then Enter three times to accept partition 5 and the full free gap
- `p` to print and verify **before** writing
- `w` to write

Verified before writing:

- p5 starts 361474048, ends 498421759, size 65.3G
- p1 through p4 unchanged, byte for byte
- Type reads `Linux filesystem` (fdisk's default, correct)
- Alignment clean: `361474048 / 2048 = 176501` exactly, so it starts on a 1 MiB boundary

Nothing is written to disk until `w`. `q` quits without saving.

### 2.3 Format and mount

```bash
mkfs.ext4 /dev/nvme0n1p5
mount /dev/nvme0n1p5 /mnt
mount --mkdir /dev/nvme0n1p1 /mnt/boot/efi
findmnt -R /mnt
```

**Only p5 gets formatted.** Running `mkfs` on `/dev/nvme0n1p1` destroys the shared ESP and
Windows stops booting.

`findmnt -R /mnt` should show exactly two lines. Linux allows stacking mounts on the same
path, so running the mount commands twice hides the first mount under the second and
produces a confusing `genfstab` result. If you see a duplicate, `umount -R /mnt` and
remount.

---

## Phase 3: Base system

### 3.1 Mirrors

```bash
reflector --country Germany,Netherlands --protocol https --age 12 --sort rate --save /etc/pacman.d/mirrorlist
```

archiso ranks mirrors at boot but does it globally. Pinning to nearby countries is
noticeably faster. Individual mirrors timing out or returning 403 during the rating run is
normal, they simply get dropped from the ranking.

### 3.2 pacstrap

```bash
pacstrap -K /mnt base linux linux-lts linux-firmware intel-ucode base-devel \
  grub efibootmgr os-prober networkmanager sof-firmware \
  nano vim sudo man-db man-pages texinfo git
```

Why these specifically:

| Package | Reason |
|---|---|
| `linux-lts` | A second kernel in the GRUB menu. When a mainline update breaks wifi or the touchpad, reboot into LTS instead of reaching for the USB stick. Costs ~100 MB on root, which GRUB makes free. |
| `intel-ucode` | CPU microcode. Confirmed Intel via `grep vendor_id /proc/cpuinfo`. |
| `sof-firmware` | Required for audio on modern hardware. Skip it and you get silence with no obvious cause. |
| `os-prober` | Finds the Windows bootloader and adds it to the GRUB menu. |
| `networkmanager` | Wifi after first reboot. Installing no network stack and rebooting into a machine that cannot get online to fix itself is the classic Arch first-boot mistake. |

### 3.3 fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab
```

**Run exactly once.** It appends. Running it twice produces duplicate entries and a system
that fails to boot.

`-U` uses UUIDs rather than device names. Device names like `/dev/nvme0n1p5` can change if
hardware is added or reordered. UUIDs cannot.

Result:

```
UUID=ed0adbb2-c29e-4bc4-bfc1-6b4965e45de3  /          ext4  rw,relatime  0 1
UUID=96CE-F798                             /boot/efi  vfat  rw,relatime,fmask=0022,...  0 2
```

---

## Phase 4: Configure the installed system

### 4.1 What arch-chroot actually is

```bash
arch-chroot /mnt
```

`chroot` changes what a process considers to be `/`. Point it at `/mnt`, run a shell, and
from that shell's perspective `/mnt` **is** the root of the filesystem. So `nano
/etc/locale.gen` inside the chroot edits `/mnt/etc/locale.gen` on the real system. You are
configuring the new install rather than the live USB.

Plain `chroot` alone is not enough. Much tooling expects `/proc`, `/sys`, `/dev` and `/run`
to exist, and in a bare chroot they are empty directories. `pacman` cannot verify keys,
`grub-install` cannot see block devices, `hwclock` cannot reach the RTC.

`arch-chroot` is a wrapper that bind-mounts those from the running system first, then calls
`chroot`. It also copies in `/etc/resolv.conf` so DNS works inside. On `exit` it unmounts
what it set up.

**Consequence:** configuring a system that is not running. No init, no systemd, no
services. Pacstrap prints `Skipped: Running in chroot`, and `systemctl
enable` works inside but `systemctl start` does not.

### 4.2 Time and locale

```bash
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc
```

Uncommented in `/etc/locale.gen`:

```
en_US.UTF-8 UTF-8
de_DE.UTF-8 UTF-8
```

```bash
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

**LANG stays English despite being in Germany.** Error messages stay greppable and
searchable, which matters a lot on a rolling release where you will be pasting errors into
search engines. The German locale is still generated, so `LC_TIME` and `LC_PAPER` can be
set to `de_DE.UTF-8` later for European date and paper formats without German error text.

### 4.3 Console keyboard

```bash
echo "KEYMAP=de-latin1" > /etc/vconsole.conf
```

This is what the `sd-vconsole` mkinitcpio warning during pacstrap was about. The file did
not exist yet, so the initramfs fell back to defaults. It gets picked up on the next
initramfs rebuild, which `grub-mkconfig` and any kernel update will trigger.

Used `de-latin1-nodeadkeys`

This covers the TTY only. A graphical session sets its layout separately.

### 4.4 Hostname

```bash
echo "VeraX13" > /etc/hostname
```

`/etc/hosts`:

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   VeraX13.fitz.box VeraX13
```

### 4.5 Users

```bash
passwd                                  # root password
useradd -m -G wheel -s /bin/bash sticks
passwd sticks
EDITOR=nano visudo                      # uncomment: %wheel ALL=(ALL:ALL) ALL
```

**Use `visudo`, never edit `/etc/sudoers` directly.** It syntax-checks before saving. A
broken sudoers file means no sudo at all, and combined with a disabled root login that is a
rescue-USB situation.

`-m` creates the home directory. `-G wheel` adds group membership that the sudoers line
then grants rights to.

### 4.6 Swap file with hibernation support

```bash
mkswap -U clear --size 12G --file /swapfile
swapon /swapfile
echo '/swapfile none swap defaults 0 0' >> /etc/fstab
```

Confirmed:

```
NAME      TYPE SIZE USED PRIO
/swapfile file  12G   0B   -1
```

Then the resume offset:

```bash
filefrag -v /swapfile | awk '$1=="0:" {print substr($4, 1, length($4)-2)}'
```

Result: **`311296`**

**Why an offset is needed at all.** On resume, the kernel has to read the hibernation image
before any filesystem is mounted. It cannot look up a file by path, because there is no
filesystem driver active yet. So it needs the physical block where the swap file starts,
expressed as an offset within the partition. `filefrag` reports the extent map and the
first extent's start block is that number.

**This is fragile in one specific way.** The offset is tied to the swap file's physical
location on the ext4 volume. Resize the filesystem, move the file, or restore it from a
backup, and the offset changes. Hibernation then *appears* to work but you get a cold boot
instead of your session, with no error. If that ever happens, re-run `filefrag` and update
the kernel parameter.

Relevant when Windows is eventually removed and the root partition is grown.

### 4.7 mkinitcpio hooks: no resume hook needed

```bash
grep '^HOOKS' /etc/mkinitcpio.conf
```

```
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block filesystems fsck)
```

This is the **systemd-based** hook set, not the older busybox one. Older guides tell you to
add a `resume` hook between `filesystems` and `fsck`. That applies to the busybox hooks
only. With the `systemd` hook present, resume is handled by
`systemd-hibernate-resume-generator`, which is already included and reads the kernel
command line directly.

So: no mkinitcpio change required. Only the kernel parameters in GRUB.

---

## Phase 5: Bootloader

### 5.1 Kernel parameters

Edited `/etc/default/grub`:

```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet resume=UUID=ed0adbb2-c29e-4bc4-bfc1-6b4965e45de3 resume_offset=311296"
GRUB_DISABLE_OS_PROBER=false
```

`resume=` tells the kernel which filesystem holds the hibernation image. `resume_offset=`
tells it where in that filesystem the swap file physically begins. Both are needed, because
at resume time no filesystem is mounted yet and the kernel cannot resolve a path.

**`GRUB_DISABLE_OS_PROBER=false` is not optional here.** GRUB ships with os-prober disabled
by default for security reasons. Leave it disabled and the Windows entry never appears in
the menu, which looks exactly like the install having destroyed Windows. It has not. The
entry is simply never generated.

The original `GRUB_CMDLINE_LINUX_DEFAULT` line was commented out rather than deleted. GRUB
parses this file as shell, so the last assignment wins and the commented line is inert.

### 5.2 Install and generate

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

`--efi-directory=/boot/efi` and not `/boot`, because of the ESP mount decision in 2.1.

The generation output is the checkpoint. This line is the one that matters:

```
Found Windows Boot Manager on /dev/nvme0n1p1@/EFI/Microsoft/Boot/bootmgfw.efi
```

Also confirmed both kernels and microcode:

```
Found linux image: /boot/vmlinuz-linux-lts
Found initrd image: /boot/intel-ucode.img /boot/initramfs-linux-lts.img
Found linux image: /boot/vmlinuz-linux
Found initrd image: /boot/intel-ucode.img /boot/initramfs-linux.img
```

### 5.3 Verify before rebooting

```bash
grep -m2 'resume' /boot/grub/grub.cfg
ls /boot/efi/EFI/
```

Both parameters present in the generated linux lines, and `/boot/efi/EFI/` contained `Boot`,
`GRUB`, `Microsoft`. Windows bootloader intact.

### 5.4 Services

```bash
systemctl enable NetworkManager
systemctl enable fstrim.timer
```

`systemctl enable` works in a chroot because it only creates symlinks. `systemctl start`
does not, because there is no running init.

`fstrim.timer` runs weekly TRIM. Without it an SSD gradually loses track of which blocks are
free and write performance degrades.

---

## Phase 6: Exit and first boot

```bash
exit                      # leave the chroot
swapoff /mnt/swapfile
umount -R /mnt
reboot
```

Order matters: `swapoff` before `umount`, since the swap file lives on the filesystem being
unmounted. Note the paths regain the `/mnt` prefix once outside the chroot.

If `umount -R /mnt` reports the target is busy, `fuser -vm /mnt` shows what is holding it.
Usual cause is a shell whose working directory is under `/mnt`, fixed with `cd /`.

USB stick pulled during the reboot.

### Result

GRUB menu appeared. Booted into Arch, logged in as `sticks`.

```bash
nmcli device wifi connect "Skynet" --ask
ping -c 3 archlinux.org
```

Wifi up.

```bash
sudo systemctl hibernate
```

Machine wrote the image, powered off fully, and resumed into the existing session on
power-on. Hibernation confirmed working on the first attempt, meaning `resume_offset=311296`
is correct.

---

## Status: base install complete and verified

- [x] UEFI dual boot, shared 200 MB ESP, Windows bootloader intact
- [x] GRUB with both `linux` and `linux-lts` entries
- [x] Windows Boot Manager entry present
- [x] Wifi via NetworkManager
- [x] Hibernation to a 12 GiB swap file, resume verified

---

## Remaining

- [x] Boot Windows once and confirm it still starts
- [x] Set `RealTimeIsUniversal` in the Windows registry (see 1.3) to fix the two-hour clock skew
- [x] Install a desktop environment
- [ ] Re-enable Secure Boot with `sbctl` signing (optional)
- [ ] When Windows is removed: grow p5, then **re-run `filefrag` and update `resume_offset`**, or
      hibernation will silently cold-boot instead of resuming

---

## Phase 7: Desktop and graphics

### 7.1 Kernel headers

```bash
sudo pacman -S linux-headers linux-lts-headers
```

`linux` and `linux-lts` were already installed by pacstrap. Only the headers were missing.

Headers are needed to compile out-of-tree kernel modules, which in practice means DKMS
packages: NVIDIA proprietary, VirtualBox, some wifi drivers, ZFS.

**Install headers for both kernels, not just one.** With headers for `linux` but not
`linux-lts`, a DKMS module builds for one kernel and silently does not exist on the other.
That is discovered at the worst moment: booting into LTS to escape a broken update, only to
find a needed module missing.

### 7.2 Graphics

```bash
lspci -k | grep -A3 -i vga
```

Every X13 Yoga generation is Intel integrated graphics, no discrete option.

**There is no driver to install.** The Intel graphics driver is `i915` and it is built into
the kernel. What gets installed is userspace libraries:

```bash
sudo pacman -S mesa vulkan-intel intel-media-driver libva-utils vulkan-tools
```

| Package | Purpose |
|---|---|
| `mesa` | OpenGL implementation (already pulled in by the desktop) |
| `vulkan-intel` | Vulkan driver |
| `intel-media-driver` | VA-API hardware video decode. Matters on battery: without it, browser video decodes on CPU |
| `libva-utils`, `vulkan-tools` | Verification only |

**Do not install `xf86-video-intel`.** It is the unmaintained X11 DDX driver and causes
tearing and crashes on modern hardware. The kernel modesetting driver is correct, and on
Wayland the question does not arise. Stale guides still recommend it.

Verify:

```bash
vainfo               # expect a list of VAProfileH264 / VAProfileHEVC entries
vulkaninfo --summary
```

### 7.3 GNOME

```bash
sudo pacman -S gnome gnome-tweaks iio-sensor-proxy
sudo systemctl enable gdm
```

`iio-sensor-proxy` feeds accelerometer data to the desktop. Without it, folding the screen
does not trigger rotation.

### 7.4 KDE Plasma 6 alongside GNOME

Both desktops can coexist, selected at the login screen. The friction is real though:

- **Portal conflicts.** Both `xdg-desktop-portal-gtk` and `xdg-desktop-portal-kde` end up
  installed. These handle file pickers and screen sharing for sandboxed apps. With both
  present the wrong dialog sometimes appears and screen sharing can pick the wrong backend.
- **Menu clutter.** Every KDE app appears in GNOME's launcher and vice versa.
- **Mimetype defaults fight.** Installing one desktop reassigns defaults the other set.

```bash
sudo pacman -S plasma-meta
sudo pacman -S dolphin konsole ark gwenview okular spectacle kate kcalc
sudo pacman -S maliit-keyboard
```

`plasma-meta` is roughly 2 GB and includes the Wayland session, network applet, power
management and Discover. The application list is deliberate rather than
`kde-applications-meta`, which is about 4 GB of software that mostly goes unopened.

`maliit-keyboard` is Plasma 6's on-screen keyboard. Without it there is no touch keyboard in
tablet mode.

**Display manager.** GDM launches Plasma sessions fine, so nothing needs changing while
evaluating. If Plasma wins:

```bash
sudo systemctl disable gdm
sudo systemctl enable sddm
```

Disable before enable. Only one can own the `display-manager.service` symlink and `enable`
fails while the other holds it.

---

## Appendix A: Full-disk encryption, for next time

This install has an unencrypted root. That is a gap worth closing on any future build,
particularly on a portable machine.

### Why it is awkward to add retroactively

`cryptsetup reencrypt --encrypt --reduce-device-size 32m` can encrypt an existing filesystem
in place. It works, but it shrinks the filesystem to make room for the LUKS header and
rewrites every block. An interruption from power loss or a panic leaves a partially
encrypted partition. Recoverable via the resume feature, but a full backup first is
mandatory. Roughly an hour on a 65 GiB partition.

**GRUB makes it worse.** GRUB's LUKS2 support is poor and it cannot handle the default
argon2id key derivation function at all. The workarounds are converting the header to
pbkdf2, which weakens it against brute force, or keeping `/boot` unencrypted on a separate
partition. Neither is good.

### The design to use instead, decided at install time

1. **Give the ESP 1 GB.** This is the decision that must be made up front, because growing
   an ESP later means moving Windows partitions. The 200 MB Lenovo default is what forced
   GRUB in this install.
2. **LUKS2 on the root partition.**
3. **systemd-boot with Unified Kernel Images**, not GRUB. A UKI bundles kernel, initramfs
   and cmdline into a single signed EFI binary. This needs the larger ESP.
4. **Enrol the key into the TPM:** `systemd-cryptenroll --tpm2-device=auto`. Unlocks
   automatically at boot with no passphrase, sealed against PCR 7.
5. **Secure Boot on, with your own keys via `sbctl`.** This is what makes step 4 meaningful.
   Without Secure Boot, PCR 7 constrains nothing and an attacker can boot a modified kernel
   that dumps the key.

That architecture is what the factory BitLocker setup in Phase 0.2 was reaching for. The
clear-key state it was left in is exactly what happens when steps 4 and 5 never complete:
encryption that provides no protection at all.

### Hibernation interaction

An encrypted swap file inside an encrypted root works. The resume happens after the LUKS
unlock, so the initramfs prompts for the passphrase before it knows whether it is resuming
or cold booting. Not a blocker, just different from the unencrypted flow.

---

## Recovery procedure

If the system ever fails to boot, this gets you back to a working shell inside the install:

```bash
# boot the archiso USB, then
mount /dev/nvme0n1p5 /mnt
mount /dev/nvme0n1p1 /mnt/boot/efi
arch-chroot /mnt
```

From there, fix whatever broke, re-run `grub-mkconfig -o /boot/grub/grub.cfg` if the issue is
bootloader related, then `exit`, `umount -R /mnt`, `reboot`.
