# VirtualBox on Linux Mint (kernel 7.0) — Setup Notes

**Host:** Linux Mint (Ubuntu noble base)  
**Verified:** 2026-09-15  
**Running kernel at verification:** `7.0.0-31-generic`  
**VirtualBox:** `7.0.16_Ubuntu r162802` (packages `7.0.16-dfsg-2ubuntu1.3`)

## Final verified state

| Component | Status |
|-----------|--------|
| `virtualbox` | installed (`ii`) |
| `virtualbox-dkms` | installed (`ii`) |
| `virtualbox-qt` | installed (`ii`) |
| DKMS `virtualbox/7.0.16` for `7.0.0-31-generic` | installed |
| DKMS `virtualbox/7.0.16` for `6.14.0-37-generic` | installed |
| Modules loaded | `vboxdrv`, `vboxnetflt`, `vboxnetadp` |
| CLI | `VBoxManage` works |
| GUI | `VirtualBox` / `virtualbox` launches |
| Headless VM start on kernel 7.0 | verified (test VMs removed) |

## Install packages

```bash
sudo apt update
sudo apt install virtualbox virtualbox-dkms virtualbox-qt
```

## Problem: DKMS build failure on kernel 7.0

Stock `virtualbox-dkms` 7.0.16 fails to build against Linux **7.0** with errors such as:

- `module vboxdrv uses symbol kvm_enable_virtualization from namespace module:kvm-amd,kvm-intel, but does not import it`
- Similar errors for `kvm_disable_virtualization`, `cr4_update_irqsoff`, `cr4_read_shadow`

**Cause:** Kernel 7.0 exports those helpers only for in-tree `kvm*` modules (`EXPORT_SYMBOL` with `module:` namespaces). Explicit `MODULE_IMPORT_NS("module:…")` is **not allowed**. The fallback CR4 path also uses `cpu_tlbstate.cr4`, which is not usable from this out-of-tree module on 7.0.

Failed DKMS autoinstall also blocked `dpkg --configure` for:

- `linux-headers-7.0.0-31-generic`
- `linux-image-7.0.0-31-generic`
- `linux-headers-generic-hwe-24.04`
- `linux-generic-hwe-24.04`

## Fix: compatibility patch

**File:** `/usr/src/virtualbox-7.0.16/vboxdrv/linux/SUPDrv-linux.c`

### 1. Limit KVM VMX helper API to pre-7.0

Change the gate from `RTLNX_VER_MIN(6,16,0)` to a range excluding 7.0+:

```c
#if (RTLNX_VER_RANGE(6,16,0, 7,0,0)) && defined(CONFIG_KVM_GENERIC_HARDWARE_ENABLING) && defined(VBOX_WITH_HOST_VMX)
```

(around line 134)

### 2. Limit `cr4_read_shadow` / `cr4_update_irqsoff` to pre-7.0

In `supdrvOSChangeCR4`:

```c
#if RTLNX_VER_RANGE(5,8,0, 7,0,0) /* 7.0+ restricts cr4_* symbols to kvm modules */
```

(around line 1005)

### 3. Limit `cpu_tlbstate.cr4` path to pre-7.0

In the `#else` branch of `supdrvOSChangeCR4`, both places that used:

```c
# if RTLNX_VER_MIN(3,20,0)
```

become:

```c
# if RTLNX_VER_RANGE(3,20,0, 7,0,0) /* 7.0+: cpu_tlbstate.cr4 not usable from modules */
```

(around lines 1014 and 1022)

On kernel 7.0+, VirtualBox then uses `ASMGetCR4` / `ASMSetCR4` instead.

## Rebuild DKMS and finish dpkg

```bash
sudo rm -rf /var/lib/dkms/virtualbox/7.0.16/build
sudo dkms build -m virtualbox -v 7.0.16 -k $(uname -r)
sudo dkms install -m virtualbox -v 7.0.16 -k $(uname -r)

# If multiple kernels are installed, build for each as needed, e.g.:
# sudo dkms build -m virtualbox -v 7.0.16 -k 7.0.0-31-generic
# sudo dkms install -m virtualbox -v 7.0.16 -k 7.0.0-31-generic

sudo dpkg --configure -a
dkms status
```

