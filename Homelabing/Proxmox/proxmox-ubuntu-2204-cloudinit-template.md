# Proxmox: Ubuntu 22.04 Minimal Cloud-Init Template

Reference VM: **901** (`ubuntu-2204-minimal-template`)
First test clone: **104** (`ubuntu-test`)
Source: LearnLinuxTV, "Proxmox VE - How to build an Ubuntu 22.04 Template (Updated Method)"
Image: Ubuntu 22.04 "Jammy Jellyfish" minimal cloud image

## 1. Download the cloud image

On the Proxmox host shell:

```
wget https://cloud-images.ubuntu.com/minimal/releases/jammy/release/ubuntu-22.04-minimal-cloudimg-amd64.img
```

## 2. Create the base VM (no disk yet)

In the Proxmox web GUI, create a new VM with ID 901:

- **General**: VMID `901`, name `ubuntu-2204-minimal-template`
- **OS**: "Do not use any media"
- **System**: default settings
- **Disks**: remove the default disk. The disk gets added later from the imported image, not here.
- **CPU/Memory**: low values are fine (1-2 cores, 1024-2048 MB). Clones can be resized up individually later.
- **Network**: `vmbr0`, VirtIO
- Confirm creation.

## 3. Add a Cloud-Init drive

Hardware tab → Add → CloudInit Drive → select storage (`local-lvm`) → Add.

## 4. Configure the Cloud-Init tab

- IP Config: DHCP
- User/password: set as needed per clone
- SSH keys: paste your public key(s) into the **SSH public key** field, one per line if using more than one. Existing keys (e.g. ones already used for GitHub/GitLab) work fine here, there's no requirement to generate dedicated ones.
- Click **Regenerate Image** to write these settings into the Cloud-Init config drive. Without this, the drive can still hold stale or default values, and the next boot won't pick up what you just entered in the form.

## 5. Add a serial console to the VM

Back on the Proxmox host shell:

```
qm set 901 --serial0 socket --vga serial0
```

Ubuntu cloud images send console output to a serial port instead of VGA, since they're built for headless cloud platforms. Without this, the noVNC console shows a black screen even though the VM is running fine.

## 6. Prepare the downloaded image

```
mv ubuntu-22.04-minimal-cloudimg-amd64.img ubuntu-22.04.qcow2
qemu-img resize ubuntu-22.04.qcow2 32G
```

The Ubuntu cloud image, despite its `.img` extension, is actually a qcow2-format file internally. Whether `qm importdisk` determines the source format from the file's actual content or from its extension isn't fully confirmed, sources on this point are mixed. Given that, and that renaming costs nothing, keep this step. Skipping it risks a silent source-format mismatch during import, which lines up with what happens if a downstream tool trusts the name over the content.

## 7. Import the disk into the VM

```
qm importdisk 901 ubuntu-22.04.qcow2 local-lvm
```

This creates an "Unused Disk" entry on VM 901. It's not attached to a controller yet and won't boot as-is.

## 8. Attach the imported disk

Hardware tab → select "Unused Disk 0" → Edit → attach (as `scsi0`).

Since the underlying storage is SSD-backed:

- Check **Discard**, so the guest can tell the storage backend when blocks are freed (e.g. after deleting files), letting thin-provisioned storage reclaim that space instead of the disk usage only ever growing.
- Under **Advanced**, check **SSD emulation**, so the guest OS sees the disk as an SSD rather than a spinning drive. This affects how the guest schedules TRIM and other assumptions it makes about the disk.

Then OK.

## 9. Set the boot order

Options tab → Boot Order → check `scsi0`, move it to the top.

## 10. Convert the VM to a template

Right-click VM 901 → Convert to Template (or `qm template 901` on the shell).

**Important:** once 901 is a template, never start it directly. Always clone first, then boot the clone. Ubuntu cloud images ship with an empty `/etc/machine-id`, and cloud-init does not regenerate this value on its own. As long as 901 stays unbooted, every clone generates its own unique ID on its own first boot. If 901 itself ever gets booted, its machine-id gets written permanently into the template disk, and every future clone would inherit the same ID, causing DHCP/network identity conflicts.

