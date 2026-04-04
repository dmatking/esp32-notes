# ESP32 Hardware Notes

Practical notes on ESP32 variants — initialization quirks, peripheral setup, and things learned the hard way. Board-specific details (pin assignments, etc.) live in each project's README; this covers chip/subsystem-level knowledge that applies across boards.

---

## ESP32-P4 (with ESP32-C6 Co-processor)

Boards like the Waveshare ESP32-P4-WIFI6-Touch-LCD-4B pair an ESP32-P4 application processor with an ESP32-C6 co-processor that handles WiFi and Bluetooth via SDIO using the `esp_hosted` library.

> **New to the P4+C6 combo?** See [p4-c6-wifi-starter](https://github.com/dmatking/p4-c6-wifi-starter) — a minimal, heavily-annotated WiFi example that walks through the architecture and init sequence step by step.

### Radio Initialization Order

The initialization sequence is strict — `esp_hosted` must be set up before anything else touches the network stack:

```c
esp_hosted_init();
esp_hosted_connect_to_slave();   // establish SDIO link to C6
nvs_flash_init();
esp_netif_init();
esp_event_loop_create_default();
esp_wifi_init(&cfg);
// then WiFi config, start, connect
```

Getting this order wrong produces cryptic failures or hangs during WiFi init.

### SDIO Configuration (P4 ↔ C6)

The SDIO interface between P4 and C6 is 4-bit mode at 40 MHz. The reset line to the C6 is active HIGH (unlike many reset signals). These settings are consistent across P4 boards using the C6 as a co-processor.

```
CONFIG_SDIO_CLK=18
CONFIG_SDIO_CMD=19
CONFIG_SDIO_D0=14 ... D3=17
CONFIG_SLAVE_RESET_GPIO=54   (active HIGH)
CONFIG_SDIO_CLOCK_FREQ_KHZ=40000
CONFIG_SDIO_BUS_WIDTH_4=y
```

### Bluetooth (NimBLE over SDIO)

The P4 has no integrated BT controller — BT runs on the C6 and is accessed via VHCI over SDIO. Key config:

```
CONFIG_BT_CONTROLLER_DISABLED=y
CONFIG_BT_NIMBLE_TRANSPORT_UART=n
CONFIG_ESP_HOSTED_ENABLE_BT_NIMBLE=y
CONFIG_ESP_HOSTED_NIMBLE_HCI_VHCI=y
```

**Known issue:** Logitech multi-device keyboards frequently lose sync when switching BT channels. Mitigation: provide a way to force re-pair (e.g., long-press BOOT button triggers disconnect + re-advertising).

### C6 Co-processor Firmware

Waveshare boards have shipped with C6 firmware v0.0.0 which does not support Bluetooth — check the C6 firmware version on first boot, as this may have been updated in later production runs. If the version is too old, flash the correct firmware manually:

```bash
esptool.py write_flash 0x310000 slave_fw.bin
```

After that, OTA updates to the slave firmware can be handled from the P4 at runtime.

### PSRAM (HEX Mode, 200 MHz)

P4 uses **HEX mode** PSRAM (not OCT). Running at 200 MHz requires experimental feature flag:

```
CONFIG_SPIRAM_MODE_HEX=y
CONFIG_SPIRAM_SPEED_200M=y
CONFIG_IDF_EXPERIMENTAL_FEATURES=y
```

**Critical — APM-560 errata (P4 v1.0/v1.3, not P4X):** An unauthorized AHB access will block ALL PSRAM and flash transactions until reset. Use 64-byte aligned allocations for anything DMA-touched:

```c
heap_caps_aligned_calloc(64, count, size, MALLOC_CAP_SPIRAM);
```

### Cache Sync Before DMA

Any time the CPU writes to a PSRAM buffer that hardware (PPA, DMA) will read, sync the cache first:

```c
esp_cache_msync(buffer, size, ESP_CACHE_MSYNC_FLAG_DIR_C2M);
```

Skipping this causes silent corruption or incorrect display output that's hard to diagnose.

### MIPI-DSI Display Init Sequence

For ST7703-based 720×720 panels, the initialization order matters:

1. Initialize LDO channel 3 at 2.5V for the MIPI PHY
2. Create DSI bus (2 lanes, 480 Mbps)
3. Create DBI IO for command channel
4. Configure DPI panel with video timing (pixel clock, porches)
5. Reset → init → enable panel
6. Allocate double render buffers in PSRAM (64-byte aligned)
7. Set up PPA SRM async client for buffer copy

Typical working DPI timing for 720×720 @ ~60 FPS:
```
pixel_clock: 38 MHz
h_back_porch: 50, h_pulse_width: 20, h_front_porch: 50
v_back_porch: 20, v_pulse_width: 4, v_front_porch: 20
```

### Double-Buffered Rendering (PPA-Accelerated)

The P4's PPA (Pixel Processing Accelerator) handles buffer copies asynchronously. The pattern:

- CPU renders into a back buffer
- PPA copies previous back buffer to framebuffer via DMA
- Semaphore synchronizes `flush_wait()` with PPA completion callback
- Cache sync (`C2M`) required before every PPA transfer

This lets the CPU render the next frame while hardware handles the display copy.

### Console / Logging Configuration

The Waveshare P4 board uses **UART as primary console** with USB JTAG as secondary. The Espressif EV board uses USB_SERIAL_JTAG as primary. Using the EV board sdkconfig on Waveshare hardware produces no application logs — make sure your sdkconfig.defaults sets:

```
CONFIG_ESP_CONSOLE_UART_DEFAULT=y
CONFIG_ESP_CONSOLE_SECONDARY_USB_SERIAL_JTAG=y
```

### Known Errata (P4 v1.0 / v1.3)

These affect the silicon revision on current boards (not P4X):

| Errata | Description | Workaround |
|--------|-------------|-----------|
| APM-560 | Unauthorized AHB access blocks all PSRAM/flash until reset | 64-byte aligned DMA allocations |
| RMT-176 | RMT idle state bug | Set `RMT_IDLE_OUT_EN_CHn=1` |
| I2C-308 | I2C slave multi-read FIFO bug | Avoid I2C slave multi-read; use single reads |

---

## ESP32-S3

### PSRAM Mode

Use **octal mode** for boards with octal PSRAM. Quad mode on an octal board produces "PSRAM chip is not connected" errors.

```
CONFIG_SPIRAM_MODE_OCT=y
```

### Display Notes

**ST7789 via SPI (240×320):**
- 40 MHz SPI clock works reliably
- Color inversion is required on most boards: `esp_lcd_panel_invert_color(panel, true)`

**AMOLED via QSPI (LilyGo T4-S3 / RM67162):**
- The vendor driver dimensions are swapped: `AMOLED_WIDTH=600` is the physical height, `AMOLED_HEIGHT=450` is the physical width. Use `amoled_height()` for column count and `amoled_width()` for row count.
- The full framebuffer must be pushed in a single call — row-by-row transfers do not work with this display.
- 36 MHz QSPI, pixel data is byte-swapped RGB565.

### I2C / OLED Performance

SH1107 OLED (128×128) performance at different I2C speeds:
- 400 kHz → ~17 FPS
- 1 MHz → ~42 FPS
- 1 MHz + dirty page tracking → 60-70 FPS achievable

Internal pull-ups are sufficient for short cable runs at 1 MHz.

**Stack warning:** A 2048-byte framebuffer on the stack will overflow default 3584-byte task stacks. Declare it `static` or heap-allocate it.

---

## ESP32-C6

### Standalone (XIAO ESP32-C6)

No onboard RGB LED — remove any `led_strip` component or it will fail to compile/link.

I2C works reliably at 1 MHz with short cables; internal pull-ups are sufficient for devices like SH1107 at address 0x3C.

---

## Cross-Variant

### FreeRTOS Tick Rate

The default 100 Hz tick rate (10 ms resolution) is too coarse for animation. For smooth display loops:

```
CONFIG_FREERTOS_HZ=1000
```

Then use:
```c
vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(33)); // ~30 FPS
```

### CMakeLists.txt Component Dependencies

ESP-IDF processes `CMakeLists.txt` twice on first configure, and `CONFIG_*` variables are not available on the first pass. Components like `bt` and `esp_hid` must be in `REQUIRES` unconditionally — source file lists can still be conditional, but the component dependency cannot.

```cmake
# Wrong — bt won't be linked if CONFIG_BT_ENABLED isn't set on first pass
if(CONFIG_BT_ENABLED)
    list(APPEND requires bt)
endif()

# Right
idf_component_register(... REQUIRES bt esp_hid ...)
```

### NVS Init Pattern

Always handle the version-mismatch case:

```c
esp_err_t ret = nvs_flash_init();
if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());
    ret = nvs_flash_init();
}
ESP_ERROR_CHECK(ret);
```

### ESP-IDF Version Notes

- **Minimum for P4 + MIPI-DSI support:** v5.3
- **Tested:** v5.5.1
