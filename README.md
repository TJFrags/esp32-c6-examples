# ESP32-C6 Display Programming Guide (LVGL + Display Drivers)

This repository contains ESP32-C6 examples focused on bringing up a display stack with:

- `esp_lcd` (panel IO + panel operations)
- custom LCD panel driver (`JD9853`)
- custom touch driver (`AXS5106`)
- `esp_lvgl_port` + `lvgl` for UI rendering

This document combines:

1. A complete step-by-step guide for programming display applications on ESP32-C6.
2. A function-by-function API reference for the reusable components in this repository.

---

## 1) Repository layout

- `./01_factory`
- `./02_sd_card_test`
- `./03_lvgl_example` ✅ main LVGL demo flow
- `./04_lvgl_image` ✅ LVGL image rendering flow

The LVGL/display-related components are duplicated in `03_lvgl_example/components` and `04_lvgl_image/components`.

---

## 2) Requirements

- ESP-IDF `>=5.1`
- ESP32-C6 target board with:
  - SPI-connected LCD panel (JD9853 in these examples)
  - I2C-connected touch controller (AXS5106 in these examples)
- USB/UART connection for flashing and monitor

Dependencies (from example `main/idf_component.yml`):

- `lvgl/lvgl` (`^8.4.0`)
- `espressif/esp_lcd_touch` (`^1.1.2`)
- `espressif/esp_lvgl_port` (`^2.5.0`)
- `espressif/button` (`^4.1.0`)

---

## 3) Hardware pin mapping used by BSP

Defined in `components/esp_bsp/*.h`.

### SPI LCD

- SCLK: `GPIO_NUM_1`
- MOSI: `GPIO_NUM_2`
- MISO: `GPIO_NUM_3`
- CS: `GPIO_NUM_14`
- DC: `GPIO_NUM_15`
- RST: `GPIO_NUM_22`
- Backlight: `GPIO_NUM_23`

### I2C Touch

- SDA: `GPIO_NUM_18`
- SCL: `GPIO_NUM_19`
- Touch INT: `GPIO_NUM_21`
- Touch RST: `GPIO_NUM_20`

### Other

- SD card CS: `GPIO_NUM_4`
- QMI8658 sensor I2C address: `0x6B`
- AXS5106 touch I2C address: `0x63`

> If your board wiring is different, update the defines in `components/esp_bsp/*.h` first.

---

## 4) Full display bring-up guide (step-by-step)

The flow below matches `03_lvgl_example/main/main.c` and `04_lvgl_image/main/main.c`.

### Step 1: Enter ESP-IDF environment

1. Install ESP-IDF.
2. Open ESP-IDF shell / export environment.
3. Confirm tools: `idf.py --version`.

### Step 2: Choose an example

Use either:

- `./03_lvgl_example`
- `./04_lvgl_image`

### Step 3: Set target

```bash
idf.py set-target esp32c6
```

### Step 4: Initialize runtime storage (NVS)

In `app_main()`, initialize NVS with:

- `nvs_flash_init()`
- on `ESP_ERR_NVS_NO_FREE_PAGES` or `ESP_ERR_NVS_NEW_VERSION_FOUND`, erase + re-init.

### Step 5: Initialize I2C bus (touch + optional sensors)

- Call `bsp_i2c_init()`
- Keep returned `i2c_master_bus_handle_t` for touch/sensor components.

### Step 6: Initialize SPI bus (LCD + SD over SPI)

- Call `bsp_spi_init()`
- This initializes `SPI2_HOST` with configured SCLK/MOSI/MISO pins.

### Step 7: Initialize LCD panel driver

- Call `bsp_display_init(&io_handle, &panel_handle, max_transfer_sz)`
- Internally:
  - creates panel IO over SPI (`esp_lcd_new_panel_io_spi`)
  - creates panel (`esp_lcd_new_panel_jd9853`)
  - resets and inits panel
  - sets invert/mirror and enables display

`max_transfer_sz` in these examples is `EXAMPLE_LCD_H_RES * EXAMPLE_LCD_DRAW_BUFF_HEIGHT`.

### Step 8: Initialize touch controller

- Call:
  - `bsp_touch_init(&touch_handle, i2c_bus_handle, hres, vres, rotation)`
- Internally:
  - adds AXS5106 device on I2C bus
  - configures `esp_lcd_touch_config_t`
  - sets coordinate transform flags based on display rotation
  - creates touch handle

### Step 9: Initialize LVGL and register display/input

