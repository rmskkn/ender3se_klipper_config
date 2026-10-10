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

`SHAPER_CALIBRATE` needs an accelerometer (ADXL345). Current `[input_shaper]` values in `printer.cfg` were copied from a different printer/frame — redo this calibration on the actual hardware.

### Setup

1. Enable SPI:
   ```
   ssh rpi "sudo raspi-config nonint do_spi 0"
   ```
   Reboot after.

2. Wire ADXL345 to Pi SPI0 (CE0/MOSI/MISO/SCLK + 3.3V/GND), mount rigid on toolhead.

3. Add to `printer.cfg`:
   ```
   [mcu rpi]
   serial: /tmp/klipper_host_mcu

   [adxl345]
   cs_pin: rpi:None
   axes_map: z,y,x    # verify for your mount orientation

   [resonance_tester]
   accel_chip: adxl345
   probe_points:
       115,115,20     # bed center, adjust to your bed size
   ```

4. Install graph deps on Pi:
   ```
   ssh rpi "~/klippy-env/bin/pip install -v numpy matplotlib"
   ```

5. `FIRMWARE_RESTART` to load config.

### Calibrate

6. Test accel wiring:
   ```
   ACCELEROMETER_MEASURE CHIP=adxl345
   ```
   Wait few sec, run again to stop. Confirm non-zero data, no error.

7. Noise floor check (printer idle):
   ```
   MEASURE_AXES_NOISE
   ```

8. Run calibration:
   ```
   SHAPER_CALIBRATE AXIS=X
   SHAPER_CALIBRATE AXIS=Y
   ```
   Toolhead moves/vibrates, keep area clear. Generates `/tmp/calibration_data_x_*.csv` / `_y_*.csv`.

9. Graph results:
   ```
   ~/klipper/scripts/calibrate_shaper.py /tmp/calibration_data_x_*.csv -o /tmp/shaper_x.png
   ```
   Repeat for Y. Read recommended shaper type/freq/max_accel.

10. Apply: `SAVE_CONFIG` (auto-writes `[input_shaper]`) or manually edit the `[input_shaper]` section in `printer.cfg`.

11. `FIRMWARE_RESTART` to load final values.
