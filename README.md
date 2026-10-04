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
