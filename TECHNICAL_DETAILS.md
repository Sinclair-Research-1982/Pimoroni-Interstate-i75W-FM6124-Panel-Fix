# Technical Details

## Root Cause

The display issue was a per-row data alignment problem in the HUB75 scanout path.

For this PIO driver, one visible logical column is two FIFO words:

```text
word 1 = top half pixel
word 2 = bottom half pixel
```

Odd dummy counts were bad because they moved the PIO into a half-column phase error. The stable fix was to add one full blank logical column pair before each FM6124 row transfer while still sending all real visible columns.

## Main Driver Change

The key behavior is in `drivers/hub75/hub75.cpp`.

For `PANEL_FM6124`, if no manual DMA alignment override is supplied, the firmware enables:

```cpp
this->dma_prepend_extra_columns = 1;
```

The row transfer length is increased:

```cpp
dma_transfer_words = (width + this->dma_prepend_extra_columns) * 2;
```

The row DMA buffer is built by prepending blank words and then copying the normal row data:

```cpp
std::fill_n(dma_row_buffer, prepend_words, Pixel(0));
std::copy_n(source, width * 2, dma_row_buffer + prepend_words);
```

So the panel receives two zero words first, then the normal visible row data.

## Why This Worked

Testing showed:

- `dummy_pixels=2` was stable but had right-edge artifacts.
- `dummy_pixels=1` and `dummy_pixels=3` caused half-column phase problems.
- `dma_prepend_columns=1` cleaned the right edge but shifted the image right.
- `dma_advance_columns=1` shifted the image left.
- `dma_prepend_extra_columns=1` cleaned the right edge without moving the visible image.

That final result showed the panel needed an extra guard clocked column, not a visible-coordinate shift.

## `hub75.hpp` Before and After

### Panel Type Constants

Before:

```cpp
enum PanelType {
    PANEL_GENERIC = 0,
    PANEL_FM6126A,
};
```

After:

```cpp
enum PanelType {
    PANEL_GENERIC = 0,
    PANEL_FM6126A,
    PANEL_FM6124DJ,
    PANEL_FM6124,
};
```

### Buffer Fields

Before:

```cpp
Pixel *back_buffer;
bool managed_buffer = false;
```

After:

```cpp
Pixel *back_buffer;
Pixel *managed_back_buffer = nullptr;
Pixel *dma_row_buffer = nullptr;
bool managed_buffer = false;
```

`dma_row_buffer` allows the driver to build a row that is wider than the visible framebuffer:

```text
blank guard column + visible row data
```

### Timing and DMA Alignment Fields

Before:

```cpp
uint brightness = 6;
```

After:

```cpp
uint brightness = 6;
uint dummy_pixels = 2;
uint latch_cycles = 0;
uint dma_prepend_columns = 0;
uint dma_advance_columns = 0;
uint dma_prepend_extra_columns = 0;
uint dma_transfer_words = 0;
```

The final important field is `dma_prepend_extra_columns`. The other fields were useful diagnostic knobs during testing.

### Constructor Overload

Before:

```cpp
Hub75(uint width, uint height, Pixel *buffer) : Hub75(width, height, buffer, PANEL_GENERIC) {};
Hub75(uint width, uint height, Pixel *buffer, PanelType panel_type) : Hub75(width, height, buffer, panel_type, false) {};
Hub75(uint width, uint height, Pixel *buffer, PanelType panel_type, bool inverted_stb, COLOR_ORDER color_order=COLOR_ORDER::RGB);
~Hub75();
```

After:

```cpp
Hub75(uint width, uint height, Pixel *buffer) : Hub75(width, height, buffer, PANEL_GENERIC) {};
Hub75(uint width, uint height, Pixel *buffer, PanelType panel_type) : Hub75(width, height, buffer, panel_type, false) {};
Hub75(uint width, uint height, Pixel *buffer, PanelType panel_type, bool inverted_stb, COLOR_ORDER color_order=COLOR_ORDER::RGB);
Hub75(uint width, uint height, Pixel *buffer, PanelType panel_type, bool inverted_stb, COLOR_ORDER color_order, uint dummy_pixels, uint latch_cycles, uint dma_prepend_columns, uint dma_advance_columns = 0, uint dma_prepend_extra_columns = 0);
~Hub75();
```

### FM6124 Setup Method

Before:

```cpp
void FM6126A_write_register(uint16_t value, uint8_t position);
void FM6126A_setup();
void set_color(uint x, uint y, Pixel c);
```

After:

```cpp
void FM6126A_write_register(uint16_t value, uint8_t position);
void FM6126A_setup();
void FM6124DJ_setup();
void set_color(uint x, uint y, Pixel c);
```

`FM6124DJ_setup()` does not need to send special initialization data in the final implementation, but it makes the panel-specific path explicit.

### DMA Row Helper

Before:

```cpp
void start(irq_handler_t handler);
void stop(irq_handler_t handler);
void dma_complete();
void update(PicoGraphics *graphics);
```

After:

```cpp
void start(irq_handler_t handler);
void stop(irq_handler_t handler);
void dma_complete();
void start_dma_for_row();
void update(PicoGraphics *graphics);
```

`start_dma_for_row()` prepares each row consistently. It either DMAs directly from the framebuffer or from the temporary row buffer with the FM6124 guard column added.

## MicroPython Binding Changes

The MicroPython `hub75` module exposes:

```python
hub75.PANEL_FM6124DJ
hub75.PANEL_FM6124
```

The `Hub75(...)` constructor accepts:

```python
dummy_pixels
latch_cycles
dma_prepend_columns
dma_advance_columns
dma_prepend_extra_columns
```

The Interstate 75 wrapper forwards these arguments so user scripts can select the panel type directly.

## Final Result

The firmware treats FM6124 panels as requiring one extra blank logical column at the start of every row DMA transfer. This preserves all visible pixel data, avoids horizontal image shifting, and fixes the right-edge corruption caused by the FM6124 scanout alignment requirement.
