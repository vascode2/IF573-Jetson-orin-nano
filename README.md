# IF573 Integration on Jetson Orin Nano (JetPack 6.2.1 / L4T 36.4.4)

This document captures a **known-working** integration flow for bringing up an Ezurio IF573 radio on a Jetson Orin Nano running JetPack 6.2.1.

## Scope and baseline

- Target: Jetson Orin Nano Dev Kit / Super
- OS baseline: JetPack 6.2.1 (L4T 36.4.4)
- Kernel observed: `5.15.148-tegra`
- Integration style: **on-target** (build/install on Jetson itself)
- Ezurio release used: `LRD-REL-13.98.0.12`
- Package names used:
  - `summit-if573-pcie-firmware-13.98.0.12.tar.bz2`
  - `summit-backports-13.98.0.12.tar.bz2`

## Prerequisites

1. Jetson is already flashed and boots normally on JP6.2.1.
2. IF573 module is installed in the correct M.2 Key-E path (PCIe).
3. Network access is available (Ethernet recommended during setup).
4. You can run `sudo`.

## If your board is not flashed yet (JP6.2.1 baseline prep)

If your board is not already on JP6.2.1/L4T 36.4.4, flash it first using NVIDIA's official flow:

- Jetson Orin Nano getting started guide: https://developer.nvidia.com/embedded/learn/get-started-jetson-orin-nano-devkit#setup
- SDK Manager download and usage: https://developer.nvidia.com/sdk-manager

Then return to this guide and continue from **Section 1**.

## 1) Verify Jetson software baseline

```bash
cat /etc/nv_tegra_release
uname -r
```

Expected baseline for this guide:

- `R36 (release), REVISION: 4.4`
- `5.15.148-tegra`

## 2) Install build dependencies

```bash
sudo apt update
sudo apt install -y build-essential bc flex bison libssl-dev libelf-dev kmod pkg-config git rsync
sudo apt install -y nvidia-l4t-kernel-headers
```

> Note: `linux-headers-$(uname -r)` may not exist as a package name on Jetson; this is normal.

Confirm headers are usable:

```bash
ls -ld /lib/modules/$(uname -r)/build
```

## 3) Copy and extract IF573 release packages

```bash
mkdir -p ~/if573
cd ~/if573
# copy files here first (scp or browser download)

tar xvf summit-if573-pcie-firmware-13.98.0.12.tar.bz2
tar xvf summit-backports-13.98.0.12.tar.bz2
```

## 4) Install IF573 firmware files

```bash
cd ~/if573
sudo tar xvf summit-if573-pcie-firmware-13.98.0.12.tar.bz2 -C /
```

`-C /` means extract into filesystem root so firmware lands under `/lib/firmware/...`.

Validate:

```bash
ls -al /lib/firmware/cypress/ | grep -E '55572|IF573|CYW55560A1' || true
```

## 5) Build backports (Wi-Fi-first path: no backported Bluetooth)

If base kernel has `CONFIG_BT=y`, full `lwb` backports config may fail to build. For this known-working flow, use `lwb_nbt`.

Check current kernel config:

```bash
zgrep -E 'CONFIG_BT=|CONFIG_CFG80211=' /proc/config.gz
```

Observed working baseline:

- `CONFIG_BT=y`
- `CONFIG_CFG80211=m`

Build:

```bash
cd ~/if573/summit-backports-13.98.0.12
export KLIB_BUILD=/lib/modules/$(uname -r)/build
export KLIB=/lib/modules/$(uname -r)

make clean
make defconfig-lwb_nbt
make -j"$(nproc)"
```

## 6) Install backports modules

```bash
sudo mkdir -p /lib/modules/$(uname -r)/updates
find . -name '*.ko' -exec sudo cp -v {} /lib/modules/$(uname -r)/updates/ \;
sudo depmod -a
```

Quick validation:

```bash
modinfo brcmfmac | head -n 20
find /lib/modules/$(uname -r)/updates -name 'brcmfmac*.ko' -o -name 'cfg80211*.ko'
```

## 7) Set regulatory domain