Confirm modules:

```bash
lsmod | grep vbox
modinfo vboxdrv | grep -E 'filename|version|vermagic'
sudo dmesg | grep -i vbox | tail
```

Expected dmesg line:

```text
vboxdrv: Successfully loaded version 7.0.16_Ubuntu r162802
```

## KVM vs VirtualBox (VT-x conflict)

Only one hypervisor can own VT-x. If a VM fails with:

```text
VERR_VMX_IN_VMX_ROOT_MODE
```

unload KVM first:

```bash
sudo modprobe -r kvm_intel kvm
# AMD hosts: sudo modprobe -r kvm_amd kvm
```

Then start the VM again.

To prefer VirtualBox after boot, blacklist KVM (optional):

```bash
echo -e 'blacklist kvm_intel\nblacklist kvm' | sudo tee /etc/modprobe.d/blacklist-kvm-for-vbox.conf
# Rebuild initramfs if you need this very early: sudo update-initramfs -u
```

## Launch VirtualBox

```bash
VirtualBox          # GUI manager
# or
virtualbox

VBoxManage --version
VBoxManage list vms
```

## Create a simple test VM (optional)

```bash
VM=my-test-vm
VBoxManage createvm --name "$VM" --ostype Linux_64 --register
VBoxManage modifyvm "$VM" --memory 512 --cpus 1 --vram 16 --nic1 nat --audio-driver none
VBoxManage storagectl "$VM" --name SATA --add sata --controller IntelAhci
VBoxManage createmedium disk --filename "$HOME/VirtualBox VMs/$VM/$VM.vdi" --size 10240 --format VDI
VBoxManage storageattach "$VM" --storagectl SATA --port 0 --device 0 --type hdd \
  --medium "$HOME/VirtualBox VMs/$VM/$VM.vdi"

# Attach an installer ISO when ready:
# VBoxManage storageattach "$VM" --storagectl SATA --port 1 --device 0 --type dvddrive --medium /path/to.iso

VBoxManage startvm "$VM" --type gui
# or headless:
# VBoxManage startvm "$VM" --type headless
```

Remove a test VM:

```bash
VBoxManage controlvm "$VM" poweroff 2>/dev/null || true
VBoxManage unregistervm "$VM" --delete
```

## Maintenance warnings

1. **Patch is overwritten on package reinstall/upgrade** of `virtualbox-dkms`. After `apt install --reinstall virtualbox-dkms` or a newer package pull that refreshes `/usr/src/virtualbox-7.0.16`, re-apply the patch and rebuild DKMS if kernel 7.0 builds fail again.
2. **Long-term fix:** upgrade to a VirtualBox release with official Linux 7.0 support (Oracle builds or a newer distro package when available).
3. **New kernels:** DKMS should auto-build on header install; on failure inspect:

   ```bash
   sudo cat /var/lib/dkms/virtualbox/7.0.16/build/make.log
   ```

4. **Secure Boot:** modules are signed with the local MOK during DKMS build. If load fails after reboot, enroll the MOK key or adjust Secure Boot policy. A “tainting kernel” / signature message in dmesg can still appear even when modules load successfully.
5. **Do not leave half-configured kernel packages:** if DKMS fails during kernel install, fix VirtualBox DKMS (or temporarily remove `virtualbox-dkms`) then run `sudo dpkg --configure -a`.

## Quick health check

```bash
uname -r
dkms status
lsmod | grep vbox
VBoxManage --version
dpkg --audit
sudo dmesg | grep -i vbox | tail
```

## Cleanup performed

Temporary test VMs (`vbox-dkms-test`, `vb-install-test`) were powered off and deleted. No guest VMs remain from this setup work. VirtualBox packages, DKMS modules, and the kernel 7.0 source patch remain installed.

## Related: Warp terminal font size

To increase Warp font size on Linux:

- Shortcuts: `Ctrl+=` (increase), `Ctrl+-` (decrease), `Ctrl+0` (reset)
- Settings file: `~/.config/warp-terminal/settings.toml`

```toml
[appearance]
font_size = 18
```

A snapshot of the appearance section is kept in `warp-appearance-settings.toml` in this repository for reference. Warp hot-reloads `settings.toml` when it changes.
