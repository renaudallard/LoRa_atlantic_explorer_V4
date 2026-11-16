# LoRa Atlantic Explorer V4

ESPHome configuration to control an Atlantic Explorer V4 heat pump water heater
from Home Assistant over io-homecontrol, using a Heltec WiFi LoRa 32 V2 board
(ESP32 + SX1276, micro-USB, also sold by AZDelivery). No Cozytouch bridge, no
cloud and no MQTT broker: the board talks to the heater over the air and to
Home Assistant through the ESPHome native API.

The io-homecontrol stack is the
[home_io_control](https://github.com/laberning/home_io_control) ESPHome
component. Its water heater support was written for an Atlantic Calypso
Connecté. The Explorer V4 works with it, but needs the fixes described in
[Known issues on the Explorer V4](#known-issues-on-the-explorer-v4).

## The home_io_control component

There is nothing to download or install by hand. This block in
`explorer-v4.yaml` tells ESPHome where to get it:

    external_components:
      - source: github://renaudallard/home_io_control@explorer-v4
        components: [home_io_control]

This is the `explorer-v4` branch of a fork of
[laberning/home_io_control](https://github.com/laberning/home_io_control):
upstream `main` plus the `max_temperature` option the Explorer V4 needs. Once
that is merged upstream, go back to
`github://laberning/home_io_control@main`.

At build time ESPHome clones the repository, takes only the
`home_io_control` component and compiles it into the firmware. This works the
same from the Home Assistant ESPHome add-on and from the command line. The
machine doing the build needs internet access.

The clone is cached and refreshed at most once a day, so a build picks up
changes to the branch within 24 hours. To get the latest fix right
away, add `refresh: always` next to `components:` for one build, or delete
the `.esphome/` directory.

Following a branch means a change can break or change behaviour
between two builds. Upstream has no release tags, so to freeze a known good
version, replace `main` with a commit hash, for example:

    - source: github://laberning/home_io_control@33e2721727008e5b1f3b6d9bcfaeee45fe42c18e

Upstream documentation worth reading:

- [Getting started](https://github.com/laberning/home_io_control/blob/main/docs/getting-started.md)
- [Pairing](https://github.com/laberning/home_io_control/blob/main/docs/pairing.md)
- [Key extraction](https://github.com/laberning/home_io_control/blob/main/docs/key-extraction.md)
- [Water heaters](https://github.com/laberning/home_io_control/blob/main/docs/configuration/water-heater.md)
- [Troubleshooting](https://github.com/laberning/home_io_control/blob/main/docs/troubleshooting.md)

## Files

- `explorer-v4.yaml`: the ESPHome configuration
- `secrets.yaml`: secrets with placeholder values, to fill in locally

## Hardware

- Heltec WiFi LoRa 32 V2 or a clone with the same pinout, 868 MHz version.
  The radio chip must be an SX1276 (an SX1278 is the 433 MHz part).
- The 868 MHz antenna. Always connect it before powering the board.
- A micro-USB data cable for the first flash.

Radio pins used (from the component's `config/boards/heltec-v2.yaml`):

| Signal | GPIO |
|--------|------|
| SPI CLK | 5 |
| SPI MOSI | 27 |
| SPI MISO | 19 |
| CS | 18 |
| RST | 14 |
| DIO0 | 26 |

## Setup

### 1. Secrets

Edit `secrets.yaml`, fill in your Wi-Fi credentials and generate the keys:

    openssl rand -base64 32   # api_key, also used to encrypt OTA updates
    openssl rand -hex 3       # hub_node_id
    openssl rand -hex 16      # hub_system_key

`hub_system_key` gives control over the heater: keep it private and do not
commit the filled in file. Never change
`hub_node_id` or `hub_system_key` once a device is paired, or you will have to
pair it again.

### 2. First flash over USB

Either from the ESPHome add-on in Home Assistant (new device, paste the YAML,
copy the secrets into the add-on secrets, then Install and "Plug into this
computer", which needs Chrome or Edge), or from the command line:

    esphome run explorer-v4.yaml --device /dev/ttyUSB0

If the upload does not start, hold PRG, press and release RST, then release
PRG. The board uses a CP2102 USB chip (clones may use a CH340). Linux has
drivers for both, Windows may need one.

Then add the board to Home Assistant, see [Adding the board](#adding-the-board).
Later updates go over Wi-Fi.

### 3. Check the radio

In the logs (`esphome logs explorer-v4.yaml` or the add-on), the boot block
must name the SX1276 without errors. Pressing any io-homecontrol remote nearby
should log `rx 1W remote` lines. If not, check the antenna and the pins.

### 4. Pair the heater

- Heater never connected to a Cozytouch bridge: start pairing from the
  heater's connectivity menu (see its manual), then press "Discover & Pair" in
  Home Assistant within a few seconds. Two or three attempts are common.
- Heater already on a Cozytouch bridge: turn on "Recover System Key", then add
  a device from the Cozytouch app. The log prints the installation's
  `node_id` and `system_key`. Put them in `secrets.yaml`, flash again and press
  "Scan Paired Devices".

Either way the log prints the heater's `io_device_id`. Put it in
`explorer_io_device_id` in `secrets.yaml`.

### 5. Second flash

Uncomment the `water_heater:` block in `explorer-v4.yaml`. If the pairing log
reports another `io_device_type` or an `io_subtype`, use those values. Flash
again.

The heater entities then show up in Home Assistant, see
[In Home Assistant](#in-home-assistant).

| Heater | Home Assistant mode |
|--------|---------------------|
| AUTO | `eco` |
| Manual, ECO on | `heat_pump` |
| Manual, ECO off | `performance` |

This is the mapping of current upstream `main`, which the fork follows. Builds
from before upstream commit 8f33376 (2026-11-15) used AUTO as `heat_pump` and
manual ECO as `eco`.

In AUTO the heater picks its own target, so set `heat_pump` or `performance`
before setting a temperature. Off and holiday modes are not available.

### 6. Clean up

Once the heater works, remove `discover_and_pair_button`,
`accept_foreign_pairing` and `scan_paired_devices_button`, then flash again.

## In Home Assistant

Water heater entities from ESPHome need Home Assistant 2026.2 or later.

### Adding the board

The data goes straight from the board to Home Assistant through the ESPHome
integration. There is no broker or cloud service to set up.

1. Once the board is on Wi-Fi, Home Assistant finds it by itself. Go to
   Settings > Devices & services, where "Explorer V4" is listed as
   Discovered, and select Configure.
2. When asked for the encryption key, paste the `api_key` from
   `secrets.yaml`.

If the board is not discovered (mDNS blocked, other VLAN), add it by hand:
Settings > Devices & services > Add Integration > ESPHome, then enter the
board's IP address (printed in the log, or shown by your router), port
`6053`, and the `api_key`.

### Entities

After the second flash, the "Explorer V4" device page lists:

| Entity | Type | Content |
|--------|------|---------|
| Hot water | water heater | water temperature, target, mode |
| Hot water energy | sensor | kWh used since installation |
| Hot water heat pump | binary sensor | on while the compressor runs |
| Hot water electric backup | binary sensor | on while the electric element heats |
| Hot water boost | switch | boost on or off, reads inverted on the Explorer V4, see [The boost switch shows the opposite](#the-boost-switch-shows-the-opposite) |

The component also adds diagnostic entities such as the radio link health.
Values appear a few seconds after boot, then the heater is read every 5
minutes (`status_poll_interval`). Entity IDs are built from the device and
entity names, for example `water_heater.explorer_v4_hot_water`. Check the
exact IDs under Settings > Devices & services > Entities before using them in
automations.

### Dashboard

Add a Tile card with the Hot water entity. Selecting it opens the controls
for the mode and the target temperature. Add the binary sensors to the same
card or to an Entities card to see what is heating.

### Energy dashboard

The energy sensor counts up since the heater was installed, which is what the
Energy dashboard expects. Go to Settings > Dashboards > Energy and add
"Hot water energy" under Individual devices. Home Assistant only counts the
consumption from that moment on.

### Automations

The water heater entity takes the standard actions. For example, to switch
to manual ECO every night:

    automation:
      - alias: "Hot water manual ECO at night"
        triggers:
          - trigger: time
            at: "22:00:00"
        actions:
          - action: water_heater.set_operation_mode
            target:
              entity_id: water_heater.explorer_v4_hot_water
            data:
              operation_mode: heat_pump

`water_heater.set_temperature` sets the target. It only takes effect in
`heat_pump` or `performance` mode, and the heater does not keep every value, see
[Some targets are not kept](#some-targets-are-not-kept).

## Known issues on the Explorer V4

### The target differs from the heater's display

The card shows a lower target than the display, and the gap grows with the
temperature: 46 °C on the card for 51 °C on the display, 49 °C for 55 °C.

The heater carries its target on a scale from 20 to 70 °C, where the
component assumes the Calypso's 20 to 62 °C. Checked on the heater: 51 °C on
the display is reported as `7C00`, 55 °C as `8C00`, and a target of 47 sent
on the old scale (`8092`) showed up as 52 °C.

To fix it, build from the fork and tell it the scale. In your configuration
(`explorer-v4.yaml`, or your own file in the ESPHome add-on):

1. Set the component source:

        external_components:
          - source: github://renaudallard/home_io_control@explorer-v4
            components: [home_io_control]
            refresh: always

   `refresh: always` makes the build fetch the branch again. Remove it once
   the firmware works.

2. Add `max_temperature: 70` to the water heater:

        water_heater:
          - platform: home_io_control
            name: "Hot water"
            io_device_id: !secret explorer_io_device_id
            io_device_type: domestic_hot_water
            max_temperature: 70

3. Install the firmware. The log shows
   `Target temperature range: 20-70 C` at boot, and the card's target now
   matches the display.

### Some targets are not kept

The heater answers every target write with "adjusted" (result code 10 in the
log) and keeps the closest value it allows. In manual ECO it keeps at most
55 °C: a higher target comes back as 55. For more, use manual with ECO off;
the heater reported 60 and 62 °C there. The card shows what the heater kept
once it reads the target back, a few seconds after the write.

### The boost switch shows the opposite

The Explorer V4 reports its boost the other way round from the Calypso. The
`boost` switch shows on while no boost runs and off during a boost, and
turning it off while no boost runs does nothing. Starting a boost from Home
Assistant has not been tested yet.

Until the component handles this, leave the `boost:` entry out and start a
boost from the heater's display. `electric_backup_running` shows a boost
instead: it was on during each boost seen so far, and off otherwise.

## Troubleshooting

Uncomment the `-DIOHOME_FRAME_LOG` build flag to log every radio frame (the
system key is masked). The build warning about `domestic_hot_water` not being
confirmed on a real device is expected. Reports about the Explorer V4 are
welcome upstream at
[home_io_control](https://github.com/laberning/home_io_control/issues).

## Tested with

- ESPHome 2026.10.2 (configuration validated and firmware built)
