# ESP32-P4 2-Lane MIPI LCD Bring-Up: A Working Checklist for a 7-inch 1024×600 Panel

The ESP32-P4 is the first ESP32 with a MIPI DSI peripheral, which means a 7-inch 1024×600 panel can hang directly off the MCU — no applications processor, no bridge chip. This is the checklist that lit the panel on the first careful pass: the hardware limits, the bring-up order, and the code that worked. Timing values are panel-specific and come from the panel datasheet; nothing here invents them.

## Know the two limits before you start

The P4's DSI peripheral runs Video mode only — there is no Command mode — so the panel must accept a continuous pixel stream. And it tops out at two lanes. Most MIPI panels can be strapped to two lanes, but confirm it in the datasheet and ask the vendor for the two-lane initialisation sequence before ordering.

The panel in this build is a 7-inch 1024×600 MIPI module with capacitive touch, driven at RGB888 over two lanes at 900 Mbps with a 52 MHz pixel clock. At 1024×600 and 60 Hz, RGB888 needs about 854 Mbps, comfortably inside the 480–1500 Mbps per-lane range Espressif recommends. The panel used here comes from Hongcai Technology's [LCD Display](https://www.lcdhc.com/) range — 1.1 to 14.5 inches, SPI, RGB, MIPI and LVDS interfaces — and the bring-up below was run on its 7-inch MIPI module.

Wiring is minimal: one clock pair and two data pairs, plus reset, backlight, and the touch lines.

## The checklist

1. **Power and reset.** Hold the panel in reset while supplies settle, then release. A panel released too early may never answer, and nothing downstream will tell you why.
2. **Initialisation commands.** The driver ICs — an EK79007AD3 timing controller with an EK73217BCGA — need their init sequence over the DSI command channel before video starts. The sequence is panel-specific and comes from the panel vendor; copying another panel's sequence is the fastest route to a panel that powers up and shows nothing.
3. **DSI video configuration.** Two lanes, the panel's resolution and timing, all from the datasheet. Every porch and pulse width matters.
4. **First frame, then backlight.** Bring the backlight up only after video is flowing. A lit panel with no video looks exactly like a dead panel.
5. **Touch last.** Initialise the capacitive touch controller over I2C once the display runs, then check touch coordinates map onto display coordinates. If they don't, swap or invert the axes in the touch driver, not the display timing.

## The key code

The listing below is the skeleton of the working bring-up, using Espressif's `esp_lcd` abstractions. It is the configuration that lit the panel: 1024×600, RGB888, two lanes at 900 Mbps, 52 MHz pixel clock. Timing values are panel-specific and are left as placeholders — fill them from your panel's datasheet and do not guess them.

```c
#include "esp_lcd_mipi_dsi.h"
#include "esp_lcd_panel_ops.h"
#include "esp_lcd_ek79007.h"

/* 7.0" IPS 1024x600, 2-lane MIPI DSI
 * Driver ICs: EK79007AD3 (TCON) + EK73217BCGA
 * RGB888, 52 MHz pixel clock (~60 Hz), 2 lanes @ 900 Mbps */

#define LCD_H_RES            1024
#define LCD_V_RES            600
#define LCD_BITS_PER_PIXEL   24   /* RGB888 */
#define LCD_DSI_LANE_NUM     2
#define LCD_DSI_LANE_BITRATE 900  /* Mbps */
#define LCD_PIXEL_CLOCK_MHZ  52

esp_lcd_dsi_bus_handle_t mipi_dsi_bus = NULL;
esp_lcd_dsi_bus_config_t bus_config = {
    .bus_id = 0,
    .num_data_lanes = LCD_DSI_LANE_NUM,
    .phy_clk_src = MIPI_DSI_PHY_CLK_SRC_DEFAULT,
    .lane_bit_rate_mbps = LCD_DSI_LANE_BITRATE,
};
ESP_ERROR_CHECK(esp_lcd_new_dsi_bus(&bus_config, &mipi_dsi_bus));

esp_lcd_panel_handle_t panel_handle = NULL;
esp_lcd_dsi_device_config_t dev_config = {
    .virtual_channel = 0,
    .lanes = LCD_DSI_LANE_NUM,
    .video_timing = {
        .h_size = LCD_H_RES,
        .v_size = LCD_V_RES,
        .hsync_pulse_width  = /* per panel datasheet */,
        .hsync_back_porch   = /* per panel datasheet */,
        .hsync_front_porch  = /* per panel datasheet */,
        .vsync_pulse_width  = /* per panel datasheet */,
        .vsync_back_porch   = /* per panel datasheet */,
        .vsync_front_porch  = /* per panel datasheet */,
    },
};
/* Vendor init commands (panel-specific) are passed here */
ESP_ERROR_CHECK(esp_lcd_new_panel_ek79007(mipi_dsi_bus, &dev_config, &panel_handle));

ESP_ERROR_CHECK(esp_lcd_panel_reset(panel_handle));
ESP_ERROR_CHECK(esp_lcd_panel_init(panel_handle));
ESP_ERROR_CHECK(esp_lcd_panel_disp_on_off(panel_handle, true));
/* Backlight on (PWM) — only after the first frame is flowing */
```

## When it fails

**Black screen.** Check in this order: reset timing, init sequence, then whether frames are actually being sent — a scope on the DSI clock lane settles the last one in minutes.

**Garbage or tearing.** Almost always a timing mismatch. Re-check every porch and pulse width against the datasheet, and confirm the lane count agrees on both sides.

**Wrong colours.** The pixel format is the usual suspect: RGB565 against RGB888, or the RGB order flipped. The format is a host-side choice, set in code, not in the panel.

Two wiring traps are worth naming because they are invisible in software. The first is D0 and D1 swapped on the flex cable — the link trains, nothing displays, and no register tells you. The second is using a four-lane init sequence on a two-lane setup; the panel accepts the commands and then waits for lanes that never come.

## Bottom line

Size the panel to the lanes you have, get the init sequence from the vendor, and bring the backlight up last. The same sequence transfers to any other MIPI panel on the ESP32-P4 — only the vendor init commands and timing values change.

*About the author: Hongcai Technology is a TFT LCD manufacturer based in Shenzhen, China, designing and building custom display modules — 1.1 to 14.5 inches, SPI, RGB, MIPI and LVDS interfaces with capacitive touch — for industrial, medical and consumer products.*
