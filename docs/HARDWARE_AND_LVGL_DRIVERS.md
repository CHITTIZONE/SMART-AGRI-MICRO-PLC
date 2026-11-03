# 📖 Smart Agri Micro-PLC Hardware Pinout & LVGL Driver Architecture Guide

**System:** Low-Cost Embedded PLC & SCADA HMI for Clean Water Treatment  
**Controller:** Espressif ESP32-S3-WROOM-1U (Dual-Core Xtensa LX7 @ 240 MHz, 16MB Flash, 8MB PSRAM)  
**Display:** 1.8" ST7735 SPI TFT LCD (160×128 pixels, 16-bit RGB565)  
**Framework:** Arduino / PlatformIO, LVGL v8.3.11 with EEZ-Flow Engine  

---

## 1. 📌 Complete GPIO Hardware Pinout Mapping

| Component | Pin Name | ESP32-S3 GPIO | Electrical Type | Circuit & Description |
|:---|:---|:---:|:---|:---|
| **ST7735 TFT** | `SDA / MOSI` | **GPIO 11** | HSPI Master Out | Hardware SPI Data Line (16 MHz rated) |
| **ST7735 TFT** | `SCL / SCK` | **GPIO 12** | HSPI Clock | Hardware SPI Clock Line |
| **ST7735 TFT** | `CS` | **GPIO 10** | Digital Output | Chip Select (Active LOW, pulled HIGH) |
| **ST7735 TFT** | `DC / A0` | **GPIO 9** | Digital Output | Data / Command Selector (LOW = Command, HIGH = Data) |
| **ST7735 TFT** | `RESET` | **GPIO 8** | Digital Output | Hardware Reset (Active LOW, pulse on startup) |
| **ST7735 TFT** | `BL (Backlight)`| **GPIO 21** | Digital Output | Backlight Control via BC547 NPN transistor (Active HIGH) |
| **START Button** | `BTN_START` | **GPIO 4** | Digital Input | Tactile Push Button with **10kΩ external pull-down**.<br>• **Short Press**: Starts pump (Auto & Manual).<br>• **3-Second Hold**: Clears E-STOP latch & turns pump ON. |
| **STOP Button** | `BTN_STOP` | **GPIO 6** | Digital Input | Tactile Push Button with **10kΩ external pull-down**.<br>• **Short Press**: Stops pump.<br>• **2-Second Hold**: Toggles between `AUTO` and `MANUAL` modes. |
| **E-STOP Button** | `BTN_ESTOP` | **GPIO 5** | Digital Input | Emergency Stop Button with **10kΩ external pull-down**.<br>• **Press**: Instant hardware-level motor cutoff, freezes logic, flashes Red RGB. |
| **Motor Output** | `OUT_MOTOR` | **GPIO 7** | Digital Output | Relay Driver Output (Active HIGH). Connected to 5V relay module driving the pump contactor. |
| **Onboard RGB** | `RGB_LED` | **GPIO 48** | WS2812B Data | Built-in DevKit NeoPixel Smart LED. Sole optical status indicator. |

---

## 2. 💡 Built-in WS2812B RGB Status Indicator (GPIO 48)

The onboard NeoPixel provides instant visual plant status without requiring external indicator lamps:

| System State | Colour | RGB Value | Pattern | Description |
|:---|:---:|:---:|:---:|:---|
| **System Idle / Standby** | ⚪ White | `(80, 80, 80)` | Steady Solid | Ready for operation, all interlocks normal. |
| **Pump Running** | 🟢 Green | `(0, 180, 0)` | Steady Solid | Motor relay energized, pumping active. |
| **Pump Stopped** | 🔴 Red | `(180, 0, 0)` | Steady Solid | Motor stopped by operator or tank empty. |
| **Emergency Stop** | 🔴 Red | `(200, 0, 0)` | **Blinking 500ms** | E-STOP latched. Motor locked OFF until reset. |
| **Voltage Trip** | 🟡 Yellow | `(180, 130, 0)` | Steady Solid | Voltage out of bounds (`HIGH VOLT` or `LOW VOLT`). |
| **Current Trip** | 🟠 Orange | `(180, 60, 0)` | Steady Solid | Current out of bounds (`DRY LOAD` or `OVER LOAD`). |
| **Booting / Initializing** | 🔵 Blue | `(0, 0, 80)` | Steady Fade | Firmware startup, splash screen display. |

---

## 3. 🖥️ LVGL Display Architecture & ST7735 Driver Pipeline

### 3.1 Display Driver Configuration
- **Resolution**: 160 × 128 (Landscape rotation 1).
- **Buffer Architecture**: Partial rendering buffer of `160 * 20` pixels (`buf1`) in internal SRAM to ensure zero memory exhaustion.
- **Flush Callback (`my_disp_flush`)**:
  ```cpp
  void my_disp_flush(lv_disp_drv_t *disp, const lv_area_t *area, lv_color_t *color_p) {
      uint16_t w = area->x2 - area->x1 + 1;
      uint16_t h = area->y2 - area->y1 + 1;
      tft.startWrite();
      tft.setAddrWindow(area->x1, area->y1, w, h);
      for (uint32_t i = 0; i < (uint32_t)(w * h); i++) tft.writeColor(color_p[i].full, 1);
      tft.endWrite();
      lv_disp_flush_ready(disp);
  }
  ```
