# TimeHat

![2025-10-01T20_09_28 539Z-thumbnail_IMG_3502](https://github.com/user-attachments/assets/57f644cd-3265-4f4c-bcdc-205dd5fd6c7b)

The Raspberry Pi 5 PCIe HAT is designed to bring precision timing and advanced network functionality to the Raspberry Pi 5 platform. Leveraging the PCIe interface via an FPC connection, this HAT expands the Pi’s capabilities by integrating a high-performance I226 Ethernet NIC, precise timing inputs and outputs via SMA connectors, and a GNSS module slot for global time synchronization. It’s a powerful tool for building small form-factor PTP (Precision Time Protocol) clients, ideal for time-sensitive networking applications.

## Order

Order from the Tindie Page:
https://www.tindie.com/products/timeappliances/timehat-i226-nic-with-pps-inout-for-rpi5/

# Intel I225 PPS Input Patch (DKMS-based igc driver replacement)

This guide shows how to install a patched version of the `igc` driver (used by Intel I225/I226 NICs) to support PPS input. It uses DKMS for rebuilds and ensures the custom driver loads at boot on Raspberry Pi OS 64-bit.

---

## 🧰 Prerequisites

Install dependencies:

```bash
sudo apt install -y net-tools gcc vim dkms linuxptp raspberrypi-kernel-headers
cd ~ ; mkdir testptp; cd testptp
wget https://raw.githubusercontent.com/torvalds/linux/refs/heads/master/tools/testing/selftests/ptp/testptp.c
wget https://raw.githubusercontent.com/torvalds/linux/refs/heads/master/include/uapi/linux/ptp_clock.h
sudo cp ptp_clock.h /usr/include/linux/ptp_clock.h
gcc -Wall -lrt testptp.c -o testptp
sudo cp testptp /usr/bin/
```

---

## 📦 Install the Patched Driver

1. **Copy the zip to your machine and unzip it:**

Depending on your RPI5 kernel, use 6_6 file for 6.6 (current stable RPI release as of 4/15/2025) or 6_12 file for 6.12 kernel. Use uname -a to get kernel version.
```bash
cd ~
wget https://github.com/Time-Appliances-Project/TimeHAT/raw/refs/heads/main/intel-igc-ppsfix_rpi5_6_6.zip
unzip intel-igc-ppsfix_rpi5_6_6.zip
cd intel-igc-ppsfix
```

2. **Build and install the patched `igc` module using DKMS:**

> Replace `5.4.0-7642.46` with the correct version if different.

```bash
sudo dkms remove igc -v 5.4.0-7642.46
sudo dkms add .
sudo dkms build --force igc -v 5.4.0-7642.46
sudo dkms install --force igc -v 5.4.0-7642.46
```

3. **Backup the current in-kernel `igc` module:**

```bash
sudo cp /lib/modules/$(uname -r)/kernel/drivers/net/ethernet/intel/igc/igc.ko.xz \
        /lib/modules/$(uname -r)/kernel/drivers/net/ethernet/intel/igc/igc.ko.xz.bak
```

4. **Override it with the DKMS-built module:**

```bash
sudo cp /lib/modules/$(uname -r)/updates/dkms/igc.ko.xz \
        /lib/modules/$(uname -r)/kernel/drivers/net/ethernet/intel/igc/igc.ko.xz
```

---

## 🔄 Update Kernel Module Cache and build image

```bash
sudo depmod -a
sudo update-initramfs -u -k $(uname -r)
```

---

## 🔁 Reboot

```bash
sudo reboot
```

---

## ✅ Confirm it's Working

After reboot, confirm the driver in use is your custom one:

```bash
modinfo igc | grep filename
ethtool -i <your-interface>  # e.g., enp1s0
```

Can test with testptp , use loopback cable between the two SMAs
```bash

sudo testptp -d /dev/ptp0 -L0,2
sudo testptp -d /dev/ptp0 -p 1000000000
sudo testptp -d /dev/ptp0 -L1,1

sudo testptp -d /dev/ptp0 -e 5
```

---

## 📝 Notes

- This method **overrides the kernel’s default igc module** at boot.
- To persist across kernel updates, repeat steps 2–4 and run `sudo update-initramfs -u -k $(uname -r)` after each kernel change.

---

## License

This project is licensed under the Creative Commons Attribution-NonCommercial 4.0 International License (CC BY-NC 4.0).

You are free to:

Share — copy and redistribute the material in any medium or format
Adapt — remix, transform, and build upon the material
Under the following terms:

Attribution — You must give appropriate credit, provide a link to the license, and indicate if changes were made.
NonCommercial — You may not use the material for commercial purposes.
For full details, see: https://creativecommons.org/licenses/by-nc/4.0/

As the project creator, I reserve the right to use this material commercially or under any other terms.
