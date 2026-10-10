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
