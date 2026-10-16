# solo2 on NXP LPC55S69

### the open source FIDO2 security key — built with Trussed®

<a href="https://solokeys.com"><img src="https://solokeys.com/cdn/shop/files/USBCHacker_600x600.jpg?v=1682534858" alt="Solo 2C Hacker" height="220"></a>
&nbsp;&nbsp;
<a href="https://www.nxp.com/products/processors-and-microcontrollers/arm-based-processors-and-mcus/lpc-cortex-m-mcus/lpc5500-cortex-m33/lpcxpresso55s69-development-board:LPC55S69-EVK"><img src="https://compoindia.com/wp-content/uploads/2022/09/LPC55S69-EVK__56222.jpg" alt="LPCXpresso55S69-EVK" height="220"></a>

This is the firmware for the **LPC55S69** — the NXP Cortex-M33 silicon inside
the shipping **[Solo 2](https://solokeys.com)**. It runs on two boards:

- **Solo 2 Hacker** — the real SoloKeys device, unlocked for developers
  (PRINCE-encrypted storage, no debug port). Get one from
  [solokeys.com](https://solokeys.com).
- **LPCXpresso55S69-EVK** — NXP's evaluation board with the same chip and an
  onboard J-Link, for recoverable development. Get one from
  [nxp.com](https://www.nxp.com/products/processors-and-microcontrollers/arm-based-processors-and-mcus/lpc-cortex-m-mcus/lpc5500-cortex-m33/lpcxpresso55s69-development-board:LPC55S69-EVK).

## Build

```
make build-dev      # EVK / development build (plain storage)
make build-release  # Solo 2 Hacker, PRINCE-encrypted (provisioned devices)
```

## Space budget

Measured on `hw/solo2` builds for the EVK board (the Solo 2 board differs by at most
0.4 KiB of flash and 1.5 KiB of RAM), all sizes in KiB. The firmware region ends at the
filesystem boundary, 0x80000, so an image has 512 KiB.

### Flash per feature

What each feature adds to the platform, which is the runner with `admin-app` (222.4):

| Feature | KiB | Mostly |
|---|---|---|
| `p384` | +120.1 | the `p384` crate's fiat-crypto field code (93.3), SHA-384/512 (10.8) |
| `fido-authenticator` | +77.7 | the app, CBOR, ctap-types |
| `rsa` | +66.0 | num-bigint-dig, trussed's allocating mechanisms, der, pkcs8, rsa |
| `panic-record` | +52.8 | every panic location string, which `panic_halt` drops |
| `hacker` | +50.0 | ML-DSA-44 FIDO and the wallet |
| `piv-authenticator` | +48.7 | |
| `opcard` | +43.9 | 11.5 shared with PIV when both are in |
| `oath` | +26.7 | |
| `mldsa44-backend` | +25.2 | |

The release profile is already at `opt-level = "z"`, LTO, one codegen unit and the machine
outliner; `p384` gets no smaller at other opt-levels.

### RAM and stack

flip-link puts the statics at the top of the 272 KiB of RAM and the stack below them, so
every static comes out of the stack, and the stack is what runs out first. `init` builds the
large RTIC resources in their statics and needs 33 to 39 KiB of stack in every image. The
deepest paths are at runtime: idle through apdu-dispatch about 51 KiB, and the trussed
service about 114 KiB when it signs with ML-DSA-44, on top of the app that asked for it.

| Static | KiB |
|---|---|
| RSA heap (`rsa` only); an RSA-4096 operation peaks at 26 | 40.0 |
| volatile store, 156 × 128 B with `rsa`, 78 × 128 B without | 19.6 / 9.75 |
| trussed service | 11.6 |
| USB classes | 11.1 |
| APDU dispatch, its 7609-byte response buffer | 7.5 |
| CTAPHID channel | 7.4 |
| external store and its littlefs state (RAM on the EVK) | 8.2 |

PIV unseals a key into the volatile store for every use. An unsealed RSA-3072 or RSA-4096
key does not fit next to the rest in the 78 blocks of a build without `rsa`, and the card
answers 6400 to the signature or import.

### Images

| Image | Flash | Spare | Stack |
|---|---|---|---|
| stock default | 423.3 | 88.7 | 193.0 |
| stock, `rsa` | 489.8 | 22.2 | 143.0 |
| FIDO, PIV, OpenPGP | 374.2 | 137.8 | ≈197 |
| FIDO, PIV, OpenPGP, `rsa` | 441.0 | 71.0 | 147.3 |
| PIV, OpenPGP, `rsa`, `p384` (no FIDO) | 487.8 | 24.2 | 150.2 |
| PIV, `rsa`, `p384` | 455.2 | 56.8 | ≈150 |
| OpenPGP, `rsa` | 332.1 | 179.9 | ≈150 |
| stock, `p384` | 543.6 | −31.6 | does not fit |

Every image that fits boots. With `rsa` and ML-DSA-44 together, as in the stock image with
`rsa`, the worst case at runtime (idle's 51 KiB, then the service's 114 KiB for an ML-DSA-44
signature) is more than the 143 KiB left; no ML-DSA-44 signature has been tried on such an
image.

The card-only image (`--no-default-features --features
board-lpcxpresso55,no-encrypted-storage,no-pfr,admin-app,opcard,piv-authenticator,rsa,p384`)
passes mind-the-gap's PIV and OpenPGP suites on every algorithm.

### Where room could come from

- Stop apdu-dispatch copying its 7609-byte buffers by value: 20 to 40 KiB of stack.
- Leave `p384` out, or move P-384 to the CASPER accelerator: 120 KiB of flash.
- Leave ML-DSA-44 out of card images: 25 KiB of flash and its 114 KiB signing stack peak.
- RSA on crypto-bigint or CASPER modexp: about 30 KiB of flash and a smaller heap.
