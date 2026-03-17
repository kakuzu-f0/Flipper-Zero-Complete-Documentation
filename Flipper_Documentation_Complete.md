
# Flipper Zero Complete Documentation

## Table of Contents

1. [Development & Build System](#development--build-system)
2. [Applications & Plugins](#applications--plugins)
3. [Hardware & Protocols](#hardware--protocols)
4. [File Formats](#file-formats)
5. [Radio & Sub-GHz](#radio--sub-ghz)
6. [Infrared](#infrared)
7. [NFC & RFID](#nfc--rfid)
8. [System & Debugging](#system--debugging)
9. [Tools & Utilities](#tools--utilities)
10. [Configuration & Settings](#configuration--settings)

---

## Development & Build System

### Flipper Build Tool (FBT)

FBT is the entry point for firmware-related commands and utilities. It is invoked by `./fbt` in the firmware project root directory. Internally, it is a wrapper around scons build system.

#### Environment

To use `fbt`, you only need `git` installed in your system.

`fbt` by default downloads and unpacks a pre-built toolchain, and then modifies environment variables for itself to use it. It does not contaminate your global system's path with the toolchain.

#### Key FBT Targets

- `fw_dist` — build & publish firmware to the `dist` folder
- `fap_dist` — build external plugins & publish to the `dist` folder
- `updater_package`, `updater_minpackage` — build a self-update package
- `flash` — flash the attached device over SWD interface
- `flash_usb`, `flash_usb_full` — build, upload and install the update package over USB
- `debug` — build and flash firmware, then attach with gdb
- `faps` — build all external & plugin apps as `.faps`

#### VSCode Integration

`fbt` includes basic development environment configuration for VS Code. Run `./fbt vscode_dist` to deploy it.

### Hardware Targets

Flipper's firmware is modular and supports different hardware configurations in a common code base. It encapsulates hardware-specific differences in `furi_hal`, board initialization code, linker files, SDK data and other information in a _target definition_.

#### Target Definition File

A target definition file, `target.json`, is a JSON file that can contain fields like:
- `include_paths`: list of strings, folder paths relative to current target folder
- `sdk_header_paths`: list of strings, folder paths to gather headers from for SDK
- `startup_script`: filename of a startup script, performing initial hardware initialization
- `linker_script_flash`: filename of a linker script for creating the main firmware image

### Unit Tests

Unit tests are special pieces of code that apply known inputs to the feature code and check the results to see if they are correct. Flipper Zero firmware includes a separate app called `unit_tests`.

#### Running Unit Tests

1. Compile the firmware with the tests enabled: `./fbt FIRMWARE_APP_SET=unit_tests updater_package`
2. Flash the firmware using your preferred method
3. Launch the CLI session and run the `unit_tests` command

---

## Applications & Plugins

### FAM (Flipper App Manifests)

All components of Flipper Zero firmware — services, user applications, and system settings — are developed independently. Each component has a build system manifest file named `application.fam`, which defines the basic properties of that component and its relations to other parts of the system.

#### App Definition Parameters

- **appid**: string, app ID within the build system
- **apptype**: member of FlipperAppType.* enumeration (SERVICE, SYSTEM, APP, PLUGIN, DEBUG, ARCHIVE, SETTINGS, STARTUP, EXTERNAL, METAPACKAGE)
- **name**: name displayed in menus
- **entry_point**: C function to be used as the app's entry point
- **requires**: list of app IDs to include in the build configuration
- **conflicts**: list of app IDs with which the current app conflicts
- **stack_size**: stack size in bytes to allocate for an app on its startup

#### External App Parameters

- **sources**: list of strings, file name masks used for gathering sources
- **fap_version**: string, app version
- **fap_icon**: name of a `.png` file, 1-bit color depth, 10x10px
- **fap_libs**: list of extra libraries to link the app against
- **fap_category**: string, app subcategory
- **fap_description**: string, short app description
- **fap_author**: string, app's author

### FAP (Flipper App Package)

`fbt` supports building apps as FAP files. FAPs are essentially `.elf` executables with extra metadata and resources bundled in.

#### Setting up FAP Development

To build your app as a FAP, create a folder with your app's source code in `applications_user`, then write its code and configure its `application.fam` manifest, setting its `apptype` to `FlipperAppType.EXTERNAL`.

#### Building FAPs

- `./fbt fap_{APPID}` - build specific app
- `./fbt launch APPSRC=applications_user/path/to/app` - build and upload
- `./fbt faps` - build all FAPs
- `./fbt fap_dist` - build and deploy to dist folder

#### FAP Assets

FAPs can include static and animated images as private assets. Put your images in a subfolder inside your app's folder, then reference that folder in your app's manifest in the `fap_icon_assets` field.

#### Debugging FAPs

`fbt` includes a script for gdb-py to provide debugging support for FAPs, `debug/flipperapps.py`. With it, you can debug FAPs as if they were a part of the main firmware.

### Asset Packs

Asset Packs are a feature that allows you to load custom Animation and Icon sets without recompiling the firmware.

#### Installing Asset Packs

1. Open qFlipper and navigate to `SD Card` and into `asset_packs`
2. Unzip your packs and upload the folders to `SD/asset_packs/PackName/`
3. Open the Momentum Settings app and select the asset pack you want

#### Creating Asset Packs

Asset Packs are made of 2 parts: Anims and Icons.

**Animations:**
- Use standard animation format
- Go in `SD/asset_packs/PackName/Anims` instead of `SD/dolphin`
- Momentum has up to level 30

**Icons:**
- Static icons use `.bmx` format: `[width][height][pixel data]`
- Animated icons stored as `.bm` sequences with `meta` files
- Structure: `SD/asset_packs/PackName/Icons/`

#### Building Asset Packs

Use the `scripts/asset_packer.py` script to compile asset packs from PNG images.

---

## Hardware & Protocols

### Expansion Module Protocol

Expansion Module Protocol is a serial-based, byte-oriented, synchronous communication protocol for third-party hardware units meant for use with Flipper Zero.

#### Features

- Automatic expansion module detection
- Baud rate negotiation
- Basic error detection
- Request-response communication flow
- Integration with Flipper RPC protocol

#### Hardware Connections

| UART   | Tx pin | Rx pin |
|--------|--------|--------|
| USART  | 13     | 14     |
| LPUART | 15     | 16     |

#### Frame Structure

Each frame consists of:
- Header (1 byte): Frame type
- Contents (0 or more bytes): Frame payload
- Checksum (1 byte): XOR checksum

#### Frame Types

- **HEARTBEAT** (0x01): Maintain idle connection
- **STATUS** (0x02): Report transaction status
- **BAUD RATE** (0x03): Negotiate communication speed
- **CONTROL** (0x04): Control communication features
- **DATA** (0x05): Transmit arbitrary data

### FuriHalBus API

On system startup, most peripheral devices are under reset and not clocked by default to reduce power consumption.

#### Peripheral Categories

**Always-on peripherals** (enabled by system, never disable):
- DMA1, DMA2, DMAMUX
- GPIOA, GPIOB, GPIOC, GPIOD, GPIOE, GPIOH
- PKA, AES2, HSEM, IPCC, FLASH

**On-demand system peripherals** (enabled/disabled by system):
- RNG, SPI1, SPI2, I2C1, I2C3, USART1, LPUART1, USB

**On-demand shared peripherals** (must be enabled by user code):
- CRC, TSC, ADC, QUADSPI, TIM1, TIM2, TIM16, TIM17, LPTIM1, LPTIM2, SAI1, LCD

#### Bus Control Functions

- `furi_hal_bus_enable()`: Enable peripheral
- `furi_hal_bus_disable()`: Disable peripheral
- `furi_hal_bus_reset()`: Reset peripheral

### Furi Check System

The best way to protect system integrity is to reduce amount cases that we must handle and crash the system as early as possible.

#### Check Functions

- `furi_assert(CONDITION)`: Assert condition in development, crash if false
- `furi_check(CONDITION)`: Always assert condition, crash if false
- `furi_crash()`: Crash the system
- `furi_halt()`: Halt the system

#### Debug vs Production

Debug builds will never reset system to preserve state for debugging. If debugger is connected, system will stop before reboot automatically.

### Furi HAL Debugging

Some Furi subsystems have additional debugging features that can be enabled by adding extra defines to firmware compilation.

#### FuriHalOs Debug

`--extra-define=FURI_HAL_OS_DEBUG` enables:
- `AWAKE` (PA7): High when system is busy, low when sleeping
- `TICK` (PA6): Flipped on system tick
- `SECOND` (PA4): Flipped each second

#### FuriHalPower Debug

`--extra-define=FURI_HAL_POWER_DEBUG` enables:
- `WFI` (PB2): Light sleep (wait for interrupt)
- `STOP` (PC3): STOP mode used

#### FuriHalSD Debug

`--extra-define=FURI_HAL_SD_SPI_DEBUG` enables SD card SPI bus logging.

---

## File Formats

### Flipper File Format (FFF)

Flipper uses a simple text-based file format for many of its data files. The format consists of key-value pairs separated by colons, with comments starting with `#`.

#### Common Fields

- `Filetype`: Type identifier for the file
- `Version`: Format version number
- Additional protocol-specific fields

### Heatshrink-compressed Tarball Format

Flipper supports the use of Heatshrink compression library for `.tar` archives.

#### Header Structure

- Magic value: `0x48 0x53 0x44 0x53` (ASCII "HSDS")
- Version number: `0x01`
- Window size: Single byte (sliding window size)
- Lookahead size: Single byte (lookahead buffer size)
- Total header size: 7 bytes

---

## Radio & Sub-GHz

### Sub-GHz File Formats

Flipper uses `.sub` files to store SubGhz signals. These files can contain either a SubGhz Key with a certain protocol or SubGhz RAW data.

#### .sub File Structure

1. **Header**: File type, version, and frequency
2. **Preset information**: Preset type and transceiver configuration data
3. **Protocol and data**: Protocol name and specific data or RAW data

#### Header Fields

| Field       | Type   | Description                                    |
| ----------- | ------ | ---------------------------------------------- |
| `Filetype`  | string | Must be `Flipper SubGhz Key File`             |
| `Version`   | uint   | Current version is 1                           |
| `Frequency` | uint   | Frequency in Hertz                             |

#### Preset Types

- `FuriHalSubGhzPresetOok270Async` — On/Off Keying, 270kHz bandwidth
- `FuriHalSubGhzPresetOok650Async` — On/Off Keying, 650kHz bandwidth
- `FuriHalSubGhzPreset2FSKDev238Async` — 2 FSK, deviation 2kHz
- `FuriHalSubGhzPreset2FSKDev12KAsync` — 2 FSK, deviation 12kHz
- `FuriHalSubGhzPreset2FSKDev476Async` — 2 FSK, deviation 47kHz

#### Protocol Types

**Key Files**: Contain protocol name and specific data (key value, bit length, etc.)

**RAW Files**: Contain raw signal data not processed through protocol-specific decoding

**BIN_RAW Files**: Record useful repeating sequence with restored byte transfer rate

### Sub-GHz Supported Protocols

#### Static & Dynamic Protocols

**Garage Door Openers & Gate Openers:**
- Alutech AT-4N `433.92MHz` `AM650` (72 bits, Dynamic)
- AN-Motors AT4 `433.92MHz` `AM650` (64 bits, Pseudo-Dynamic)
- BFT Mitto `433.92MHz` `AM650` (64 bits, Dynamic)
- CAME Atomo `433.92MHz, 868MHz` `AM650` (62 bits, Dynamic)
- CAME TWEE `433.92MHz` `AM650` (54 bits, Static)
- FAAC SLH `433.92MHz, 868.35MHz` `AM650` (64 bits, Dynamic)
- Nice FloR-S `433.92MHz` `AM650` (52 bits, Dynamic)
- Security+1.0 `315MHz, 433.92MHz, 390MHz` `AM650` (42 bits, Dynamic)
- Security+2.0 `310MHz, 390MHz, 868MHz` `AM650` (62 bits, Dynamic)

**Sensors & Smart Home:**
- Intertechno V3 `AM650` (32 bits, Static)
- Somfy Telis `433.92MHz` `AM650` (56 bits, Dynamic)
- Honeywell `AM650` (64 bits, Static)

#### KeeLoq Rolling Code Manufacturers

Over 60 manufacturers supported including:
- Allmatic, Aprimatic, Beninca, CAME Space, Cardin S449
- DoorHan, FAAC RC/XT, Hormann EcoStar, Nice Smilo
- And many more alarm and automotive systems

### Sub-GHz Settings

#### Adding Custom Frequencies

Edit user settings file `subghz/assets/setting_user`:

```
Filetype: Flipper SubGhz Setting File
Version: 1

Add_standard_frequencies: true
Default_frequency: 433920000

Frequency: 300000000
Frequency: 310000000
Hopper_frequency: 300000000
```

#### Frequency Range Extension

**CC1101 Frequency range specs:** 300-348 MHz, 386-464 MHz, and 778-928 MHz

**Extended range:** 281-361 MHz, 378-481 MHz, and 749-962 MHz

⚠️ **WARNING**: Extending frequency ranges can damage hardware!

### Sub-GHz Counter Mode

Experimental Counter Mode allows customization of how rolling codes increment when transmitting SubGHz signals.

#### Usage

Add `CounterMode: X` to your `.sub` file, where X is the mode number.

#### Supported Protocols

**Nice Flor S:**
- Mode 0: Standard (+1)
- Mode 1: Counter sequence `0x0001 / 0xFFFE`
- Mode 2: Counter sequence `0x0000 / 0x0001`

**Came Atomo:**
- Mode 0: Standard (+1)
- Mode 1: Counter sequence `0x0000 / 0x0001 / 0xFFFE / 0xFFFF`
- Mode 2: Counter sequence `0x807B / 0x807C / 0x007B / 0x007C`
- Mode 3: Counter freeze

**KeeLoq:**
- Mode 0: Standard (+1)
- Mode 1: Counter sequence `0x0000 / 0x0001 / 0xFFFE / 0xFFFF`
- Mode 2: Incremental mode `+0x3333`
- Mode 3: Counter sequence `0x8006 / 0x8007 / 0x0006 / 0x0007`
- And more...

### Sub-GHz Remote Programming

#### Creating New Remotes

**FAAC SLH:**
1. Create new remote: SubGHz → Add Manually → FAAC SLH
2. Hold Up arrow on Flipper + press programming button on receiver
3. Press Send button couple times

**Doorhan:**
1. Create new remote: SubGHz → Add Manually → KL: Doorhan
2. Press Radio button on receiver until LED lights
3. Press FZ button 2 times

**CAME Atomo:**
1. Create new remote: SubGHz → Add Manually → CAME Atomo
2. Press and hold button on existing remote for ~10 seconds
3. Long press Send on Flipper for 3-4 seconds

**Nice Flor S:**
1. Create new remote: SubGHz → Add Manually → Nice FloR-S
2. Press Send on Flipper for 5+ seconds, release
3. Press button on existing remote 3 times slowly
4. Press Send on Flipper slowly

### Sub-GHz Remote Plugin

The SubGHz Remote Tool requires creation of custom user map with `.txt` extension in the `subghz/remote` folder.

#### Map File Format

```
UP: /ext/subghz/Up.sub
DOWN: /ext/subghz/Down.sub
LEFT: /ext/subghz/Left.sub
RIGHT: /ext/subghz/Right.sub
OK: /ext/subghz/Ok.sub
ULABEL: Up Label
DLABEL: Down Label
LLABEL: Left Label
RLABEL: Right Label
OKLABEL: Ok Label
```

---

## Infrared

### Infrared File Formats

#### Supported Protocols

```
NEC, NECext, NEC42, NEC42ext, Samsung32, RC6, RC5, RC5X, 
SIRC, SIRC15, SIRC20, Kaseikyo, RCA
```

#### Infrared Remote File Format

**Filename extension:** `.ir`

**Example:**
```
Filetype: IR signals file
Version: 1
name: Button_1
type: parsed
protocol: NECext
address: EE 87 00 00
command: 5D A0 00 00
```

**Fields:**
- `name`: Button name (ASCII only)
- `type`: `parsed` or `raw`
- `protocol`: Protocol name (for parsed)
- `address`: Payload address (4 bytes, for parsed)
- `command`: Payload command (4 bytes, for parsed)
- `frequency`: Carrier frequency in Hz (for raw)
- `duty_cycle`: Carrier duty cycle (for raw)
- `data`: Raw signal timings in microseconds (for raw)

#### Infrared Library File Format

Used for universal remote libraries. Identical to remote format but with different `Filetype` field.

#### Infrared Test File Format

**Filename extension:** `.irtest`

Used for technical test data. Supports parsed signal arrays and raw data.

### Infrared Captures

#### Recording Process

1. Get the remote and point it to Flipper's IR receiver
2. Start learning a new remote or press `+` to add new button
3. Do a Quick Press of remote button (don't hold down)
4. Save under corresponding name

#### Data Types

**Parsed Data:** Clean, recognized code
```
name: EXAMPLE
type: parsed
protocol: NEC
address: 07 00 00 00
command: 02 00 00 00
```

**Raw Data:** Unrecognized signal
```
name: EXAMPLE
type: raw
frequency: 38000
duty_cycle: 0.330000
data: 2410 597 1189 599 592 600 ...
```

#### Air Conditioner Recording

Air conditioners track state in the remote. Recording process:

1. Press POWER button to turn A/C ON
2. Set A/C to corresponding mode (temperature, fan on AUTO)
3. Press POWER to switch A/C off
4. Start learning new signal on Flipper
5. Press POWER button once again
6. Save under specified name

**Required signals:** Off, Dh, Cool_hi, Cool_lo, Heat_hi, Heat_lo

### Universal Remotes

#### Televisions

Up to 6 signals: Power, Mute, Vol_up, Vol_dn, Ch_next, Ch_prev

#### Audio Players

Up to 8 signals: Power, Play, Pause, Vol_up, Vol_dn, Next, Prev, Mute

#### Projectors

Up to 4 signals: Power, Mute, Vol_up, Vol_dn

#### Air Conditioners

6 signals: Off, Dh, Cool_hi, Cool_lo, Heat_hi, Heat_lo

---

## NFC & RFID

### NFC File Formats

#### General Format

```
Filetype: Flipper NFC device
Version: 4
Device type: ISO14443-4A
UID: 04 48 6A 32 33 58 80
-------------------------
(Device-specific data)
```

#### Supported Device Types

- ISO14443-3A, ISO14443-3B, ISO14443-4A
- NTAG/Ultralight, Mifare Classic, Mifare DESFire

#### ISO14443-3A

```
Device type: ISO14443-3A
UID: 34 19 6D 41 14 56 E6
ATQA: 00 44
SAK: 00
```

#### NTAG/Ultralight

Includes internal data, signature, version, counters, and page data.

#### Mifare Classic

```
Device type: Mifare Classic
Mifare Classic type: 4K
Data format version: 2
Block 0: BA E2 7C 9D B9 18 02 00 46 44 53 37 30 56 30 31
Block 1: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
...
```

#### Mifare DESFire

Contains PICC version, free memory, applications, and file data.

#### Mifare Classic Dictionary

Contains list of Mifare Classic keys as hex strings.

#### EMV Resources

Stores EMV currency codes, country codes, or AIDs and their names.

### LF RFID File Format

#### Example

```
Filetype: Flipper RFID key
Version: 1
Key type: EM4100
Data: 01 23 45 67 89
```

#### Supported Key Types

- EM4100, H10301, Idteck, Indala26, IOProxXSF
- AWID, FDX-A, FDX-B, HIDProx, HIDExt
- Pyramid, Viking, Jablotron, Paradox, PAC/Stanley
- Keri, Gallagher, GProxII

### Reading RAW RFID Data

Flipper Zero can read RAW data from 125 kHz cards:

1. Activate Debug mode: Main Menu → Settings → System → Debug → ON
2. Go to Main Menu → 125 kHz RFID → Extra Actions
3. Select RAW RFID data and name the file
4. Apply card to Flipper's back
5. Two files (ASK and PSK) saved in `lfrfid` folder

### iButton File Format

#### Example

```
Filetype: Flipper iButton key
Version: 2
Protocol: DS1992
Rom Data: 08 DE AD BE EF FA CE 4E
Sram Data: 4E 65 76 65 72 47 6F 6E 6E 61 47 69 76 65...
```

#### Supported Protocols

- DS1990, DS1992, DS1996, DS1971, DS1420
- DSGeneric, Cyfral, Metakom

#### Protocol-specific Fields

- **Dallas protocols**: Rom Data, Sram Data, Eeprom Data
- **Cyfral & Metakom**: Data field

---

## System & Debugging

### Key Combinations

#### Basic Combos

**Hardware Reset:**
- Press `LEFT` and `BACK` and hold for couple of seconds
- Release `LEFT` and `BACK`

**Hardware Power Reset:**
- Disconnect USB and external power
- Press `BACK` and hold for 30 seconds
- Release `BACK` key

**Software DFU:**
- Press `LEFT` on boot to enter DFU with Flipper boot-loader

**Hardware DFU:**
- Press `OK` on boot to enter DFU with ST boot-loader

#### DFU Combos

**Hardware Reset + Software DFU:**
- Press `LEFT` and `BACK` and hold
- Release `BACK` (device enters DFU with indication)
- Release `LEFT`

**Hardware Reset + Hardware DFU:**
- Press `LEFT`, `BACK` and `OK` and hold
- Release `BACK` and `LEFT` (enters DFU without indication)

### OTA Update Process

Flipper firmware supports Over-The-Air updates through a special boot mode that loads system images into RAM.

#### Update Stages

1. **Backing up internal storage** (/int)
2. **Performing device update**
3. **Restoring internal storage and updating resources**

#### Update Manifest

Contains mandatory fields:
- Filetype: "Flipper firmware upgrade configuration"
- Version: 2
- Info: Package description
- Target: Hardware revision
- Loader: Stage 2 loader filename
- Loader CRC: CRC32 of loader file

Optional fields:
- Radio: Radio stack image
- Resources: TAR archive with SD card resources
- OB reference/mask/write mask: Option byte values

#### Error Codes

Format: `[XX-YY]` where XX is failed operation, YY contains progress details.

- **1**: Loading update manifest errors
- **2**: Backing up configuration errors
- **3-6**: Radio firmware operations
- **7**: Core2 operations
- **8**: Option bytes validation
- **9-11**: DFU file operations
- **12-15**: Resource operations

### Custom Flipper Name

#### Instructions

1. Go to Momentum → Misc → Spoofing Options → Flipper Name
2. Enter new custom name and click `Save`
3. Exit from Momentum app (Flipper will automatically reboot)
4. New name appears in device info and passport

#### Effects

Changing Flipper name affects:
- Bluetooth device name
- Bluetooth MAC address (3 bytes of Flipper ID + 3 bytes of ASCII from custom name)
- USB device name
- Serial number

To reset: Leave name empty and click `Save`.

---

## Tools & Utilities

### BadUSB Script Format

BadUsb app uses extended Duckyscript syntax compatible with USB Rubber Ducky 1.0.

#### Script File Format

- Text scripts from `.txt` files
- Both `\n` and `\r\n` line endings supported
- Empty lines allowed
- Spaces or tabs for indentation

#### Command Set

**Comments:**
```
REM Comment text
```

**Delays:**
```
DELAY 1000              # Single delay
DEFAULT_DELAY 500       # Delay before every command
```

**Special Keys:**
DOWNARROW/DOWN, LEFTARROW/LEFT, RIGHTARROW/RIGHT, UPARROW/UP
ENTER, DELETE, BACKSPACE, END, HOME, ESCAPE/ESC, INSERT
PAGEUP, PAGEDOWN, CAPSLOCK, NUMLOCK, SCROLLLOCK, PRINTSCREEN
BREAK/PAUSE, SPACE, TAB, MENU/APP, F1-F12

**Modifier Keys:**
CTRL/CONTROL, SHIFT, ALT, GUI/WINDOWS

**Key Hold/Release:**
```
HOLD CTRL
STRING test
RELEASE CTRL
```

**Strings:**
```
STRING Hello World
STRINGLN Hello World    # With enter
```

**Mouse Commands:**
```
LEFTCLICK
RIGHTCLICK
MOUSEMOVE 100 50
MOUSESCROLL -3
```

**Device ID:**
```
ID 1234:abcd Flipper Devices:Flipper Zero    # USB
BLE_ID AA:BB:CC:DD:EE:FF Smart Fridge       # Bluetooth
```

### MultiConverter

Expanded unit converter supporting multiple unit groups.

#### Current Conversions

- Decimal / Hexadecimal / Binary
- Celsius / Fahrenheit / Kelvin
- Kilometers / Meters / Centimeters / Miles / Feet / Inches
- Degree / Radian

#### Usage

- Base keyboard allows numbers from `0` to `F`
- Long press on `0` toggles negative value
- Long press on `1` sets decimal point
- `<` removes last character
- `#` changes to Unit Select Mode

#### Adding New Units

1. Add units to `MultiConverterUnitType` enum
2. Increase `MULTI_CONVERTER_AVAILABLE_UNITS` constant
3. Define conversion and check functions
4. Add `MultiConverterUnit` structs
5. Add to main `multi_converter_available_units` array

### NRF24 Driver

NRF24 driver for Flipper Zero device. Popular 2.4GHz radio transceivers from Nordic Semiconductors.

#### Usage

1. Connect NRF24 to Flipper using provided pinouts
2. Open NRF24: Sniffer and scan channels
3. When you get address → Open NRF24: Mouse Jacker
4. Select Address and open badusb file

#### Pinout

```
2/A7 (FZ) → MOSI/6 (nrf24l01)
3/A6 (FZ) → MISO/7 (nrf24l01)
4/A4 (FZ) → CSN/4 (nrf24l01)
5/B3 (FZ) → SCK/5 (nrf24l01)
6/B2 (FZ) → CE/3 (nrf24l01)
8/GND (FZ) → GND/1 (nrf24l01)
9/3V3 (FZ) → VCC/2 (nrf24l01)
```

#### Hardware Notes

If module is flakey, try adding capacitor (3.3 µF to 10 µF) to VCC/GND lines.

### Sentry Safe Plugin

Exploits vulnerability to open Sentry Safe and Master Lock electronic safes without pin code.

#### Usage

1. Start "Sentry Safe" plugin
2. Place wires:
   - 8/GND (Flipper GPIO) → Black wire (Safe)
   - 15/C1 (Flipper GPIO) → Green wire (Safe)
3. Press enter
4. Open safe

---

## Configuration & Settings

### Sub-GHz Bypass & Extend

#### Region Lock Bypass

**CC1101 Frequency range specs:** 300-348 MHz, 386-464 MHz, and 778-928 MHz

Enable in `Momentum > Protocols > SubGHz Bypass Region Lock` to unlock whole CC1101 frequency range regardless of country limits.

#### Frequency Range Extension

**Extended range:** 281-361 MHz, 378-481 MHz, and 749-962 MHz

⚠️ **WARNING**: 
1. Only use if you know what you're doing
2. Not needed for most use cases
3. Can damage hardware
4. Developers not responsible for damage

Enable in `Momentum > Protocols > SubGHz Extend Freq Bands`.

### Sub-GHz Frequency Management

#### From Flipper

Manage from `Momentum > Protocols > SubGHz Freqs`:
- Use Defaults: Include default frequency list
- Static Freqs: Used by Read, Read RAW, Frequency Analyzer
- Hopper Freqs: Used by Read > Config > Hopping: ON

#### Default Frequency List

Includes frequencies across all supported bands:
- 300-348 MHz range
- 387-464 MHz range  
- 779-928 MHz range

#### Custom Frequencies

Add to `subghz/assets/setting_user`:
```
Frequency: 928000000
Hopper_frequency: 345000000
```

---

## Appendix

### File Extension Reference

| Extension | Description |
|-----------|-------------|
| `.fam` | Flipper App Manifest |
| `.fap` | Flipper App Package |
| `.ir` | Infrared remote file |
| `.irtest` | Infrared test file |
| `.sub` | Sub-GHz signal file |
| `.rfid` | LF RFID key file |
| `.ibtn` | iButton key file |
| `.nfc` | NFC device file |

### Frequency Bands

| Band | Range (MHz) | Typical Use |
|------|-------------|-------------|
| LPD433 | 433.050-434.790 | General purpose |
| ISM | 315, 433.92, 868, 915 | Various protocols |
| UHF | 300-348, 386-464, 778-928 | Extended range |

### Modulation Types

| Type | Description |
|------|-------------|
| AM650 | Amplitude Modulation 650kHz |
| FM | Frequency Modulation |
| FSK | Frequency Shift Keying |
| OOK | On-Off Keying |

---

*This documentation compilation is based on Flipper Zero official documentation, Momentum Firmware documentation, and community contributions. Last updated: 2026*
