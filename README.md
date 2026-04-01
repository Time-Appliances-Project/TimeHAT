# TimeHAT

![2025-10-01T20_09_28 539Z-thumbnail_IMG_3502](https://github.com/user-attachments/assets/57f644cd-3265-4f4c-bcdc-205dd5fd6c7b)

The Raspberry Pi 5 PCIe HAT is designed to bring precision timing and advanced network functionality to the Raspberry Pi 5 platform. Leveraging the PCIe interface via an FPC connection, this HAT expands the Pi's capabilities by integrating a high-performance I226 Ethernet NIC, precise timing inputs and outputs via SMA connectors, and a GNSS module slot for global time synchronization. It's a powerful tool for building small form-factor PTP (Precision Time Protocol) clients, ideal for time-sensitive networking applications.

## Order

Order from the Tindie Page:
https://www.tindie.com/products/timeappliances/timehat-i226-nic-with-pps-inout-for-rpi5/

---

# Intel I226 PPS Fix — Patched `igc` Driver (DKMS)

The stock Linux `igc` driver has limitations that prevent proper PPS (Pulse Per Second) input and output on the I226 NIC. This patched driver fixes those issues and is packaged as a DKMS module for easy installation.

Choose the correct zip for your kernel version (`uname -r`):

| Kernel | Zip File |
|--------|----------|
| 6.6.x (RPi5 stable as of 4/2025) | `intel-igc-ppsfix_rpi5_6_6.zip` |
| 6.12.x (RPi5 previous, Intel OOT driver based) | `intel-igc-ppsfix_rpi5_6_12.zip` |
| 6.12.62 (RPi5 latest as of 3/9/2026, upstream source based) | `intel-igc-ppsfix_rpi5_6.12.62.zip` |

## What the Patch Fixes

### 1. PPS Input (EXTTS) — Rising Edge Filtering

**Problem:** The I226 hardware triggers timestamps on both rising and falling edges. The stock driver requires `PTP_STRICT_FLAGS` with both edges enabled, making single-edge filtering impossible. GPS PPS signals are rising-edge only.

**Fix:** Bypasses the strict dual-edge requirement and forces rising-edge-only mode when enabling EXTTS. A GPIO-level workaround in the interrupt handler reads the actual pin state after each edge event and filters out the wrong edge, since the hardware doesn't natively support single-edge timestamping.

### 2. PPS Output (PEROUT) — 1PPS Alignment

**Problem:** When requesting 1PPS output via `testptp -p 1000000000`, the kernel computes `period/2 = 500000000 ns`. The stock driver treats this as a "use frequency mode" case, producing a free-running 1Hz square wave that is **not aligned to the PTP clock's second boundary**.

**Fix:** Removes `500000000` from the frequency-mode condition, forcing 1PPS to use Target Time (TT) mode instead. In TT mode, the output toggles at each target time and re-arms in the interrupt handler, producing PPS pulses aligned to the PTP clock's top of second.

### 3. GPIO Edge Check Workaround

**Problem:** The I226's internal GPIO read has propagation delays that can cause the pin value to be read incorrectly immediately after an edge interrupt.

**Fix:** Adds a configurable microsecond delay before reading the GPIO pin state (`edge_check_delay_us`, default 20µs). Also adds an invertible check (`edge_check_invert`, default 0) since some I226 chips report the opposite GPIO level. Both parameters are tunable at runtime via sysfs without rebuilding the module.

### 4. Per-Channel Pin & Flags Tracking

**Problem:** The stock `igc_adapter` struct doesn't track which pins and flags are assigned to each EXTTS channel.

**Fix:** Adds `ts0_pin`, `ts1_pin`, `ts0_flags`, `ts1_flags` fields to the adapter struct, used by the GPIO edge filtering logic.

---

## 🧰 Prerequisites

Install dependencies:

```bash
sudo apt install -y net-tools gcc vim dkms linuxptp raspberrypi-kernel-headers
```

Install `testptp` (used for testing PPS):

```bash
cd ~ && mkdir -p testptp && cd testptp
wget https://raw.githubusercontent.com/torvalds/linux/refs/heads/master/tools/testing/selftests/ptp/testptp.c
wget https://raw.githubusercontent.com/torvalds/linux/refs/heads/master/include/uapi/linux/ptp_clock.h
sudo cp ptp_clock.h /usr/include/linux/ptp_clock.h
gcc -Wall -lrt testptp.c -o testptp
sudo cp testptp /usr/bin/
```

---

## 📦 First-Time Install

1. **Download and unzip the correct patch for your kernel:**

```bash
cd ~
# For kernel 6.12.62 (RPi5 latest as of 3/9/2026):
wget https://github.com/Time-Appliances-Project/TimeHAT/raw/refs/heads/main/intel-igc-ppsfix_rpi5_6.12.62.zip
unzip intel-igc-ppsfix_rpi5_6.12.62.zip -d intel-igc-ppsfix_6_12
cd intel-igc-ppsfix_6_12
```

2. **Register, build, and install the DKMS module:**

```bash
sudo dkms add .
sudo dkms build --force igc -v 6.12.0-ppsfix.1
sudo dkms install --force igc -v 6.12.0-ppsfix.1
```

3. **Back up the stock kernel module and replace it:**

```bash
sudo cp /lib/modules/$(uname -r)/kernel/drivers/net/ethernet/intel/igc/igc.ko.xz \
        /lib/modules/$(uname -r)/kernel/drivers/net/ethernet/intel/igc/igc.ko.xz.bak
sudo cp /lib/modules/$(uname -r)/updates/dkms/igc.ko.xz \
        /lib/modules/$(uname -r)/kernel/drivers/net/ethernet/intel/igc/igc.ko.xz
```

4. **Update module cache and initramfs:**

```bash
sudo depmod -a
sudo update-initramfs -u -k $(uname -r)
```

5. **Reboot:**

```bash
sudo reboot
```

---

## 🔄 Rebuilding After Code Changes

If you edit the source files and need to rebuild:

```bash
# Copy changed source to DKMS tree
sudo cp ~/intel-igc-ppsfix_6_12/src/igc_main.c /usr/src/igc-6.12.0-ppsfix.1/src/igc_main.c

# Rebuild and reinstall
sudo dkms build --force igc -v 6.12.0-ppsfix.1
sudo dkms install --force igc -v 6.12.0-ppsfix.1

# Reload without rebooting
sudo rmmod igc && sudo modprobe igc
```

---

## ✅ Verify Installation

```bash
# Confirm DKMS module is loaded (should show /updates/dkms/ path)
modinfo igc | grep filename

# Check driver info
ethtool -i enp1s0
```

---

## 🧪 Testing PPS

### PPS Output + Input Loopback Test

Connect an SMA loopback cable between the two SMA connectors, then:

```bash
# Configure pin 0 as PPS output (perout), pin 1 as PPS input (extts)
sudo testptp -d /dev/ptp0 -L 0,2
sudo testptp -d /dev/ptp0 -L 1,1

# Start 1PPS output
sudo testptp -d /dev/ptp0 -p 1000000000

# Read 5 external timestamps (should show ~1.000000000s intervals)
sudo testptp -d /dev/ptp0 -e 5
```

### GPS PPS Input Test

With a GPS PPS signal connected to an SMA input:

```bash
# Configure pin 2 as extts input
sudo testptp -d /dev/ptp0 -L 2,1

# Read timestamps (should show 1 event per second)
sudo testptp -d /dev/ptp0 -e 5
```

### Disciplining PTP Clock to GPS

Use `ts2phc` from the `linuxptp` package:

```bash
sudo ts2phc -c /dev/ptp0 -s generic --ts2phc.pin_index 2 -m -l 7
```

---

## ⚙️ Runtime Module Parameters

These can be changed live without rebuilding:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `edge_check_delay_us` | 20 | Microsecond delay before GPIO read for edge filtering |
| `edge_check_invert` | 0 | Invert GPIO level check (0=normal, 1=inverted) |

```bash
# View current values
cat /sys/module/igc/parameters/edge_check_delay_us
cat /sys/module/igc/parameters/edge_check_invert

# Change at runtime
echo 50 | sudo tee /sys/module/igc/parameters/edge_check_delay_us
echo 1 | sudo tee /sys/module/igc/parameters/edge_check_invert
```

To make parameter overrides persistent across reboots:

```bash
echo "options igc edge_check_invert=1 edge_check_delay_us=20" | sudo tee /etc/modprobe.d/igc-ppsfix.conf
```

---

## 📝 Notes

- DKMS will **automatically rebuild** the module when the kernel is updated (`AUTOINSTALL=yes` in `dkms.conf`).
- After a kernel update, you may need to re-run the stock module replacement (step 3) and `update-initramfs`.
- The patched module is built from the **upstream Linux 6.12 igc source** with PPS fixes applied on top.

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
