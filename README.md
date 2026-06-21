# Interstate 75 W FM6124 HUB75 Firmware Fix

This repository packages a local firmware fix for Pimoroni Interstate 75 W RP2350 boards driving HUB75 LED matrix panels that use FM6124-style shift register drivers.

The fix was developed against local checkouts of:

- `pimoroni/pimoroni-pico`
- `pimoroni/interstate75`

It includes:

- source patches for both upstream projects
- prebuilt RP2350 UF2 firmware artifacts
- a technical explanation of the alignment issue and why the fix works

## Problem

FM6124 panels showed scanout artifacts, most visibly at the right edge of the display. Testing showed the issue was not caused by display orientation, framebuffer layout, or latch timing.

The HUB75 PIO driver sends one visible logical column as two FIFO words:

```text
word 1 = top half pixel
word 2 = bottom half pixel
```

That means inserting an odd number of dummy words causes a half-column phase error. The successful fix was to prepend one complete blank logical column before each FM6124 row transfer.

Since one logical column is two words, the driver sends two zero words before the normal visible row data.

## Fix Summary

For `PANEL_FM6124`, the HUB75 driver now automatically enables:

```cpp
this->dma_prepend_extra_columns = 1;
```

That changes each row DMA transfer from:

```text
128 visible logical columns * 2 words per column
```

to:

```text
1 blank guard column + 128 visible logical columns
```

The visible image is not shifted because the blank column is extra prepended DMA data, not a replacement for a visible pixel column.

## Files in This Repository

```text
patches/pimoroni-pico-fm6124.patch
patches/interstate75-rp2350-wrapper.patch
firmware/i75w_rp2350-fm6124.uf2
firmware/i75w_rp2350-fm6124-with-filesystem.uf2
TECHNICAL_DETAILS.md
CHECKSUMS.txt
```

## Applying the Patches

Apply the `pimoroni-pico` patch from the root of a `pimoroni-pico` checkout:

```bash
cd pimoroni-pico
git apply ../interstate75w-fm6124-fix/patches/pimoroni-pico-fm6124.patch
```

Apply the Interstate 75 RP2350 wrapper patch from the root of an `interstate75` checkout:

```bash
cd interstate75
git apply ../interstate75w-fm6124-fix/patches/interstate75-rp2350-wrapper.patch
```

The wrapper patch exposes `PANEL_FM6124` and forwards the scanout tuning arguments through the RP2350 `Interstate75` Python class.

## Python Usage

After flashing firmware built with the patch, use:

```python
from interstate75 import Interstate75

i75 = Interstate75(
    display=Interstate75.DISPLAY_INTERSTATE75_128X64,
    panel_type=Interstate75.PANEL_FM6124,
    stb_invert=True,
    dummy_pixels=2,
)
```

No manual `dma_prepend_extra_columns=1` argument is normally required. Selecting `PANEL_FM6124` applies the extra guard column automatically.

## Prebuilt Firmware

Prebuilt UF2 files are included for convenience:

```text
firmware/i75w_rp2350-fm6124.uf2
firmware/i75w_rp2350-fm6124-with-filesystem.uf2
```

Use the `with-filesystem` build if you want the bundled filesystem image included. Flashing firmware may replace the board firmware and can affect files stored on the device, so back up any scripts or data first.

## Technical Details

See [TECHNICAL_DETAILS.md](TECHNICAL_DETAILS.md) for the full explanation, including the `hub75.hpp` before/after changes.

## Status

This is a local working fix packaged for other users to test. It is not an official Pimoroni release.
