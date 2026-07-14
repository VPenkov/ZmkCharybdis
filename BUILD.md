# Building & Flashing the Firmware

This is the Charybdis (Bastardkb) wireless split running [ZMK]. The keyboard is
two `nice_nano_v2` controllers — a **right** half (central: talks to the host
over USB/Bluetooth) and a **left** half (peripheral: relays keystrokes to the
right over Bluetooth).

There are two ways to get firmware:

- **[GitHub Actions](#option-a-github-actions-recommended)** — no local toolchain, builds in the cloud on every push. Recommended.
- **[Local build with `west`](#option-b-local-build-with-west)** — faster iteration, needs the Zephyr toolchain installed.

Either way, the source of truth for *what* gets built is [`build.yaml`](build.yaml).
Today that produces three artifacts:

| Artifact | Shield | Notes |
| --- | --- | --- |
| `charybdis_qwerty_left`  | `charybdis_left`  | Left half (peripheral) |
| `charybdis_qwerty_right` | `charybdis_right` | Right half (central); includes the `studio-rpc-usb-uart` snippet for [ZMK Studio] |
| `firmware_reset_nano_v2` | `settings_reset`  | Wipes stored settings/pairings |

The keymap itself lives in [`config/charybdis.keymap`](config/charybdis.keymap)
(plus `config/*.conf` for build options). Editing that file is all you need to
change key behavior — see the [main README](README.md#modify-key-mappings).

---

## Option A: GitHub Actions (recommended)

1. Push your changes (e.g. an edit to `config/charybdis.keymap`) to GitHub. The
   [`ZMK Firmware` workflow](.github/workflows/build.yml) runs automatically on
   every push, or trigger it manually from the **Actions** tab
   ("Run workflow").
2. Open the completed run and download the artifacts from the **Artifacts**
   section at the bottom of the summary page. You'll get the `.uf2` files listed
   in the table above.
3. Unzip, then [flash](#flashing) each half.

That's the whole loop — no local setup required.

---

## Option B: Local build with `west`

Use this if you want to build without pushing to GitHub.

### Prerequisites

Install the ZMK / Zephyr toolchain by following the official
[ZMK Toolchain Setup] guide (installs `west`, the Zephyr SDK, and the ARM
cross-compiler). On macOS the short version is:

```sh
# Zephyr build dependencies
brew install cmake ninja python3 ccache dtc gperf

# west, in a virtualenv or --user
pip3 install --user west
```

### One-time workspace setup

Initialize a west workspace pointed at this repo's `config/` directory and pull
down ZMK + the trackball driver modules declared in
[`config/west.yml`](config/west.yml):

```sh
cd /path/to/ZmkCharybdis
west init -l config
west update
west zephyr-export
```

> `west update` fetches ZMK from the `feat/pointers-with-input-processors`
> branch plus badjeff's PMW3610 driver and input-relay/listener modules. It can
> take a few minutes the first time.

### Build each half

Run one build per shield. The `-c` clears the previous build so stale config
never leaks between the two halves:

```sh
# Right half (central) — includes ZMK Studio support
west build -p -s zmk/app -b nice_nano_v2 -- \
  -DSHIELD=charybdis_right \
  -DZMK_CONFIG="$(pwd)/config" \
  -DSNIPPET=studio-rpc-usb-uart
cp build/zephyr/zmk.uf2 charybdis_right.uf2

# Left half (peripheral)
west build -p -s zmk/app -b nice_nano_v2 -- \
  -DSHIELD=charybdis_left \
  -DZMK_CONFIG="$(pwd)/config"
cp build/zephyr/zmk.uf2 charybdis_left.uf2
```

Optionally build the settings-reset image (handy to keep around):

```sh
west build -p -s zmk/app -b nice_nano_v2 -- -DSHIELD=settings_reset
cp build/zephyr/zmk.uf2 firmware_reset_nano_v2.uf2
```

You now have `.uf2` files ready to [flash](#flashing).

---

## Flashing

Each `nice_nano_v2` mounts as a USB mass-storage device in bootloader mode; you
flash by copying a `.uf2` onto it.

1. **First time, or when things misbehave:** flash `firmware_reset_nano_v2.uf2`
   to *both* halves first to clear old settings and pairings.
2. Plug the half into USB.
3. **Double-tap the reset button** to enter the bootloader. The controller
   mounts as a drive named `NICENANO`.
4. Copy the matching `.uf2` onto that drive:
   - Right half → `charybdis_right.uf2` (or `charybdis_qwerty_right.uf2`)
   - Left half  → `charybdis_left.uf2` (or `charybdis_qwerty_left.uf2`)
5. The drive unmounts and the controller reboots on its own.
6. Repeat for the other half.

The halves pair automatically after both are flashed. If they don't connect,
tap reset on both at the same time; if that fails, see the ZMK
[connection-issues troubleshooting][ZMK Connection Issues].

> **Tip:** Because the right half is the central, plug *it* into the host (or
> pair it over Bluetooth). The left half only ever talks to the right.

---

## Applying keymap changes

The typical loop after editing `config/charybdis.keymap`:

1. Rebuild — push for [Option A](#option-a-github-actions-recommended) or run
   the `west build` commands in [Option B](#option-b-local-build-with-west).
2. Reflash **both** halves (a keymap lives on both, so flash both to stay in
   sync).

For runtime tweaks without a rebuild, the right-half firmware ships with
[ZMK Studio] support — see [Modify Key Mappings](README.md#modify-key-mappings).

---

## Debugging with USB logging

To see whether a key press actually reaches the firmware (e.g. diagnosing a dead
switch), build a temporary **right-half** logging firmware that streams matrix
events over a USB serial console.

The catch: this board has a *single* USB CDC port, and the normal right build
already uses it for Studio (`snippet: studio-rpc-usb-uart`). USB logging needs
that same port as `zephyr,console`, so you must swap the snippet and turn Studio
off — otherwise the build fails (`zephyr_console` undeclared, then
`zmk,studio-rpc-uart` undeclared). Three temporary changes:

1. **[`build.yaml`](build.yaml)** — in the `charybdis_right` entry, replace
   `snippet: studio-rpc-usb-uart` with `snippet: cdc-acm-console` (Zephyr's
   snippet that creates the CDC node and sets `zephyr,console`).
2. **[`config/charybdis_right.conf`](config/charybdis_right.conf)** — add
   `CONFIG_ZMK_USB_LOGGING=y`.
3. **[`boards/shields/charybdis-bt/charybdis_right.conf`](boards/shields/charybdis-bt/charybdis_right.conf)**
   — set `CONFIG_ZMK_STUDIO=n`. (Its `=y` overrides `config/`, and Studio's UART
   RPC transport won't compile once its snippet is gone.)

Then build, flash the **right** half, and keep it on USB:

```sh
ls /dev/tty.usbmodem*                 # find the CDC console (Studio is off, so just one)
screen /dev/tty.usbmodemXXXX 115200   # or: tio /dev/tty.usbmodemXXXX   (Ctrl-A K to quit screen)
```

ZMK logs each key press/release with its **position** — the 0-based index into
the keymap. Press a known-good key to learn what a real event looks like, then
press the suspect key: no log line = the press never reached the matrix
(hardware — switch/diode/solder), a log line = the matrix read it (look higher
up, at keymap/HID).

**Revert all three changes** once you're done so the normal Studio firmware
returns.

[ZMK]: https://zmk.dev/
[ZMK Studio]: https://zmk.studio/
[ZMK Toolchain Setup]: https://zmk.dev/docs/development/local-toolchain/setup
[ZMK Connection Issues]: https://zmk.dev/docs/troubleshooting/connection-issues
