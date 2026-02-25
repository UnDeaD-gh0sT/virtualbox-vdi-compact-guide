# Compacting VirtualBox .VDI Files for Linux VMs by Zeroing Free Space

This guide provides a step-by-step process to reduce the physical file size of a dynamically expanding .VDI disk image in VirtualBox for Linux virtual machines. Over time, deleted files and changes can cause the .VDI file to grow larger than the actual used space inside the VM, even though the virtual size remains fixed. By filling unused space with zeros and compacting the image, you can reclaim host storage.

This method uses a simple `dd` approach inside the VM to zero free space, which is effective and does not require booting into a read-only mode (unlike tools like `zerofree`). It works for most Linux distributions (e.g., Kali, Ubuntu) with ext4 filesystems. Tested on Kali Linux 2025.2, reducing a ~57 GB .VDI to ~36 GB when internal usage was ~32 GB.

## Prerequisites
- VirtualBox installed on the host (Windows, Linux, or macOS).
- The Linux VM must be powered off before compacting.
- Administrative access on the host (e.g., run commands as admin on Windows).
- Sufficient free space inside the VM (at least as much as the free space you want to zero—temporary file will fill it).
- Backup the entire VM folder first (copy to another location) to avoid data loss.
- Confirm the VM's root filesystem is ext4 (run `df -T` inside the VM).
- Note the full path to your .VDI file (e.g., `C:\Path\To\YourVM.vdi` on Windows).

## Step-by-Step Instructions

### Step 1: Prepare Inside the VM
1. Start the VM and log in as a user with sudo privileges.
2. Open a terminal.
3. Update packages if needed (optional but recommended):
   ```
   sudo apt update
   ```
   (For non-Debian-based distros, use the equivalent like `sudo dnf update` or `sudo pacman -Syu`.)
4. Install `pv` for progress monitoring (optional; skip if not available):
   ```
   sudo apt install pv
   ```
5. Fill all free space on the root filesystem with zeros by creating a temporary file:
   ```
   sudo dd if=/dev/zero of=/bigzero bs=1M status=progress || true
   ```
   - This command reads zeros from `/dev/zero` and writes to `/bigzero` until the disk is full.
   - It will error out when space is exhausted (normal; the `|| true` ignores the error).
   - Time: 10–60 minutes depending on free space and disk speed.
6. Sync changes to disk:
   ```
   sudo sync
   ```
7. Delete the temporary file:
   ```
   sudo rm -f /bigzero
   ```
8. Sync again:
   ```
   sudo sync
   ```
9. Shut down the VM cleanly:
   ```
   sudo shutdown -h now
   ```

### Step 2: Compact the .VDI on the Host
1. Ensure the VM is fully powered off in VirtualBox.
2. Open a command prompt/terminal on the host with administrative privileges:
   - On Windows: Search for "cmd", right-click, and select "Run as administrator".
   - On Linux/macOS: Use `sudo` where needed.
3. Navigate to the VirtualBox installation directory (if not in PATH):
   - Windows example:
     ```
     cd "C:\Program Files\Oracle\VirtualBox"
     ```
   - Linux/macOS: VirtualBox tools are usually in PATH; if not, find via `which VBoxManage`.
4. Run the compact command, replacing the path with your .VDI's full location:
   ```
   VBoxManage modifymedium "C:\Path\To\YourVM.vdi" --compact
   ```
   - Progress shows as percentages (e.g., `0%...10%...`).
   - Time: 10–60+ minutes; do not interrupt.
   - If access denied: Ensure admin mode and VM is off; check file permissions.
5. Verify the new file size:
   - On Windows: Right-click the .VDI → Properties.
   - On Linux/macOS: `ls -lh /path/to/YourVM.vdi`.

## Expected Results
- The .VDI actual size should shrink close to the VM's internal used space (e.g., from 57 GB to ~36 GB if used space is ~32 GB), plus some overhead.
- The virtual size (e.g., 80 GB) remains unchanged.
- Internal VM storage usage is unaffected.

## Alternative: Using Zerofree (More Thorough but Complex)
If the `dd` method does not shrink enough, use `zerofree` for better results. This requires booting the VM into a read-only root shell via GRUB edits:
1. Install `zerofree` inside the VM: `sudo apt install zerofree`.
2. Shut down the VM.
3. Boot and access GRUB (hold Shift or spam Esc during early boot).
4. Edit the boot entry: Add `init=/bin/sh` to the linux line → Boot.
5. In the shell: `mount -o remount,ro /` → `zerofree -v /dev/sda1` (replace with your root partition from `lsblk`).
6. Reboot: `sync` → `reboot -f`.
7. Compact as in Step 2 above.

Note: GRUB access can be tricky; edit `/etc/default/grub` to set `GRUB_TIMEOUT_STYLE=menu` and `GRUB_TIMEOUT=10`, then `sudo update-grub` for easier future access.

## Troubleshooting
- **Compact fails with access denied**: Run as admin; close VirtualBox; check if .VDI is locked.
- **No shrinkage**: Ensure free space was zeroed properly; run `sudo fstrim -av` before `dd` if supported.
- **Slow process**: Use faster host drives; avoid running other tasks.
- **Non-ext4 filesystems**: Zerofree works only on ext2/3/4; for others (e.g., XFS), use filesystem-specific tools like `xfs_fsr`.
- **Errors in dd**: Normal if it says "No space left"; proceed to delete.

