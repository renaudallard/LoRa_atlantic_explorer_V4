# LoRa Atlantic Explorer V4

ESPHome configuration to control an Atlantic Explorer V4 heat pump water heater
from Home Assistant over io-homecontrol, using a Heltec WiFi LoRa 32 V2 board
(ESP32 + SX1276, micro-USB, also sold by AZDelivery). No Cozytouch bridge, no
cloud and no MQTT broker: the board talks to the heater over the air and to
Home Assistant through the ESPHome native API.

The io-homecontrol stack is the
[home_io_control](https://github.com/laberning/home_io_control) ESPHome
component. Its water heater support has been confirmed on an Atlantic Calypso
Connecté. The Explorer V4 has not been reported there yet, so expect some
rough edges.

## The home_io_control component

There is nothing to download or install by hand. This block in
`explorer-v4.yaml` tells ESPHome where to get it:

    external_components:
      - source: github://laberning/home_io_control@main
        components: [home_io_control]

At build time ESPHome clones the repository, takes only the
`home_io_control` component and compiles it into the firmware. This works the
same from the Home Assistant ESPHome add-on and from the command line. The
machine doing the build needs internet access.

The clone is cached and refreshed at most once a day, so a build picks up
upstream changes from `main` within 24 hours. To get the latest fix right
away, add `refresh: always` next to `components:` for one build, or delete
the `.esphome/` directory.

Following `main` means an upstream change can break or change behaviour
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
| AUTO | `heat_pump` |
| Manual, ECO on | `eco` |
| Manual, ECO off | `performance` |

In AUTO the heater picks its own target, so set `eco` or `performance` before
setting a temperature. Off and holiday modes are not available.

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
| Hot water boost | switch | boost on or off (experimental) |

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
to ECO every night:

    automation:
      - alias: "Hot water ECO at night"
        triggers:
          - trigger: time
            at: "22:00:00"
        actions:
          - action: water_heater.set_operation_mode
            target:
              entity_id: water_heater.explorer_v4_hot_water
            data:
              operation_mode: eco

`water_heater.set_temperature` sets the target. It only takes effect in
`eco` or `performance` mode. The Calypso only accepts its own fixed
temperature steps, the Explorer V4 may behave the same: if a target is not
taken, use one of the values from the heater's display.

## Troubleshooting

Uncomment the `-DIOHOME_FRAME_LOG` build flag to log every radio frame (the
system key is masked). The build warning about `domestic_hot_water` not being
confirmed on a real device is expected. Reports about the Explorer V4 are
welcome upstream at
[home_io_control](https://github.com/laberning/home_io_control/issues).

## Tested with

- ESPHome 2026.10.2 (configuration validated and firmware built)
