# G0 run sheet: a radio on Linux (about 10 minutes)

The G0 spike of the joystick reader. Runs A, B and D were done on the real TX16S (findings: evdev works for the desktop user with no
`input` group, all 8 axes live, 24 buttons in joydev's order, a 50 Hz poll holds, `ENODEV` at the next read after an unplug, the ACL comes
264 ms after the node); **run C (joydev) has not been run on a real device.** What is kept here is the tool, for the next radio, the next
machine, or the joydev fallback.

What this is: `evdev-probe.pi` only READS. It lists `/dev/input`, tests whether it may open each node, and for the radio asks the
kernel what it has and its state 50 times a second. No grab, no write, no keystroke is read: a keyboard or mouse node is opened for the
permission test and closed again at once (a node you may not open says `EACCES`, which is also an answer).

Before: the radio in USB mode **Joystick (HID)**, plugged in, `pico` on the PATH, a terminal in the repository's folder. Save each run:
`... 2>&1 | tee g0-a.txt`.

## Run A: permissions, identity, the 8 axes, the buttons (10 s)
```
pico run spike/evdev-probe.pi -- --seconds 10 2>&1 | tee g0-a.txt
```
When it prints `4. polling ...` (about 2 s after the start), for the 10 s: move **every stick to all four ends** and back to the centre,
**every switch through all its positions**, **every pot and slider**, and press a few buttons, **including ones with high numbers**.
Section 5 then says per axis `reached both ends / moved / NEVER MOVED`.

## Run B: unplug (20 s)
```
pico run spike/evdev-probe.pi -- --seconds 20 2>&1 | tee g0-b.txt
```
Pull the USB cable about **5 s after** the polling lines start. Expected: a line `ENDED: EVIOCGABS(...) FAILED ... ENODEV` within one
tick.

## Run C: joydev (`/dev/input/js*`) (10 s) — not run on a device yet
```
pico run spike/evdev-probe.pi -- --joydev --seconds 10 2>&1 | tee g0-c.txt
```
Plug the radio in first, move everything as in A. Section 7 says which ABS code and key code joydev gives every axis and button number,
and whether that is the order `jstest` shows (`same: true/false`).

## Run D: hot-plug (30 s)
Unplug the radio, then:
```
pico run spike/evdev-probe.pi -- --watch 30 2>&1 | tee g0-d.txt
```
Plug the radio in after about 5 s, then pull it out again at about 20 s. Section 9 says when the node appeared, whether `open()` worked
**at once** or only some ms later (the permission can lag behind the node), and when it went.

## Run E (optional): jitter under load
Run A again while the machine is busy (a build, a video). Paste only section 6.

## Paste back
`g0-a.txt`, `g0-b.txt` (sections 4 to 6 are enough), `g0-c.txt`, `g0-d.txt`, and the output of:
```
id
ls -l /dev/input/ /dev/input/by-id/
getfacl /dev/input/js0 /dev/input/event* 2>/dev/null | head -60
jstest --normal /dev/input/js0
```