- **Driver Initialization**:
  - SPI frequency configured to **16 MHz** on `HSPI` pins.
  - Native custom WTP boot splash screen drawn directly before LVGL handoff, bypassing legacy logo headers.

---

## 4. 📊 EEZ-Flow Global Variables & LVGL Widget Bindings

The display layout is defined via EEZ-GUI (`screens.c` / `vars.h`). The firmware communicates directly with LVGL widgets via **Flow Global Variables**:

| Variable Constant | Index | Bound Widget | Data Type | Screen Format / Value | Description |
|:---|:---:|:---|:---:|:---|:---|
| `FLOW_GLOBAL_VARIABLE_MOTOR_STATUS` | `0` | `timer_state_1` | `String` | `"PUMP RUNNING"`, `"MOTOR OFF - HIGH VOLT"`, etc. | Top-center status banner label. Displays exact operating and fault states. |
| `FLOW_GLOBAL_VARIABLE_TIME_HOUR` | `1` | `timer_hour` | `String` | `"01"` to `"12"` | Centered 12-hour digit display (Montserrat 20). |
| `FLOW_GLOBAL_VARIABLE_TIME_MIN` | `2` | `timer_min` | `String` | `"00"` to `"59"` | 2-digit minute display. |
| `FLOW_GLOBAL_VARIABLE_TIMER_APM` | `3` | `timer_apm` | `String` | `"AM"` or `"PM"` | 12-hour AM/PM meridian indicator. |
| `FLOW_GLOBAL_VARIABLE_TIMER_STATE` | `4` | `timer_state` | `String` | `" AUTO MODE"` / `" MANUAL"` | Operational mode sub-label. |
| `FLOW_GLOBAL_VARIABLE_VOLTAGE_AC` | `5` | `voltage_ac` | `String` | `"230 V"` | Live AC Voltage reading (3-digit integer format). |
| `FLOW_GLOBAL_VARIABLE_CURRENT_AC` | `6` | `current_ac` | `String` | `"5.5 A"` | Live AC Current reading (1 decimal place format). |
| `FLOW_GLOBAL_VARIABLE_VOLTAGE_LOW_AC` | `10` | `voltage_low_ac_` | `String` | `"180 V"` | Preset Undervoltage protection cutoff threshold. |
| `FLOW_GLOBAL_VARIABLE_VOLTAGE_HIGH_AC`| `11` | `voltage_high_ac` | `String` | `"260 V"` | Preset Overvoltage protection cutoff threshold. |
| `FLOW_GLOBAL_VARIABLE_CURRENT_AC_LOW` | `12` | `current_ac_low` | `String` | `"0.5 A"` | Preset Dry Run / Low Load cutoff threshold. |
| `FLOW_GLOBAL_VARIABLE_CURRENT_AC_HIGH`| `13` | `current_ac_high`| `String` | `"12.0 A"` | Preset Overload protection cutoff threshold. |

---

## 5. 🔗 C-Exported Widget Getters (`display_mgr.cpp`)

Certain EEZ widgets invoke compiled C-getter callbacks directly from `screens.c`. These are exported with `extern "C"` to guarantee binary compatibility:

```cpp
extern "C" {
    // Mode Info widget: Replaces legacy LoRa status with clean "HMI"
    const char* getModeInfoDisplayText()    { return "    HMI"; }

    // Preset threshold dynamic bindings
    const char* getVoltageLowThreshText()   { return str_voltage_low.c_str(); }
    const char* getVoltageHighThreshText()  { return str_voltage_high.c_str(); }
    const char* getCurrentLowThreshText()   { return str_current_low.c_str(); }
    const char* getCurrentHighThreshText()  { return str_current_high.c_str(); }

    // Timer seconds & status indicators
    const char* getTimerSecondText()        { return str_timer_second.c_str(); }
    const char* getTimerOnOffText()         { return str_timer_on_off.c_str(); }
}
```

---

## 6. 🌐 SCADA WebSockets & Virtual Testing Protocol

The ESP32-S3 broadcasts plant telemetry every **250ms** over WebSocket port `81` to connected SCADA dashboards:

```json
{
  "state": "RUNNING",
  "mode": "AUTO",
  "motor": true,
  "estop": false,
  "v": 230.0,
  "i": 5.5,
  "sumpH": true,
  "sumpL": true,
  "vMin": 180.0,
  "vMax": 260.0,
  "iMin": 0.5,
  "iMax": 12.0,
  "banner": "PUMP RUNNING",
  "time": "01:06:40"
}
```

Commands received from the SCADA dashboard:
- `start` / `stop`: Manual motor override.
- `estop` / `estop_reset`: Emergency shutdown & latch clear.
- `mode`: Toggle between `"auto"` and `"manual"`.
- `limit_sw`: Update virtual float switches (`high: 0/1`, `low: 0/1`).
- `live_vi`: Push simulated test voltage (`v`) and current (`i`).
- `presets`: Save new protection thresholds to ESP32 NVS memory.
- `set_time`: Synchronize POSIX epoch timestamp (`IST-5:30` timezone).