In `app_lvgl_init()`:

1. `lvgl_port_init(...)` to start LVGL port/task/timer.
2. `lvgl_port_add_disp(...)` with:
   - LCD IO handle
   - panel handle
   - draw buffer size
   - resolution
   - DMA + double buffering config
3. Set panel gap based on rotation (`esp_lcd_panel_set_gap`).
4. `lvgl_port_add_touch(...)` to bind touch input to display.

### Step 10: Enable backlight and brightness

1. `bsp_display_brightness_init()` to configure LEDC PWM.
2. `bsp_display_set_brightness(0..100)`.

### Step 11: Build UI content safely

Always lock LVGL before object creation:

1. `lvgl_port_lock(0)`
2. create widgets/images (`lv_demo_widgets()` or `lv_img_create(...)`)
3. `lvgl_port_unlock()`

### Step 12: Build, flash, monitor

```bash
idf.py build
idf.py -p <PORT> flash monitor
```

---

## 5) Rotation/resolution alignment rules

In these examples:

- Rotation is controlled by `EXAMPLE_DISPLAY_ROTATION` (`0/90/180/270`)
- `hres/vres` are swapped for 90/270
- LVGL rotation flags and touch rotation flags must match
- panel gap (`esp_lcd_panel_set_gap`) must match panel orientation

If these are inconsistent, touch points and rendered UI will appear shifted or mirrored.

---

## 6) Public API reference (all component functions)

This section documents public, callable functions declared in component headers.

## 6.1 `esp_bsp` component APIs

### `i2c_master_bus_handle_t bsp_i2c_init(void)`

- **Defined in:** `components/esp_bsp/bsp_i2c.h`
- **Purpose:** Initialize ESP-IDF I2C master bus for touch/sensors.
- **Returns:** I2C bus handle.
- **Use when:** before `bsp_touch_init()` or `bsp_qmi8658_init()`.

---

### `void bsp_spi_init(void)`

- **Defined in:** `components/esp_bsp/bsp_spi.h`
- **Purpose:** Initialize SPI bus used by LCD and SPI SD card.
- **Returns:** none.
- **Use when:** before `bsp_display_init()` or `bsp_sdcard_init()`.

---

### `void bsp_display_init(esp_lcd_panel_io_handle_t *io_handle, esp_lcd_panel_handle_t *panel_handle, size_t max_transfer_sz)`

- **Defined in:** `components/esp_bsp/bsp_display.h`
- **Purpose:** Initialize LCD panel IO + JD9853 panel and turn display on.
- **Parameters:**
  - `io_handle` (out): panel IO handle.
  - `panel_handle` (out): panel handle.
  - `max_transfer_sz`: maximum frame transfer chunk size.
- **Use when:** once after SPI init, before LVGL display registration.

---

### `void bsp_display_brightness_init(void)`

- **Defined in:** `components/esp_bsp/bsp_display.h`
- **Purpose:** Initialize LEDC PWM channel for LCD backlight control.
- **Returns:** none.
- **Use when:** before first call to `bsp_display_set_brightness()`.

---

### `void bsp_display_set_brightness(uint8_t brightness)`

- **Defined in:** `components/esp_bsp/bsp_display.h`
- **Purpose:** Set backlight intensity in percent.
- **Parameters:** `brightness` in range `0..100` (values >100 are clamped).
- **Returns:** none.
- **Use when:** after `bsp_display_brightness_init()`.

---

### `uint8_t bsp_display_get_brightness(void)`

- **Defined in:** `components/esp_bsp/bsp_display.h`
- **Purpose:** Read cached brightness percentage.
- **Returns:** current brightness value (`0..100`).

---

### `void bsp_touch_init(esp_lcd_touch_handle_t *touch_handle, i2c_master_bus_handle_t bus_handle, uint16_t xmax, uint16_t ymax, uint16_t rotation)`

- **Defined in:** `components/esp_bsp/bsp_touch.h`
- **Purpose:** Initialize AXS5106 touch device and map coordinates.
- **Parameters:**
  - `touch_handle` (out): touch handle.
  - `bus_handle`: initialized I2C bus.
  - `xmax`, `ymax`: display dimensions.
  - `rotation`: 0/90/180/270.
- **Use when:** after I2C init and before `lvgl_port_add_touch()`.

---

### `void bsp_wifi_init(const char *ssid, const char *pass)`

