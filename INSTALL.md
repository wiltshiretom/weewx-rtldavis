# Salisbury Weather Station — Fresh Install Guide

Complete steps to reproduce the Salisbury Weather Pi setup from scratch.

**Hardware**: Raspberry Pi (aarch64), Davis Vantage Pro ISS (EU 868 MHz),
RTL-SDR dongle (RTL2838), BMP280 pressure/temperature sensor on I²C.

**OS**: Debian GNU/Linux 13 (Trixie)

---

## 1. Install OS-level dependencies

```bash
sudo apt update
sudo apt install -y git golang rtl-sdr librtlsdr-dev
```

Verify the RTL-SDR dongle is seen:
```bash
lsusb | grep -i realtek
```

---

## 2. Install weewx 5

```bash
# Add weewx apt repo
wget -qO - https://weewx.com/keys.html | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/weewx.gpg
echo "deb [arch=all] https://weewx.com/apt/python3 buster main" | sudo tee /etc/apt/sources.list.d/weewx.list
sudo apt update
sudo apt install -y weewx

# Verify
weewx --version  # should show 5.x
```

---

## 3. Build and install rtldavis binary

```bash
# Clone the fork with EU scan-confirmed frequencies
git clone https://github.com/wiltshiretom/rtldavis ~/Code/rtldavis
cd ~/Code/rtldavis
go build -o rtldavis .
sudo cp rtldavis /usr/local/bin/rtldavis

# Test standalone (stop weewx first)
sudo systemctl stop weewx
rtldavis -ppm 11 -gain 300 -tf EU -tr 2
# Should see packets every ~2.5s from Transmitter:1
# Ctrl-C when confirmed
```

---

## 4. Install the rtldavis weewx driver

```bash
git clone https://github.com/wiltshiretom/weewx-rtldavis ~/Code/weewx-rtldavis
sudo cp ~/Code/weewx-rtldavis/bin/user/rtldavis.py /etc/weewx/bin/user/rtldavis.py
```

---

## 5. Install the BMP280 pressure sensor driver

```bash
sudo cp ~/Code/weewx-rtldavis/bin/user/bmp280.py /etc/weewx/bin/user/bmp280.py
```

Enable I²C if not already on:
```bash
sudo raspi-config nonint do_i2c 0
# Verify sensor is visible at address 0x77:
sudo i2cdetect -y 1
```

---

## 6. Install the Salisbury skin

```bash
git clone https://github.com/wiltshiretom/salisbury-weather ~/Code/salisbury-weather
cd ~/Code/salisbury-weather
./pi-install.sh setup
./pi-install.sh staging   # deploy WIP skin to staging
```

This sets up:
- `/etc/weewx/skins/Salisbury` — production skin
- `/etc/weewx/skins/SalisburyWIP` — staging skin
- `/var/www/html/weewx` — production HTML root
- `/var/www/html/weewx-wip` — staging HTML root
- Webcam cron entry (every 5 min, runs as tom)
- Forecast cron entry (every 30 min, runs as tom)

---

## 7. Configure weewx.conf

Edit `/etc/weewx/weewx.conf` and apply the following settings:

### [Station]
```ini
[Station]
    location = "Salisbury, UK"
    latitude = 51.09
    longitude = -1.80
    altitude = 94, meter
    station_type = Rtldavis
    station_url = https://salisburyweather.co.uk
    rain_year_start = 1
    week_start = 6
```

### [Rtldavis]
```ini
[Rtldavis]
    cmd = /usr/local/bin/rtldavis -ppm 11 -gain 300
    transceiver_frequency = EU
    iss_channel = 2          # transmitter ID of the ISS (bitmask: tr1=1, tr2=2)
    anemometer_channel = 0
    leaf_soil_channel = 0
    temp_hum_1_channel = 0
    temp_hum_2_channel = 0
    rain_bucket_type = 1     # 0 = 0.01 inch, 1 = 0.2 mm
    debug_parse = 0
    debug_rain = 0
    debug_rtld = 2           # 1=info; 2=+data+chan; 3=+packets
    save_pct_good_per_transmitter = False
    driver = user.rtldavis
```

### [BMP280]
```ini
[BMP280]
    address = 0x77
    bus = 1
```

