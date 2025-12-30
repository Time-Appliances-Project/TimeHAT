# GNSS Time Stack (u-blox + gpsd + ts2phc)

This setup lets:

- **gpsd** fully control a u-blox receiver on `/dev/ttyAMA0`
- **gpspipe + socat** expose NMEA from gpsd on a virtual TTY
- **ts2phc** consume that NMEA plus 1PPS on `eth1` to discipline the NIC’s PHC

It’s designed for the TimeHAT + GPS reciever like Neo-M9N or ZED-F9T on a Raspberry Pi, but the pattern is generic.

Why? Ts2phc doesn't seem to like natively talking to any serial port, at least that's what I found. This method isolates ts2phc from the serial port, letting it consume NMEA messages to work properly.

---

## Architecture

```text
GPS receiver (ex. Neo-M9N) (UART @ 38400)
/dev/ttyAMA0
     |
     v
   gpsd
     |
     |  (GPSD protocol)
     v
 gpspipe -r
     |
     |  (NMEA stream)
     v
  socat  →  /dev/gps-ts2phc (PTY)
     |
     v
  ts2phc  +  PPS on eth1  →  eth1 PHC disciplined to GNSS time


## Installation

These steps set up a GNSS time stack where:

- `gpsd` talks directly to your GNSS receiver on `/dev/ttyAMA0`
- `gpspipe + socat` expose NMEA from `gpsd` on `/dev/gps-ts2phc`
- `ts2phc` reads NMEA from `/dev/gps-ts2phc` and disciplines the PHC on `eth1` using PPS

Adjust device names or paths as needed for your system.
```
---


## Installation

# 1) Install dependencies
```bash
#    - gpsd / gpsd-clients: GNSS daemon + tools
#    - socat: creates the PTY that ts2phc reads from
#    - linuxptp: provides ts2phc
sudo apt update
sudo apt install gpsd gpsd-clients socat linuxptp
```

# 2) Configure gpsd
```bash
#    This config points gpsd at the GNSS serial port and sets basic options
#    (for example, using /dev/ttyAMA0 and -n so it starts polling immediately).
#    Edit /etc/default/gpsd later if your device or options differ.
sudo wget -O /etc/default/gpsd \
  https://raw.githubusercontent.com/Time-Appliances-Project/Products/refs/heads/main/TimeHAT/GPS/gpsd
```

# 3) Install gpspipe + socat bridge service
```bash
#    This systemd service:
#      - runs `gpspipe -r` against gpsd
#      - uses socat to create /dev/gps-ts2phc
#      - feeds NMEA from gpsd into /dev/gps-ts2phc for ts2phc to consume
sudo wget -O /etc/systemd/system/gpspipe-socat.service \
  https://raw.githubusercontent.com/Time-Appliances-Project/Products/refs/heads/main/TimeHAT/GPS/gpspipe-socat.service
```

# 4) Install ts2phc service
```bash
#    This systemd service:
#      - runs ts2phc in NMEA mode
#      - reads NMEA from /dev/gps-ts2phc
#      - uses PPS on eth1 to discipline the NIC’s PHC
sudo wget -O /etc/systemd/system/ts2phc-gps.service \
  https://raw.githubusercontent.com/Time-Appliances-Project/Products/refs/heads/main/TimeHAT/GPS/ts2phc-gps.service
```

# 5) Install ts2phc configuration
```bash
#    The config tells ts2phc:
#      - where to read NMEA (/dev/gps-ts2phc)
#      - which interface has PPS (eth1)
#      - where the leap-seconds file is located
sudo mkdir -p /etc/linuxptp

sudo wget -O /etc/linuxptp/ts2phc-gps.cfg \
  https://raw.githubusercontent.com/Time-Appliances-Project/Products/refs/heads/main/TimeHAT/GPS/ts2phc-gps.cfg
```

# 6) Reload systemd units
```bash
#    Let systemd discover the new service files.
sudo systemctl daemon-reload
```

# 7) Enable services (run at boot)
```bash
#    Enable all components so they start automatically on boot:
#      - gpsd: talks to the GNSS receiver
#      - gpspipe-socat: exposes NMEA on /dev/gps-ts2phc
#      - ts2phc-gps: disciplines eth1 using NMEA + PPS
sudo systemctl enable gpsd
sudo systemctl enable gpspipe-socat.service
sudo systemctl enable ts2phc-gps.service
```

# 8) Start services now (without reboot)
```bash
sudo systemctl start gpsd
sudo systemctl start gpspipe-socat.service
sudo systemctl start ts2phc-gps.service
```

# 9) Stop and disable if you want 
```bash
sudo systemctl disable gpsd
sudo systemctl disable gpspipe-socat.service
sudo systemctl disable ts2phc-gps.service
sudo systemctl stop gpsd
sudo systemctl stop gpspipe-socat.service
sudo systemctl stop ts2phc-gps.service
```


# 10) Quick verification

# 10a) gpsd talking to the receiver (position / satellites should show up)
```bash
cgps
```

# 10b) ts2phc disciplining eth1 (watch offsets / status)
```bash
sudo journalctl -u ts2phc-gps.service -f
```
