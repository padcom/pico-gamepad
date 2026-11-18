# pico-gamepad

Game controllers for [Pico](https://github.com/padcom/pico-lang): list the pads that are plugged in, open one, and ask it what it says **now**, 50 times a second if you like. A radio that works as a USB joystick (a TX16S in "Joystick (HID)" mode) is a pad.

Status: **Linux first** (evdev). On every other platform a program using the package builds and runs, finds no pads, and `openPad` fails with `UNSUPPORTED`; macOS and Windows backends come later behind the same API.

It is the native piece behind the drone configurator's joystick reader. It doesn't depend on it and works in any Pico program.

## Use

```pico
import { listPads, openPad } from "padcom/pico-gamepad"

const pads = await listPads()                 // joysticks only; nothing is opened
for (const p of pads) {
  println(`${p.name}  ${p.id}  ${p.axisCodes.length} axes, ${p.buttonCodes.length} buttons`)
}

const pad = await openPad(pads[0])            // read only, never grabbed
const s = pad.state()                         // the kernel's state NOW: no event queue, nothing old
println(`${s.axes.join(" ")}  ${s.buttons.filter((down: bool) => down).length} buttons down`)
pad.close()
```

- `listPads(system?)` gives the joysticks that are plugged in, in a stable order, from what the system says of them (on Linux: sysfs). A keyboard, a mouse or a touchpad is never listed, and never opened.
  - `id`: `vendor:product` (4 hex digits each), `#serial` when the device has one, `@port` when two pads would otherwise be the same: no two pads share an id.
  - `legacyId`: the string a browser gives the pad (`name (Vendor: 1209 Product: 4f54)`), for a mapping that was saved by the browser.
  - `axisCodes` / `buttonCodes`: what the pad has, in the order it numbers them.
- `openPad(info, system?)` opens a pad read only. It throws `PadError` (with the `errno` of the call that failed, when there was one): `PERMISSION_DENIED`, `NOT_FOUND`, `NOT_A_PAD` (not a joystick, or codes that are not codes), `CHANGED` (another device is behind the node than the one that was listed), `UNSUPPORTED`, `IO`. `Pad.state()` throws `DISCONNECTED` and `CLOSED`.
- `Pad.state()` reads the pad **now** (it throws `DISCONNECTED` at the very next read after an unplug, and `CLOSED` after `close()`):
  - `raw`: every axis as the device reports it; `axes`: the same from -1 to 1 by the range the device declares (**nothing is calibrated**: the device's own range is the range; its dead zone and noise filter are reported in `Pad.axes` and not applied);
  - `buttons`: which are down;
  - `at`: when it was read, in µs on the monotonic clock. A reading is as old as its `at`: the device's own timestamp only moves when the state changes, so it is never used.
- `Pad.axes[i]`: `code`, `name` (`ABS_X`), `minimum`, `maximum`, `flat`, `fuzz`. Axis `i` is the `i`th ABS code in ascending order, which is how browsers number them; buttons are in joydev's order, which is the one `jstest` and the browsers use.
- `watchPads(handler, options?, system?)` tells what is there at once (`added`), and then every change: `added`, `removed`, `denied`. It looks every 100 ms. A pad that appears and cannot be opened yet is tried again for 2 s before it is `denied`: the desktop's access rule can come a moment after the node does (264 ms on the real radio).

### Access on Linux

The desktop gives the logged-in user the joystick (a `uaccess` ACL); **this package never asks anyone to join the `input` group**, which would also give them every keyboard. A pad that is `denied` is a machine where the desktop does not do that: a udev rule for the radio (`TAG+="uaccess"`) is the way. Only nodes that sysfs says are joysticks are opened.

### Testing what uses it

`FakeSystem` and `FakeDevice` (exported) are a made-up machine: plug a pad in and out, make `open` fail for a while (the ACL lag), set its axes and buttons, unplug it so that the next `state()` says `DISCONNECTED`. A `PadSystem` is two methods (`scan`, `open`), so a test can also bring its own.

## Examples

`examples/pad-monitor.pi`:

```sh
pico run examples/pad-monitor.pi                   # lists the pads
pico run examples/pad-monitor.pi -- 0              # prints the first one, 10 times a second, until it is unplugged
pico run examples/pad-monitor.pi -- --watch        # tells what is plugged in and out
```

## Check on a real radio

The unit tests run without hardware. To check a real device, with the radio in USB "Joystick (HID)" mode:

1. `pico run examples/pad-monitor.pi` lists it (before any stick is moved), with 8 axes and 24 buttons for a TX16S.
2. `pico run examples/pad-monitor.pi -- 0`: every stick, pot and switch moves a bar, **axes 4 and 7 as well** (the browser never shows them), the buttons show their number. Pull the cable: the line `DISCONNECTED: ... no such device (ENODEV)` and the program ends.
3. `pico run examples/pad-monitor.pi -- --watch`: plug the radio in: `added`; pull it out: `removed`.

`spike/` is the program the first measurements on a real TX16S were made with (`spike/RUNSHEET.md`): permissions, identity, the 8 axes, the order of the buttons, the jitter of a 50 Hz poll, the unplug, the lag of the ACL.

## Tests

```sh
pico test .
```

No hardware, no `/dev/input` and no privilege: canned `ioctl` answers (`EVIOCGABS`, `EVIOCGKEY`, ...) over a made-up sysfs and /dev tree for the Linux backend, a fake system for the pad, the watch and the hot-plug, the kernel's own ioctl numbers checked against known values.

## Known limits

- Linux only so far (evdev). joydev (`/dev/input/js*`) is only in `spike/`, for the case that evdev is not reachable; it was not run on a device.
- A pad that is frozen but plugged in looks like sticks that do not move (true of every program that reads one).
- Only `unsigned long` words of 64 bits (a 64-bit kernel) are read from sysfs by default, on a little-endian machine (x86-64, arm64).
- A device that has no button in the joystick or gamepad range (pedals, a device with only `BTN_MISC` buttons) is not taken for a pad: it is not listed, and `openPad` of it is `NOT_A_PAD`.
- "No two pads share an id" holds while the pads differ in vendor, product, serial number or port (`phys`, which every USB device has); two identical devices with neither a serial number nor a `phys` would share one.
- The kernel gives the lowest free event number to the next device, so a node can be another device by the time it is opened: `openPad` asks the open node who it is (id, name, capabilities: never what is pressed) and lets go of it with `CHANGED` (or `NOT_A_PAD`) when it is not the pad that was listed. `watchPads` takes a pad at the same path with another id, serial number or port for a new pad: `removed`, then `added`.
