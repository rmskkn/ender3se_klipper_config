# Klipper config

Printer config for a Creality Ender-3 V3 SE upgraded with a [300°C high-temperature hotend](https://www.amazon.com/Creality-Ender-V3-SE-Temperature/dp/B0D7GX6PCH/).

## Files

- `printer.cfg` — Klipper printer configuration.
- `moonraker.conf` — Moonraker API server configuration.

## Setup

Paths in these files use the placeholder `<YOUR_USER>` — replace with the actual Linux user Klipper/Moonraker run as before deploying:

- `printer.cfg`: `[virtual_sdcard] path`
- `moonraker.conf`: `[server] klippy_uds_address`

Also verify `[mcu] serial` matches the actual device on your machine (`ls /dev/serial/by-id/` for a stable path, or `/dev/ttyUSB0` if there's only one USB-serial device).

## Note on hotend upgrade

`printer.cfg` still has `[extruder] max_temp: 260`, left over from the stock hotend. Review and raise to match the 300°C-rated hotend's actual safe limit before relying on higher-temperature filaments.

## Shaper calibration

Needs ADXL345 (USB RP2040 board). Current `[input_shaper]` in `printer.cfg` copied from different printer — redo on actual hardware.
Board source: https://mellow-3d.github.io/fly_adxl345_usb_klipper_usb.html

```
# 1. compile firmware
cd ~/klipper
make clean KCONFIG_CONFIG=config.adxl345
make menuconfig KCONFIG_CONFIG=config.adxl345
# set: extra low-level config options [*], Architecture=RP2040,
# bootloader=16KiB, interface=USB, set USB ids
make KCONFIG_CONFIG=config.adxl345 -j4

# 2. flash
ls /dev/serial/by-id/   # note id containing 'rp2040'
make KCONFIG_CONFIG=config.adxl345 flash FLASH_DEVICE=/dev/serial/by-id/<your rp2040 id>

# 3. printer.cfg: set [mcu adxl345] serial to id from step 2
#    verify cs_pin / spi_software_*_pin against board's own pin diagram
FIRMWARE_RESTART

# 4. test
ACCELEROMETER_MEASURE CHIP=adxl345   # run, wait few sec, run again to stop; confirm non-zero data
MEASURE_AXES_NOISE                   # printer idle

# 5. calibrate
SHAPER_CALIBRATE AXIS=X
SHAPER_CALIBRATE AXIS=Y
# generates /tmp/calibration_data_{x,y}_*.csv

# 6. graph on host PC, not Pi
scp 'rpi:/tmp/calibration_data_*_*.csv' .
pip install -v numpy matplotlib   # once
python3 calibrate_shaper.py calibration_data_x_*.csv -o shaper_x.png
python3 calibrate_shaper.py calibration_data_y_*.csv -o shaper_y.png
# calibrate_shaper.py: ~/klipper/scripts/ — copy/clone to host PC

# 7. apply
SAVE_CONFIG   # auto-writes [input_shaper], or edit printer.cfg manually
FIRMWARE_RESTART
```

### Raw communication (debug only, bypasses Klippy)

Pins per printer.cfg: cs=gpio9, sclk=gpio10, mosi=gpio11, miso=gpio12. SPI mode 3.

ADXL345 registers used below (addresses/values per datasheet and `src/sensor_adxl345.c`):

| Register      | Addr   | Value used | Meaning                                    |
|----------------|--------|------------|---------------------------------------------|
| `DEVID`        | `0x00` | (read-only)| Fixed ID, always reads `0xE5` on a real chip |
| `BW_RATE`      | `0x2C` | `0x0A`     | Output data rate = 100Hz                     |
| `POWER_CTL`    | `0x2D` | `0x08`     | Measure bit set — starts conversions         |
| `DATA_FORMAT`  | `0x31` | `0x08`     | Full resolution, +/-2g range                 |
| `DATAX0`       | `0x32` | (read-only)| Start of 6-byte X/Y/Z burst (auto-increments)|
| `FIFO_CTL`     | `0x38` | `0x90`     | Stream mode, watermark=16 — **must be exactly this**, firmware sanity-checks it on every read |
| `FIFO_STATUS`  | `0x39` | (read-only)| Sample count currently in FIFO               |

SPI read/write framing: write = `addr` + `value` (e.g. `data=2c0a` writes `0x0A` to `0x2C`); single read
= `addr\|0x80` + dummy byte (e.g. `data=8000` reads `DEVID`), response's 2nd byte is the register value.

Tested on klipper v0.13.0-787-ge2b99a915 (RP2040) — command names/params below are current for that
version; `spi_set_software_bus`/`rate=`/`query_adxl345 clock=` from older docs no longer exist.

```
# connect — console.py takes baud via -b, not positional
~/klippy-env/bin/python ~/klipper/klippy/console.py -b 250000 /dev/serial/by-id/<rp2040 id>

# 1. allocate oids: oid 0 = spi bus, oid 1 = adxl345 device
# one-shot per MCU boot — a second attempt on the same boot shuts the MCU down with
# "oids already allocated"; power-cycle the board (unplug/replug USB) before retrying,
# reconnecting console.py alone does not reset MCU state
allocate_oids count=2

# 2. configure CS pin (oid 0)
config_spi oid=0 pin=gpio9 cs_active_high=0

# 3. configure software SPI bus on oid 0
# pulse_ticks = software SPI half-period in MCU clock ticks (not a rate=Hz param)
spi_set_sw_bus oid=0 miso_pin=gpio12 mosi_pin=gpio11 sclk_pin=gpio10 mode=3 pulse_ticks=2

# 4. bind adxl345 device (oid 1) to spi bus (oid 0)
config_adxl345 oid=1 spi_oid=0

# 5. finalize config
finalize_config crc=0

# 6. init chip registers over raw SPI (Klippy normally does this; required when bypassing it).
# data= is a hex string, bytes sent in the order written (see msgproto._parse_buffer).
spi_send oid=0 data=3108   # DATA_FORMAT (0x31) = 0x08: full-res, +/-2g
spi_send oid=0 data=2c0a   # BW_RATE (0x2C) = 0x0A: 100Hz output rate
spi_send oid=0 data=3890   # FIFO_CTL (0x38) = 0x90: stream mode, watermark=16 samples
                            # MUST be exactly 0x90 — firmware's query burst-reads this register
                            # as a built-in sanity check (see src/sensor_adxl345.c SET_FIFO_CTL)
                            # and silently fills every sample with 0xff if it doesn't match,
                            # which looks exactly like a dead/disconnected chip
spi_send oid=0 data=2d08   # POWER_CTL (0x2D) = 0x08: measure bit set, start conversions

# optional sanity check before trusting any of the above: DEVID (0x00) must read back 0xE5
# spi_transfer oid=0 data=8000

# 7. start measurement: rest_ticks = CLOCK_FREQ / sample_rate_hz
# read actual CLOCK_FREQ from the "MCU config:" line console.py prints on connect
# (varies per board, e.g. 12000000 — don't assume 48000000); no clock= param
query_adxl345 oid=1 rest_ticks=3750

# 8. check fifo status (buffer/overflow counters only, not samples)
query_adxl345_status oid=1

# 9. stop
query_adxl345 oid=1 rest_ticks=0
```

Samples stream back as `sensor_bulk_data oid=1 sequence=N data=b'...'` (generic bulk-sensor API) —
there is no separate fetch step. `query_adxl345` returns this raw binary FIFO data, unparsed; for
real calibration use `ACCELEROMETER_MEASURE`/`SHAPER_CALIBRATE` gcode above, not this.

Each sample is **5 bytes, bit-packed** (not 6 bytes / plain int16 triples):
`xlow, ylow, zlow, xzhigh, yzhigh`. Decode (matches `klippy/extras/adxl345.py` `_convert_samples`):

```python
def decode(xlow, ylow, zlow, xzhigh, yzhigh):
    rx = (xlow | ((xzhigh & 0x1f) << 8)) - ((xzhigh & 0x10) << 9)
    ry = (ylow | ((yzhigh & 0x1f) << 8)) - ((yzhigh & 0x10) << 9)
    rz = (zlow | ((xzhigh & 0xe0) << 3) | ((yzhigh & 0xe0) << 6)) - ((yzhigh & 0x40) << 7)
    return rx, ry, rz  # raw LSB counts, 3.9 mg/LSB at full resolution
```

All-`0xff` bytes in every sample almost always mean `FIFO_CTL` wasn't written as exactly `0x90`
(see step 6 above) — firmware treats any other readback as an SPI glitch and discards the sample.
This looks identical to a dead/disconnected chip but is a one-register config mistake, not a
wiring fault; rule it out first; before suspecting hardware, read `DEVID` (`0x00`) via
`spi_transfer` — a genuine ADXL345 always returns `0xE5` there regardless of `FIFO_CTL`.