- **Defined in:** `components/esp_bsp/bsp_wifi.h`
- **Purpose:** Initialize netif/event loop/Wi-Fi driver and register event handlers.
- **Parameters:** optional `ssid`, `pass` for immediate STA connect.
- **Returns:** none.
- **Use when:** first Wi-Fi API call in app.

---

### `esp_err_t bsp_wifi_sta_connect(const char *ssid, const char *password)`

- **Defined in:** `components/esp_bsp/bsp_wifi.h`
- **Purpose:** Configure station credentials and start connection.
- **Returns:** `ESP_OK` on requested connection start.
- **Use when:** after `bsp_wifi_init()`.

---

### `bool bsp_wifi_scan(wifi_ap_record_t *ap_info, uint16_t *scan_number, uint16_t scan_max_num)`

- **Defined in:** `components/esp_bsp/bsp_wifi.h`
- **Purpose:** Start Wi-Fi scan and fill AP records.
- **Parameters:**
  - `ap_info`: caller buffer for AP records.
  - `scan_number` (out): total (and capped) AP count.
  - `scan_max_num`: capacity of `ap_info`.
- **Returns:** `true` on success, `false` on failure/timeout.
- **Use when:** after Wi-Fi init.

---

### `void bsp_wifi_get_ip(char *ip)`

- **Defined in:** `components/esp_bsp/bsp_wifi.h`
- **Purpose:** Get station IP as string.
- **Parameters:** `ip` output buffer.
- **Returns:** none.
- **Use when:** after successful Wi-Fi connection.

---

### `void bsp_sdcard_init(void)`

- **Defined in:** `components/esp_bsp/bsp_sdcard.h`
- **Purpose:** Mount SD card filesystem at `/sdcard` over SDSPI.
- **Returns:** none.
- **Use when:** after SPI init.

---

### `uint64_t bsp_sdcard_get_size(void)`

- **Defined in:** `components/esp_bsp/bsp_sdcard.h`
- **Purpose:** Return mounted card capacity in bytes.
- **Returns:** card size, or `0` if card is not mounted.

---

### `void bsp_battery_init(void)`

- **Defined in:** `components/esp_bsp/bsp_battery.h`
- **Purpose:** Initialize ADC one-shot and calibration for battery channel.
- **Returns:** none.
- **Use when:** before reading battery values.

---

### `void bsp_battery_get_voltage(float *voltage, uint16_t *adc_value)`

- **Defined in:** `components/esp_bsp/bsp_battery.h`
- **Purpose:** Read ADC and compute battery voltage estimate.
- **Parameters:**
  - `voltage` (out): voltage estimate.
  - `adc_value` (optional out): raw ADC code.
- **Returns:** none.
- **Use when:** after `bsp_battery_init()`.

---

### `void bsp_qmi8658_init(i2c_master_bus_handle_t bus_handle)`

- **Defined in:** `components/esp_bsp/bsp_qmi8658.h`
- **Purpose:** Initialize QMI8658 IMU over I2C and configure accel/gyro modes.
- **Returns:** none.
- **Use when:** after I2C init.

---

### `bool bsp_qmi8658_read_data(qmi8658_data_t *data)`

- **Defined in:** `components/esp_bsp/bsp_qmi8658.h`
- **Purpose:** Read accel/gyro raw data and compute tilt angles.
- **Parameters:** `data` output struct.
- **Returns:** `true` when fresh data available, otherwise `false`.
- **Use when:** after `bsp_qmi8658_init()`.

---

### `void bsp_qmi8658_test(void)`

- **Defined in:** `components/esp_bsp/bsp_qmi8658.h`
- **Purpose:** Start internal test task that continuously reads sensor data.
- **Returns:** none.
- **Use when:** quick validation/debug.

---

## 6.2 LCD panel driver API (`esp_lcd_jd9853`)

### `esp_err_t esp_lcd_new_panel_jd9853(const esp_lcd_panel_io_handle_t io, const esp_lcd_panel_dev_config_t *panel_dev_config, esp_lcd_panel_handle_t *ret_panel)`

- **Defined in:** `components/esp_lcd_jd9853/include/esp_lcd_jd9853.h`
- **Purpose:** Create JD9853 panel object for use with `esp_lcd` panel ops.
- **Parameters:**
  - `io`: initialized panel IO handle.
  - `panel_dev_config`: panel settings (`reset_gpio_num`, color order, bpp, optional vendor config).
  - `ret_panel` (out): created panel handle.
