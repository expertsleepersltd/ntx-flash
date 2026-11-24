# NTX Flash Tool

A unified command-line tool for flashing NTX-8CV firmware. Replaces the need for Python/SPSDK installation.

## Features

- Flash from local ZIP firmware packages
- Download and flash specific firmware versions
- Download and flash the latest firmware
- Puts the module into bootloader mode for you (MIDI SysEx), on macOS, Linux and Windows
- Cross-platform: Linux, macOS, Windows

## Installing a release

Download the artifact for your platform from the repository's
[Releases](https://github.com/expertsleepersltd/ntx-flash/releases) page.

On macOS, open the signed and notarized `.pkg` installer. It installs the
universal executable at `/usr/local/bin/ntx-flash` and prompts for administrator
authorization. Confirm the installation with:

```bash
command -v ntx-flash
ntx-flash --help
```

Linux and Windows releases remain downloadable archives that can be extracted
and run directly.

## Building

### Prerequisites

**Linux:**
```bash
sudo apt-get install libudev-dev libasound2-dev
```
(Fedora: `libudev-devel alsa-lib-devel`, Arch: `systemd-libs alsa-lib`)

**macOS:**
- Xcode Command Line Tools: `xcode-select --install`

**Windows (MinGW-w64):**
- Install MinGW-w64 from https://www.mingw-w64.org/
- Ensure `gcc` and `make` are in your PATH

### Build

```bash
# Clone with submodules
git clone --recursive https://github.com/expertsleepersltd/ntx-flash.git
cd ntx-flash

# Build
make

# Or to build on Windows with MinGW
mingw32-make
```

The build produces a single binary: `ntx-flash` (or `ntx-flash.exe` on Windows).

Maintainers configuring GitHub releases should follow the
[macOS signing and notarization guide](docs/macos-release-signing.md).

## Usage

### Put NTX-8CV in bootloader mode

The tool does this for you. If the module is running normally, it is asked over
MIDI SysEx to reboot into bootloader mode, and the tool waits for it to
re-enumerate before continuing. The module must be connected over USB and powered,
and its MIDI port must be visible to the host.

```bash
ntx-flash --enter-bootloader   # enter bootloader mode and exit
ntx-flash --list-midi-ports    # show MIDI ports, e.g. "NTX-8CV"
```

If that is not possible (no MIDI port, unusual configuration), use the
configuration tool (`Settings` page, `Reboot in bootloader mode` button) or the
bootloader jumper on the module (refer to the user manual).

### Flash from local file

```bash
ntx-flash /path/to/ntx8cv_1.1.0.zip
```

### Download and flash specific version

```bash
ntx-flash --version 1.1.0
```

### Download and flash latest

```bash
ntx-flash --latest
```

### Download from custom URL

```bash
ntx-flash --url https://example.com/firmware.zip
```

### List available versions

```bash
ntx-flash --list
```

### Options

| Option | Description |
|--------|-------------|
| `-v, --verbose` | Show detailed output |
| `-n, --dry-run` | Validate without flashing |
| `-m, --machine` | Machine-readable output (see [MACHINE.md](MACHINE.md)) |
| `--enter-bootloader` | Put the module in bootloader mode and exit |
| `--list-midi-ports` | List MIDI output ports |
| `--midi-port <name>` | MIDI port to send the bootloader request to (default `NTX-8CV`) |
| `--sysex-id <0-127>` | SysEx device ID of the module (default `0`) |
| `--no-auto-bootloader` | Never request bootloader mode automatically |
| `-h, --help` | Show help |

## Firmware Package Format

The tool expects ZIP files containing:
```
ntx8cv_X.Y.Z.zip
├── MANIFEST.json
├── bootable_images/
│   ├── unsigned_MIMXRT1010_flashloader.bin
│   └── NTX-8CV.bin
└── ...
```

Official [NTX-8CV firmware packages from Expert Sleepers](https://www.expert-sleepers.co.uk/ntx8cvfirmwareupdates.html) are fully supported.

## How It Works

The flash process involves two stages:

1. **SDP Mode** (USB 0x1FC9:0x0145): Upload flashloader to RAM and execute it
2. **Bootloader Mode** (USB 0x15A2:0x0073): Configure flash, erase, write firmware, reset

Before stage 1, if the module is not already in a bootloader state, the tool
sends this SysEx message to the module's MIDI port:

```
F0 00 21 27 6A <SysEx ID> 7F 7F F7
```

`00 21 27` is the Expert Sleepers manufacturer ID, `6A` addresses the NTX-8CV,
and `7F 7F` means "reboot into bootloader mode". The same message is sent by the
`Reboot in bootloader mode` button in the web configuration tool. MIDI output is
implemented without extra libraries: CoreMIDI on macOS, the ALSA sequencer on
Linux, and WinMM on Windows.

## Troubleshooting

### Device not found

- Check the module is powered and connected over USB
- Run `ntx-flash --list-midi-ports` - the module should appear as `NTX-8CV` when it
  is running normally. If it does not, the tool cannot request bootloader mode;
  use the configuration tool or the jumper instead
- If the module uses a non-default SysEx device ID, pass `--sysex-id <n>`
- On Linux, you may need udev rules. Create `/etc/udev/rules.d/99-ntx-8cv.rules`:
  ```
  SUBSYSTEM=="usb", ATTR{idVendor}=="1fc9", ATTR{idProduct}=="0145", MODE="0666"
  SUBSYSTEM=="usb", ATTR{idVendor}=="15a2", ATTR{idProduct}=="0073", MODE="0666"
  ```
  Then: `sudo udevadm control --reload-rules`

### Permission denied (Linux)

Run with sudo or set up udev rules as above.

## License

This tool incorporates:
- BLFWK from NXP (BSD-3-Clause)
- miniz (Public Domain)
- cJSON (MIT)

## Credits

- [Expert Sleepers](https://www.expert-sleepers.co.uk/) for the NTX-8CV
- [NXP Semiconductors](https://www.nxp.com/) for BLFWK/SPSDK
- [apexrtos/nxp_blhost_sdphost](https://github.com/apexrtos/nxp_blhost_sdphost) for the portable build setup
