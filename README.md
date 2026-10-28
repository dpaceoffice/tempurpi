# TempurPi

A Raspberry Pi 5 that carries TV audio from the optical output to the massage transducers in a
Tempur-Pedic Ergo ProSmart base, and powers the bed-side USB DAC only while the TV is actually
playing something.

```
TV optical out
  -> HiFiBerry Digi+ I/O (WM8804 S/PDIF receiver, ALSA card sndrpihifiberry)
  -> CamillaDSP (tempur-dsp.service)
  -> 32 ft active USB extension -> "AB13X USB Audio" dongle (001f:0b26, ALSA card "Audio")
  -> 3.5 mm -> aux splitter -> bed
```

The bed's own Bluetooth receiver shares the same aux splitter. While the USB DAC has power, the
bed's massage mode does not work, so `tempur-usb-power` cuts the DAC's 5V whenever the TV has
been silent for a few seconds and restores it when audio returns.

The transducers carry the low end only; a separate TV soundbar plays the highs. CamillaDSP runs
a bass-only limiter that keeps the transducers from distorting on the loudest bass drops.

## Shopping list

The parts this build uses, with US list prices (October 2026) where the vendor publishes one.

| Item | This build | Notes | Price |
|---|---|---|---|
| Raspberry Pi 5 | [Raspberry Pi 5](https://www.raspberrypi.com/products/raspberry-pi-5/), 16 GB | 2 GB is enough: the whole stack, desktop session included, uses under 1 GB. | 2 GB $77.50, 4 GB $110, 16 GB $305 |
| Power supply | Raspberry Pi 27W USB-C Power Supply | Use a supply rated 3 A or more; the official 27W (5 A) supply is the safe pick. On a supply that negotiated only 900 mA, this Pi sagged to 4.6-4.8 V and logged undervoltage whenever the DAC was on, and undervoltage can corrupt the SD card. On a 3 A supply it holds 5.0 V with the DAC playing. | $12-15 |
| Cooler | [Raspberry Pi 5 Active Cooler](https://www.amazon.com/dp/B0CW164TCW) | | |
| microSD card | [SanDisk Extreme 128 GB](https://www.amazon.com/dp/B0G8LLXFJH) (SDSQXH9-128G-GZ6MA) | Any 16 GB or larger card works. | |
| S/PDIF input HAT | [HiFiBerry Digi+ I/O](https://www.hifiberry.com/shop/boards/hifiberry-digi-io/) | WM8804 with optical and electrical inputs, needed for the TV's optical output. Output-only Digi boards will not work. Uses `dtoverlay=hifiberry-digi`. Another ALSA capture device works too: set `CAPTURE_DEV` and the capture `device` in the DSP configs. | $54.90 |
| Case | [HiFiBerry Steel case Pi5](https://www.hifiberry.com/shop/cases/steelcase-pi5/) | | $29.90 |
| Optical cable | TOSLINK, any length | TV optical out to the Digi+ I/O optical in. | |
| USB audio dongle | [USB-A to 3.5 mm audio adapter](https://www.amazon.com/dp/B099FLWJD3) | AB13X chip, reports as "AB13X USB Audio" (USB ID 001f:0b26). It resets itself once after power-on, which adds about 3.5 s before sound. Another class-compliant dongle that accepts 48 kHz S16_LE will work: put its ALSA card name in `DAC_CARD`, `tempur.yml` and the WirePlumber rule. | |
| USB extension | [BlueRigger active USB 2.0 extension, 32 ft](https://www.amazon.com/dp/B005LJKEXS) | Runs from the Pi to the bed. The cable has a hub chip inside (shows up as a Prolific 067b:2586 "USB 2.0 Hub"). It is bus-powered, so cutting VBUS powers down the cable and the dongle together. | |
| Aux cable | 3.5 mm stereo, male to male (Walmart) | Dongle to splitter. | |
| Aux splitter | 3.5 mm stereo splitter (Walmart) | Shares the bed's aux input between the Pi and the bed's Bluetooth receiver. | |

Software: Debian 13 (trixie) with Raspberry Pi OS packages, CamillaDSP 4.1.3.

## Files

| Repo path | Installs to | Purpose |
|---|---|---|
| `bin/tempur-usb-power` | `/usr/local/sbin/` | Watcher that switches the USB 5V and starts/stops the DSP |
| `systemd/tempur-usb-power.service` | `/etc/systemd/system/` | Runs the watcher as root; holds the tunables |
| `systemd/tempur-dsp.service` | `/etc/systemd/system/` | CamillaDSP, optical in to USB DAC out |
| `systemd/tempur-dsp.service.d/websocket.conf` | `/etc/systemd/system/tempur-dsp.service.d/` | Opens the CamillaDSP websocket on 127.0.0.1:1234 for the watcher; retries the DAC volume step while the card settles |
| `systemd/tempur-dsp.service.d/recovery.conf` | `/etc/systemd/system/tempur-dsp.service.d/` | Disables the start rate limit so the DSP always restarts |
| `camilladsp/tempur.yml` | `~/.config/camilladsp/` | The deployed config: 48 kHz, S24 capture from the Digi+ I/O, S16 playback to the dongle, async resampler for clock drift, bass limiter (see below) |
| `camilladsp/tempur-plain.yml` | `~/.config/camilladsp/tempur.yml` (optional) | Same devices, no processing |
| `camilladsp/tempur-lowpass.yml` | `~/.config/camilladsp/tempur.yml` (optional) | Bass-only output with a -1 dBFS ceiling (see below) |
| `wireplumber/50-tempur-dac-ignore.conf` | `~/.config/wireplumber/wireplumber.conf.d/` | Keeps PipeWire from claiming the dongle, which CamillaDSP opens as `hw:Audio` |

The units run CamillaDSP as user `david` with the config under `/home/david`. Change `User=` and
the config path in both `tempur-dsp.service` and `websocket.conf` to match your system.

## How the power switching works

**Cutting 5V on a Pi 5.** Disabling the port in sysfs, hub port power control and USB
autosuspend all leave 5V on the Pi 5's USB ports. The RP1 GPIO line `USB_VBUS_EN`
(gpiochip0 line 42) gates VBUS for all four USB ports:

```
gpioset -z USB_VBUS_EN=0   # 5V off
gpioset -z USB_VBUS_EN=1   # 5V on
```

This switches every USB port on the Pi, so nothing else that needs power can live on USB.

**Detecting audio.** In standby the TV keeps the optical carrier running and sends digital
zeros, so the WM8804 stays locked and lock status says nothing about whether audio is playing.
The watcher looks at sample values:

- DAC on: CamillaDSP owns the capture device, so the watcher asks it over the websocket for
  `GetCaptureSignalPeakSince`. After `OFF_DELAY` seconds below `SIGNAL_DB` it stops the DSP
  and cuts 5V.
- DAC off: the watcher holds the capture device open with `arecord` and waits for the first
  sample above `SIGNAL_ABS`.

**The dongle's self-reset.** The AB13X dongle enumerates about 1 s after power-on, drops off the
bus, and re-enumerates about 3.4 s later. The watcher waits for that cycle (up to `RESET_WAIT`
seconds) before starting CamillaDSP, otherwise the DSP's first start fails with "No such
device". From the first non-zero sample to sound at the bed takes about 5-6 s. A dongle without
this quirk would get there in about 2 s.

## Install

1. In `/boot/firmware/config.txt`:
   ```
   dtparam=audio=off
   dtoverlay=hifiberry-digi
   ```
2. Packages and CamillaDSP:
   ```
   sudo apt install python3-numpy gpiod alsa-utils
   ```
   Put the CamillaDSP binary (4.x, from https://github.com/HEnquist/camilladsp/releases) at
   `/usr/local/bin/camilladsp`.
3. Files:
   ```
   sudo install -m 755 bin/tempur-usb-power /usr/local/sbin/
   sudo install -m 644 systemd/tempur-dsp.service systemd/tempur-usb-power.service /etc/systemd/system/
   sudo install -D -m 644 -t /etc/systemd/system/tempur-dsp.service.d systemd/tempur-dsp.service.d/*.conf
   install -D -m 644 camilladsp/tempur.yml ~/.config/camilladsp/tempur.yml
   install -D -m 644 -t ~/.config/wireplumber/wireplumber.conf.d wireplumber/50-tempur-dac-ignore.conf
   sudo systemctl daemon-reload
   sudo systemctl enable --now tempur-dsp.service tempur-usb-power.service
   ```

**Read-only root.** This Pi runs with the raspi-config overlay filesystem because the SD card
kept corrupting. Changes to the running system vanish on reboot. To install onto the persistent
layer without turning the overlay off, remount it read-write, copy the files under
`/media/root-ro`, and enable the units inside it:

```
sudo mount -o remount,rw /media/root-ro
# copy the files to the same paths under /media/root-ro, then:
sudo chroot /media/root-ro systemctl enable tempur-dsp.service tempur-usb-power.service
sudo mount -o remount,ro /media/root-ro
```

If the last command reports the mount point busy, reboot to bring the layer back read-only.

## Tunables

Set as `Environment=` lines in `tempur-usb-power.service`.

| Variable | Unit value | Script default | Meaning |
|---|---|---|---|
| `OFF_DELAY` | 5 | 60 | Seconds of silence before 5V is cut |
| `POLL` | 1 | 2 | Seconds between peak checks while on |
| `SIGNAL_DB` | -80 | -80 | CamillaDSP capture peak (dBFS) that counts as audio |
| `SIGNAL_ABS` | 8 | 8 | Raw 16-bit sample magnitude that counts as audio while off |
| `RESET_WAIT` | | 5 | Seconds to wait for the dongle's self-reset before starting the DSP |
| `ON_WAIT` | | 15 | Seconds to wait for the dongle to enumerate after 5V on |
| `CAPTURE_DEV` | | `hw:sndrpihifiberry` | Capture device |
| `DAC_CARD` | | `Audio` | ALSA card name of the USB DAC |
| `DSP_UNIT` | | `tempur-dsp.service` | Unit the watcher starts and stops |
| `WS_HOST`, `WS_PORT` | | `127.0.0.1`, `1234` | CamillaDSP websocket |
| `VBUS_LINE` | | `USB_VBUS_EN` | GPIO line name for VBUS |

`tempur-usb-power --probe` prints whether the DAC is present, whether the DSP is running, and
the recent capture peak, then exits.

## Bass limiter

The ProSmart's massage transducers distort on the loudest bass drops. Tempur does not publish
power ratings for the soundbar,
its amplifier or the transducers (the owner's manual rates only the base's 29 V DC, 2 A supply),
so the threshold is set by ear. `camilladsp/tempur.yml` splits each channel at 100 Hz, limits
the low band only, and sums the bands back:

| Stage | Setting |
|---|---|
| Crossover | Linkwitz-Riley, 100 Hz, 4th order. The two halves sum to a flat magnitude response. |
| Low band | Compressor: threshold -15 dBFS, ratio 10:1, attack 10 ms, release 0.3 s, no makeup gain, stereo-linked |
| High band | Untouched |

Measured on the Pi with CamillaDSP 4.1.3, feeding test signals through the pipeline from a file:

| Input | Change |
|---|---|
| 1 kHz tone, 0 dBFS | 0.0 dB |
| Full-band noise peaking at 0 dBFS | 0.0 dB |
| 60 Hz tone, -30 dBFS | 0.0 dB |
| 60 Hz tone, 0 dBFS | -12.5 dB |
| 20 Hz tone, 0 dBFS | -18.0 dB |

On TV audio it takes up to about 6 dB off the loudest bass passages and leaves the rest alone. The
crossover's phase shift can raise peaks slightly (0.9 dB on full-band noise); the TV here peaks
around -3 dBFS, so nothing clips. CamillaDSP's websocket reports clipped samples
(`GetClippedSamples`) if yours runs hotter.

To tune it, change `threshold` (lower means more limiting) and reload without interrupting
playback:

```
pkill -HUP -x camilladsp
```

`camilladsp/tempur-plain.yml` is the same config with no processing.

## Optional: bass-only output

`camilladsp/tempur-lowpass.yml` sends only bass to the bed, with an output ceiling and without
the bass limiter.

| Stage | Setting | Purpose |
|---|---|---|
| Lowpass | Butterworth, 120 Hz, 4th order | Bass only |
| Ceiling | Hard limit at -1 dBFS | Keeps filter overshoot from pushing the output past full scale |

A lowpass can overshoot full scale (a filtered full-scale square wave peaks about 2 dB over).
The ceiling holds the output below full scale, and the extra 1 dB covers inter-sample peaks in
the dongle's DAC. Nothing below -1 dBFS is touched. There is no highpass: the ProSmart's
massage transducers run at 28 and 40 Hz.

Measured on the Pi with CamillaDSP 4.1.3, feeding test signals through this pipeline from a file:

- The highest output sample across all tests is -1.00 dBFS.
- Full-band white noise peaking at 0 dBFS comes out peaking at -7.5 dBFS and never reaches the
  ceiling.
- A 60 Hz tone at -30 dBFS passes through unchanged.
- A full-scale 1 kHz tone comes out at -77 dBFS RMS.
- Only full-scale bass test tones reach the ceiling, and lose 1 dB.

To use it:

```
cp camilladsp/tempur-lowpass.yml ~/.config/camilladsp/tempur.yml
camilladsp --check ~/.config/camilladsp/tempur.yml
sudo systemctl restart tempur-dsp.service
```

Move `freq` to change how much of the sound reaches the bed. The watcher measures the capture
side, before the filter, so neither stage changes when the DAC powers on or off.

## Pi 5 notes

- `vcgencmd get_throttled` returns `0x0` when the Pi has not seen undervoltage or throttling
  since boot; `vcgencmd pmic_read_adc EXT5V_V` reads the 5V input.
- Reading `/sys/kernel/debug/regmap/dummy-avs-monitor@*/registers` hard-locks the Pi 5 and needs
  a power cycle. Do not loop over `/sys/kernel/debug/regmap/*`; read the WM8804 entry
  (`1-003b`) by name.
- WM8804 lock status is register `0x0c` bit 6 (UNLOCK) in that regmap. It is only live while a
  capture stream is open, because DAPM powers the receiver down otherwise.