### [StdReport] — skins and rsync targets
```ini
[StdReport]
    SKIN_ROOT = skins
    HTML_ROOT = /var/www/html/weewx
    data_binding = wx_binding

    [[SalisburyReport]]
        skin = Salisbury
        enable = true
        HTML_ROOT = /var/www/html/weewx

    [[SalisburyWipReport]]
        skin = SalisburyWIP
        enable = true
        HTML_ROOT = /var/www/html/weewx-wip

    [[RSYNC]]
        skin = Rsync
        enable = true
        server = hostinger
        path = domains/salisburyweather.co.uk/public_html
        delete = 0
        HTML_ROOT = /var/www/html/weewx

    [[RSYNC_WIP]]
        skin = Rsync
        enable = true
        server = hostinger
        path = domains/salisburyweather.co.uk/public_html/staging
        delete = 0
        HTML_ROOT = /var/www/html/weewx-wip
        rsync_options = --copy-links

    [[Defaults]]
        lang = en
        unit_system = metric
        [[[Units]]]
            [[[[Groups]]]]
                group_altitude = meter
                group_pressure = mbar
                group_rain = mm
                group_rainrate = mm_per_hour
                group_temperature = degree_C
```

### [StdConvert]
```ini
[StdConvert]
    target_unit = US    # internal storage in US units
```

### [StdQC]
```ini
[StdQC]
    [[MinMax]]
        barometer = 26, 32.5, inHg
        pressure = 24, 34.5, inHg
        outTemp = -40, 120, degree_F
        inTemp = 10, 120, degree_F
        outHumidity = 0, 100
        inHumidity = 0, 100
        windSpeed = 0, 120, mile_per_hour
        rain = 0, 10, inch
```

### [StdArchive]
```ini
[StdArchive]
    archive_interval = 300
    record_generation = hardware
    loop_hilo = True
    data_binding = wx_binding
```

### [Engine] — add BMP280 data service
```ini
[Engine]
    [[Services]]
        data_services = user.bmp280.BMP280Service,
```
(Leave all other service lists at their defaults — only `data_services` needs changing.)

---

## 8. Configure SSH access to Hostinger (rsync target)

Weewx rsyncs to Hostinger shared hosting. The alias `hostinger` must be in
`~/.ssh/config` for the `weewx` user:

```
Host hostinger
    HostName srv123.hostinger.com    # check your actual host
    User u123456789                  # your Hostinger SSH user
    IdentityFile ~/.ssh/id_hostinger
```

Copy the key to the `weewx` user:
```bash
sudo mkdir -p /var/lib/weewx/.ssh
sudo cp ~/.ssh/id_hostinger /var/lib/weewx/.ssh/
sudo cp ~/.ssh/config /var/lib/weewx/.ssh/
sudo chown -R weewx:weewx /var/lib/weewx/.ssh
sudo chmod 700 /var/lib/weewx/.ssh
sudo chmod 600 /var/lib/weewx/.ssh/*
```

---

## 9. Start weewx

```bash
sudo systemctl enable weewx
sudo systemctl start weewx

# Watch logs
sudo journalctl -u weewx -f
```

You should see LOOP packets arriving every ~2.5 seconds and archive records
every 5 minutes.

---

## 10. Verify everything

```bash
# Packets arriving?
sudo journalctl -u weewx -n 30

# Test rtldavis standalone:
sudo systemctl stop weewx
rtldavis -ppm 11 -gain 300 -tf EU -tr 2
sudo systemctl start weewx

# Test rsync manually:
sudo -u weewx rsync -n hostinger:domains/salisburyweather.co.uk/public_html/
```

---

## Key file locations

| What | Where |
|---|---|
| weewx config | `/etc/weewx/weewx.conf` |
| rtldavis binary | `/usr/local/bin/rtldavis` |
| weewx driver | `/etc/weewx/bin/user/rtldavis.py` |
| BMP280 driver | `/etc/weewx/bin/user/bmp280.py` |
| Salisbury skin | `/etc/weewx/skins/Salisbury/` |
| SalisburyWIP skin | `/etc/weewx/skins/SalisburyWIP/` |
| Production HTML | `/var/www/html/weewx/` |
| Staging HTML | `/var/www/html/weewx-wip/` |
| SQLite database | `/var/lib/weewx/weewx.sdb` |
| Webcam log | `/home/tom/webcam-capture.log` |
| Forecast log | `/home/tom/forecast.log` |