- **Returns:** `ESP_OK` / error code.
- **Use when:** after creating panel IO (`esp_lcd_new_panel_io_spi`).

### Related helper macros

- `JD9853_PANEL_BUS_SPI_CONFIG(...)` for SPI bus config literals.
- `JD9853_PANEL_IO_SPI_CONFIG(...)` for panel IO config literals.

---

## 6.3 Touch driver API (`esp_lcd_touch_axs5106`)

### `esp_err_t esp_lcd_touch_new_i2c_axs5106(i2c_master_dev_handle_t dev_handle, const esp_lcd_touch_config_t *config, esp_lcd_touch_handle_t *out_touch)`

- **Defined in:** `components/esp_lcd_touch_axs5106/include/esp_lcd_touch_axs5106.h`
- **Purpose:** Create AXS5106 touch driver instance over I2C.
- **Parameters:**
  - `dev_handle`: I2C device handle for AXS5106.
  - `config`: touch config (max coords, reset/int pins, axis flags).
  - `out_touch` (out): touch handle.
- **Returns:** `ESP_OK` / error code.
- **Use when:** after I2C device registration (`i2c_master_bus_add_device`).

### Related constants/macros

- `ESP_LCD_TOUCH_IO_I2C_AXS5106_ADDRESS` (`0x63`)
- `ESP_LCD_TOUCH_IO_I2C_AXS5106_CONFIG()` (helper config literal)

---

## 6.4 LVGL UI helper APIs (`lvgl_ui`)

### `void lvgl_ui_init(void)`

- **Defined in:** `components/lvgl_ui/lvgl_ui.h`
- **Purpose:** Build repository-provided tileview UI on current LVGL screen.
- **Use when:** after LVGL display/input setup and inside LVGL lock.

### Tile initialization helpers

All are defined under `components/lvgl_ui/tileview/*.h` and each builds content into a tile/container parent:

- `void rgb_tile_init(lv_obj_t *parent)`
- `void system_tile_init(lv_obj_t *parent)`
- `void qmi8658_tile_init(lv_obj_t *parent)`
- `void camera_tile_init(lv_obj_t *parent)`
- `void wifi_tile_init(lv_obj_t *parent)`
- `void image_tile_init(lv_obj_t *parent)`

Use them from your custom `lv_tileview` composition flow.

---

## 7) External APIs used in the main bring-up (important)

These are not defined in local components but are central to usage:

- `lvgl_port_init`
- `lvgl_port_add_disp`
- `lvgl_port_add_touch`
- `lvgl_port_lock` / `lvgl_port_unlock`
- `esp_lcd_panel_set_gap`
- `esp_lcd_panel_reset` / `esp_lcd_panel_init` / `esp_lcd_panel_disp_on_off`

Check ESP-IDF and component package docs for full argument details and version-specific behavior.

---

## 8) Typical initialization order (recommended)

1. `nvs_flash_init`
2. `bsp_i2c_init`
3. `bsp_spi_init`
4. `bsp_display_init`
5. `bsp_touch_init`
6. `lvgl_port_init`
7. `lvgl_port_add_disp`
8. `lvgl_port_add_touch`
9. `bsp_display_brightness_init`
10. `bsp_display_set_brightness`
11. `lvgl_port_lock` + create LVGL objects + `lvgl_port_unlock`

---

## 9) Build and flash

Example:

```bash
cd ./03_lvgl_example
idf.py set-target esp32c6
idf.py build
idf.py -p <PORT> flash monitor
```

For image rendering example:

```bash
cd ./04_lvgl_image
idf.py set-target esp32c6
idf.py build
idf.py -p <PORT> flash monitor
```

---

## 10) Troubleshooting

### Screen stays black

- Check backlight init and brightness (`bsp_display_brightness_init`, `bsp_display_set_brightness`).
- Verify LCD power and SPI pin mapping.

### Garbled/offset image

- Verify `hres/vres`, rotation flags, and `esp_lcd_panel_set_gap` are aligned.

### Touch mismatch (mirrored/swapped)

- Verify `bsp_touch_init(... rotation ...)` matches display rotation.

### SD mount fails

- Ensure SPI bus already initialized.
- Check pull-ups and card wiring.

### Wi-Fi scan/connect unstable

- Ensure `bsp_wifi_init` called once before scan/connect operations.

---

## 11) Notes

- `03_lvgl_example` and `04_lvgl_image` currently include similar driver/component copies.
- If you update one component copy, apply the same change to the other to keep examples consistent.
