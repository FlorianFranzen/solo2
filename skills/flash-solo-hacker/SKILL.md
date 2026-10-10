---
name: flash-solo-hacker
description: Use this before writing any firmware to a physical Solo 2 Hacker key (the unlocked retail key, not the EVK) - how the image gets written through the ROM bootloader, which checks to run first (lock hash, secure boot, PRINCE keystore, the image header), what actually bricks the key and which of those states are still recoverable over USB, and the CFPA version-counter consequence of custom builds. Use it whenever a task says Hacker, solo2 app admin maintenance, lpc55 write-flash, bootloader mode, brick or unbrick; the EVK has its own skill (lpc55-evk) and is where every build is proven first.
---

# Solo 2 Hacker: flashing without bricking, and what recovery exists

Established from this repository, lpc55-hal, lpc55-host, the solo2 CLI, SoloKeys' developer
page and solo2-cli issue #65, and NXP's LPC55S6x errata, on 2026-10-10. Statements marked *unverified* could not be settled from those sources.

## The one fact that governs everything

The Hacker has no reachable ISP pin and no debug header. The ROM's ISP pin is PIO0_5; on
the Solo 2 it is not exposed (nickray in solo2-cli#65: "it's not exposed. You can attach a
Tag-Connect at the bottom"). The user button is a charge-transfer touch sensor read by the
firmware, not by the ROM. So:

- The ROM enters its USB HID bootloader (VID:PID `1209:b000`, LED off) only when the image
  at address 0 is invalid (erased page 0, or an initial stack pointer outside SRAM), or when
  running firmware jumps there itself.
- If the ROM accepts the image and the firmware dies before it reaches its own bootloader
  entry, nothing over USB brings it back. Recovery is then SWD through a Tag-Connect pad
  (J-Link `loadfile app.bin 0x0`), which is fiddly but documented by SoloKeys.

An LPCXpresso55S69 is such a probe: its on-board J-Link can drive an external target through
the board's SWD header once the jumpers isolate the on-board chip (board manual UM11158; the
jumper names were not verified here), with a TC2030-CTX-NL Tag-Connect cable (the three longer pins clipped) to the pads. The
restore image for that path is the plain program-flash backup taken before the flash (SoloKeys'
SB2 downloads are encrypted with the key's own SBKEK and cannot be flashed over SWD).

Every flash also writes off every credential on the key: the filesystem is PRINCE ciphertext
under PUF-held keys, not backupable over ISP, and the new firmware reformats or migrates it.

Everything below exists to keep the key on the recoverable side of that line. The whole
procedure was run on key 56EAA88D on 2026-10-10 (secure boot off, image written, suites green);
the record is `~/Gli.al/solo2-key-backups/56EAA88D.../PLAN-flash-2026-10-10.md`.

## How the image is written

1. `solo2 app admin maintenance`: admin vendor command 0x51, needs a tap within 15 s, then
   the firmware calls `hal::boot_to_bootrom()`: interrupts off, peripherals reset, the IOCON
   invert bit set on PIO0_5 so the ROM believes the ISP pin is asserted, jump to ROM. Flash
   is untouched. (The destructive variant that erases page 0 is never sent by the CLI.)
2. `lpc55 ls` must show `Bootloader { vid: 1209, pid: B000, uuid: <same uuid> }`.
3. `lpc55 write-flash app-hacker.bin`: address 0, pads to 512, erases exactly the image
   range, writes. CFPA, CMPA, keystore and the filesystem (from 0x8_0000) are not touched.
4. `lpc55 reboot`, then `solo2 list` shows `1209:beee`, the new version, `unlocked`.

The ROM does not check length, CRC or signature on a plain type-0 image when secure boot is
off; it boots whatever is there. That is why the image check below matters.

## Pre-flight, every time

**Build**: `board-solo2,hacker` only (`make -C runners/lpc55 build-hacker`, or the lpc55-evk
skill's cargo line with those features and `llvm-objcopy`). Never `board-lpcxpresso55`
(reassigns the button pin: SoloKeys say unrecoverable), never `no-encrypted-storage`,
`no-pfr`, `test-up-control` or `log-defmt` on a Hacker. Prove the same source on the EVK
first (`board-lpcxpresso55,no-encrypted-storage,test-up-control,no-reset-time-window,hacker,
no-pfr`), which exercises the same code with a J-Link behind it.

**Image**: `od -A x -t x4 -N 0x40 app-hacker.bin`. Word 0 (SP) must be `2002xxxx` (SRAM;
errata ROM.7: an SP at or above 0x20040000 sends the ROM into ISP instead of booting). Words
at 0x20, 0x24, 0x28 and 0x34 must be zero (plain image type 0). Size below 0x80000.

**Key**, with the key plugged in and before entering the bootloader:

```bash
solo2 list                      # must say unlocked
solo2 app admin maintenance     # tap; then:
lpc55 ls                        # 1209:B000 with the same UUID
lpc55 pfr yaml > pfr-<uuid>-<date>.yaml   # read-only backup, keep it
lpc55 read-memory 0 512 -o hdr.bin      # decimal arguments; imageLength at 0x20, type at 0x24
lpc55 read-memory 0 <ceil(imageLength/512)*512> -o flash-<uuid>-<date>.bin   # erased pages past
                                        # the image raise ECC errors and abort the whole read
lpc55 read-memory 648192 512 -o cmpa-<uuid>-<date>.bin  # the live CMPA page (0x9E400; 647168 is
                                        # the CFPA ping page, which reads back as 56 bytes repeated)
```

In the PFR dump check (only the CFPA scratch page is a faithful image in `lpc55 pfr` output; the
ping and pong pages come back as their first 56 bytes repeated, so read IVs and counters from
scratch):

- `factory-settings.sha256-hash` is absent or all zero. That hash is the seal. `lpc55 pfr`
  never prints a `seal` field (it parses with `seal: false` and omits defaults), so do not
  look for one. A non-zero hash means a Secure key: stop.
- `secure-boot-enabled` must be false before a write-flash. Retail Hackers ship with it true
  and the CMPA unsealed (SoloKeys' developer page; confirmed on key 56EAA88D on 2026-10-10):
  the ROM then boots only signed images and a plain write-flash leaves it sitting in the ROM.
  Turning it off is a one-time CMPA write, done like this and only like this: take the `factory-settings` block of the key's own `lpc55 pfr
  yaml`, flip `secure-boot-enabled` to false and nothing else, generate both pages offline with
  `lpc55 configure factory-settings --output <file.bin> <file.yaml>`, check that they differ in
  exactly one byte (offset 0x1F, 0xC0 to 0x00: the SEC_BOOT_EN bits of SECURE_BOOT_CFG at
  0x1C) and that the seal hash at the end stays zero, read the live page with `lpc55
  read-memory 648192 512 -o live.bin` (0x9E400) and require it to equal the page generated
  from the unchanged settings, then `lpc55 configure factory-settings -vvv <no-secure-boot.yaml>`,
  read the page back, `lpc55 reboot`. The official signed image still boots with secure boot
  off (observed on key 56EAA88D, ROM K3.0.0, 2026-10-10); if a key stays in the ROM instead,
  write the saved page back or flash the plain image. NXP AN12283 step 10 is the basis: the CMPA can be rewritten over
  ISP until it is sealed (not on 0A silicon). The byte comparison is mandatory because
  lpc55-host does not round-trip the page faithfully: it zeroes 0x30..0x4F and 0x70..0xFF,
  widens two-bit fields to 0b11 and mis-serializes prince-configuration; a cross-check is to
  patch the live page directly (`dd` the one byte at offset 31) and require it to equal the
  generated page. `lpc55 info` lists the ROM's commands off by one (lpc55-host uses bit = tag,
  NXP bit = tag - 1): WriteMemory is available even though the list does not show it.
- `boot-configuration.mode: Usb` (not fall-through disabled), debug access not closed.
- PRINCE provisioned: keystore `header` 0x95959595 with a non-empty `activation-code` and
  `prince-region-2`, and customer `prince-ivs[2]` non-zero. The `board-solo2` build asserts
  exactly that (`validate_cfpa`) BEFORE the button check, so a PRINCE build on an
  unprovisioned key panics every boot and is unrecoverable over USB. (CMPA
  `prince-configuration` reads zero even when provisioned; do not use it as the test.)

**Version counter**: the firmware writes its own version into the CFPA's monotonic
`secure_fw_version`/`ns_fw_version` on first boot, encoded `(major << 22) | (minor << 6) |
patch`. A workspace version above the key's current one raises the counter; above every SoloKeys
release (latest 2.964.0) it makes the key refuse older signed releases (the CLI's
rollback check, and the ROM's own check on signed images); whether `lpc55 configure
customer-settings --overwrite` can lower it again is *unverified*. Decide this before
flashing: set the workspace version equal to the version the key reports (`solo2 list`,
release 2:20220822.0 is 2.964.0), then the counters are left untouched; anything higher is a
one-way step.

## What actually bricks (and what does not)

| Situation | ROM ISP reachable afterwards? | Recovery |
|---|---|---|
| Write aborted mid-way (unplugged) | yes, page 0 is erased first | `lpc55 write-flash` again |
| PRINCE build on an unprovisioned key | no (asserts before the button check) | SWD only |
| EVK build (`board-lpcxpresso55`) on a Hacker | no | SWD only |
| Early panic/hang before `initialize_basic` finishes (clock, PFR/FFR, CFPA read) | no | SWD only |
| Panic after the button check (filesystem mount, NFC, USB init) | via the 5 s three-button hold at plug-in | hold, then write-flash |
| Plain (`no-encrypted-storage`) build on a PRINCE key | boots, reads ciphertext, formats the FS: data loss, *bricking unverified* | reflash a PRINCE build, filesystem is gone |
| Secure boot still enabled in CMPA | ROM stays in ISP, image never boots | disable secure boot (CMPA write) or flash a signed image |
| Any `lpc55 configure`/`provision`/`keystore` write with wrong values, or `seal: true` | depends; sealing and closing SWD are permanent | none for the permanent ones |

The firmware's own software path back: at power-up, `initialize_basic` runs PFR enable and
CFPA validation, then samples the three touch buttons for 5 s; if all are held the whole
time it blinks and jumps to the ROM. It runs before the filesystem, NFC and USB come up, so
anything that dies later is still reachable this way. It does not run on NFC-only power.

`no-reset-time-window` is unrelated to all of this (it only drops the 10 s window on CTAP2
reset); `no-buttons` auto-approves every user-presence check so `maintenance` works
unattended (the DEV build), it does not touch bootloader entry. `format-filesystem` is not a
recovery tool: an unmountable filesystem is formatted automatically.

## Recovery decision tree

- `lsusb` shows `1209:b000`: the key is in ISP. `lpc55 write-flash` a known-good
  `app-hacker.bin`, `lpc55 reboot`.
- Shows `1209:beee` but the app misbehaves: `solo2 app admin maintenance` (tap), reflash.
  If the admin app does not answer: unplug, hold all three touch areas, plug in, keep holding
  5 s until the LED blinks, release, `lpc55 ls`.
- Shows neither, LED off: try the 5 s hold once. If `lpc55 ls` stays empty, the ROM booted
  the image and the firmware died before the button check: SWD (Tag-Connect, J-Link
  `loadfile ... 0x0` with the key simply plugged in) is the only way. No USB-only recovery.
- In `b000` but a correct image will not boot: it is a PFR question (`secure-boot-enabled`,
  CFPA counters), not an image question. Read `lpc55 pfr` before changing anything.

## Not established

The ROM's complete validity checks for a type-0 image (UM11126 is login-gated); whether
errata ROM.5 (fallback failure with a partially erased image) can bite with secure boot off
(the errata conditions it on secure boot being on); whether the ROM enforces the CFPA
firmware version on plain write-flash images (it does on signed ones); whether a plain build
on a PRINCE key really boots (code says yes, the previous version of this skill said brick, SoloKeys recommend
exactly that build for Hackers); the "bad shard/storage build bricked a Hacker" sentence in
the previous version of this skill has no issue, commit or log behind it.