Use correct country code for your deployment (example below uses US):

```bash
echo 'options brcmfmac regdomain="US"' | sudo tee /etc/modprobe.d/brcmfmac.conf
sudo depmod -a
```

## 8) Reboot and verify IF573 bring-up

```bash
sudo reboot
```

After reboot:

```bash
iw dev
nmcli dev status
nmcli dev wifi list
sudo dmesg | egrep -i 'brcmfmac|cfg80211|firmware|cyfmac|pcie|wl'
```

## Expected success indicators

- `lspci` shows IF573 endpoint (example seen: `12be:bd31`).
- `brcmfmac` loads from backports v13.98.0.12.
- Firmware loads for `BCM55560/2`.
- Wi-Fi interface appears (example seen: `wlP1p1s0`).
- `nmcli dev wifi list` returns nearby APs.

## Example successful boot log

A trimmed example from a successful IF573 bring-up is provided at:

- `examples/dmesg_if573_success.log`

Use it as a reference when comparing your own `dmesg` output.

## Known warning that can still be acceptable

Messages like these were observed while still achieving working Wi-Fi:

- missing board-specific firmware filename fallback
- missing `txcap_blob` fallback

Driver fell back to generic IF573 firmware/NVRAM and interface came up successfully.

## Hardware troubleshooting quick guide

If IF573 does **not** appear:

1. Check PCIe enumeration:

   ```bash
   lspci -nn
   lspci -nn | egrep -i 'network|wireless|broadcom|14e4|cypress|infineon|12be'
   ```

2. Check for link failures:

   ```bash
   sudo dmesg | egrep -i 'pcie|phy link never came up'
   ```

3. Reseat module and verify correct M.2 Key-E slot/path.
4. Power off fully, then cold boot.
5. If available, A/B test with a known-good radio in same slot.

## Bluetooth note

This guide uses `defconfig-lwb_nbt` to avoid backported Bluetooth build conflict when base kernel has `CONFIG_BT=y`.

- Wi-Fi path is validated by this guide.
- For full IF573 Bluetooth via backports, use a kernel configuration where Bluetooth is not built-in (`CONFIG_BT` as module or disabled), then rebuild kernel and use full `defconfig-lwb` flow.

---

## Command recap (copy/paste)

```bash
# baseline
cat /etc/nv_tegra_release
uname -r

# deps
sudo apt update
sudo apt install -y build-essential bc flex bison libssl-dev libelf-dev kmod pkg-config git rsync
sudo apt install -y nvidia-l4t-kernel-headers

# workspace
mkdir -p ~/if573
cd ~/if573

# install firmware
sudo tar xvf summit-if573-pcie-firmware-13.98.0.12.tar.bz2 -C /

# build+install backports (wifi-first)
tar xvf summit-backports-13.98.0.12.tar.bz2
cd ~/if573/summit-backports-13.98.0.12
export KLIB_BUILD=/lib/modules/$(uname -r)/build
export KLIB=/lib/modules/$(uname -r)
make clean
make defconfig-lwb_nbt
make -j"$(nproc)"
sudo mkdir -p /lib/modules/$(uname -r)/updates
find . -name '*.ko' -exec sudo cp -v {} /lib/modules/$(uname -r)/updates/ \;
sudo depmod -a

# regdomain
echo 'options brcmfmac regdomain="US"' | sudo tee /etc/modprobe.d/brcmfmac.conf
sudo depmod -a

# reboot + verify
sudo reboot
```

## References

- NVIDIA Jetson Orin Nano Dev Kit getting started: https://developer.nvidia.com/embedded/learn/get-started-jetson-orin-nano-devkit#setup
- NVIDIA SDK Manager: https://developer.nvidia.com/sdk-manager
- Ezurio Sona IF / Sterling LWBx integration guide (general): https://www.ezurio.com/documentation/sona-if-and-sterling-lwbx-software-integration-guide
- Ezurio release package used in this guide: https://github.com/Ezurio/Connectivity_Stack_Release_Packages/releases/tag/LRD-REL-13.98.0.12