## 11. Clone and boot

Right-click the template → Clone → choose the clone type → assign a new VMID and name → Clone.

Both clone types only become available once 901 is an actual template. Which one to pick depends on the use case:

| | Linked Clone | Full Clone |
|---|---|---|
| **What it does** | New disk stores only the differences from the template's disk | Copies the entire disk into a new, independent one |
| **Creation speed** | Near instant | Slower, has to copy the full disk size |
| **Storage used at creation** | Minimal, grows as the clone changes | Full disk size right away |
| **Dependency on the template** | Yes. The template must stay untouched and can't be deleted while linked clones exist | None. The template can be changed or removed after cloning |
| **Storage backend requirement** | Needs snapshot support (LVM-thin, ZFS, Ceph). `local-lvm` on a default Proxmox install qualifies | Works on any storage type |
| **Good for** | Spinning up and tearing down many similar VMs quickly (dev/test, throwaway labs) | Production VMs, or anything that needs to be fully independent from the template |

This setup used **Full Clone**, matching the tutorial.

Start the new clone.

If the first start attempt fails with something like:

```
trying to acquire lock...
TASK ERROR: can't lock file '/var/lock/qemu-server/lock-<VMID>.conf' - got timeout
```

this just means the clone task (copying the disk) hadn't finished yet. Wait a moment and start it again.

## 12. Install the QEMU Guest Agent (on the clone, first boot)

```
sudo apt install qemu-guest-agent
```

Reboot the clone.

The guest agent is a small daemon that runs inside the VM and talks to the Proxmox host over a dedicated virtio-serial channel. Without it, Proxmox has no visibility into what's happening inside the guest OS and no clean way to ask it to do anything. With it installed and running, Proxmox can:

- **Show the real IP address** in the VM's Summary panel. Without the agent, Proxmox has no way to know it, since that information lives inside the guest.
- **Shut down and reboot cleanly.** Without the agent, the Shutdown/Reboot buttons rely on an ACPI signal, which most Linux guests handle fine but isn't guaranteed. With the agent, Proxmox asks the guest OS directly to shut down, the same as running the command locally.
- **Freeze and thaw the filesystem during backups.** When `vzdump` takes a snapshot backup, the agent briefly pauses filesystem writes so the backup captures a consistent state, instead of one that might be mid-write.
- **Run commands and query guest info from the host** (`qm agent <vmid> exec ...`), useful for scripting without needing SSH access.

After this, Start/Stop/Shutdown/Reboot from the Proxmox web UI work reliably.

## Recommended follow-up (not done yet, optional)

Enable the "QEMU Guest Agent" option on the template itself (Options tab, or `qm set 901 --agent enabled=1`), in addition to installing the package inside the guest. This tells Proxmox to expect the agent, which enables IP address reporting in the VM summary panel and cleaner filesystem freeze/thaw during backups. Worth setting on 901 before the next clone, since it only needs to be done once on the template.

## 13. Verify SSH key access

With the public key(s) added to the template's Cloud-Init tab and Regenerate Image clicked, clone a fresh test VM and connect using the matching private key:

```
ssh -i ~/.ssh/id_ed25519 <user>@<clone-ip>
```

**Confirmed working:** logging in from WSL using an existing key (originally generated for GitHub) succeeded on the first try, straight to a shell prompt, no password requested. This confirms the full chain works end to end: key pasted into the template's Cloud-Init field → Regenerate Image → clone → first boot → cloud-init writes the key into `authorized_keys` → key-based login succeeds.

Do this same check from each device/environment (Windows native, WSL, etc.) that should have access, since each one may use its own key file and SSH client configuration.

## Open items / future improvements

- **Key rotation / SSH certificates**: idea to build a script or switch to certificate-based auth (private CA, `ssh-keygen -s`) so access can be rotated or revoked per device without editing `authorized_keys` everywhere by hand. Logged in Todoist under Homelab Setup List for later.
