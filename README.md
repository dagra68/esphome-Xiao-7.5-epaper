# ESPHome Dashboard – Xiao ESP32-C3 + 7.5" ePaper Display

An ESPHome configuration for a **Seeed Studio Xiao ESP32-C3** driving a **Waveshare 7.5" e-ink display** (800×480, model `7.50inv2`). It pulls sensor data from Home Assistant and renders a grid-based dashboard with icons, values, and a status bar — then goes to deep sleep to save power.

## Features

- 6 sensor tiles in a 3×2 grid (Pool Temp, Redox, pH, Outdoor Temp, UV Index, PV Production)
- Status bar with pump states and daily energy consumption
- Material Design Icons via the MDI webfont
- Deep sleep (5 min sleep / 1 min awake) for battery-friendly operation
- Single display update per wake cycle (waits for WiFi + sensor data)

## Hardware Wiring (Xiao ESP32-C3 → Waveshare 7.5" ePaper)

| Function | GPIO |
|----------|------|
| SPI CLK  | GPIO8 |
| SPI MOSI | GPIO10 |
| CS       | GPIO3 |
| DC       | GPIO5 |
| BUSY     | GPIO4 (inverted) |
| RESET    | GPIO2 |

## Prerequisites

1. Place the MDI font file at `fonts/materialdesignicons-webfont.ttf` relative to your ESPHome config directory. You can download it from the [MDI GitHub releases](https://github.com/Templarian/MaterialDesign-Webfont/releases).
2. Create a `secrets.yaml` with your credentials:
   ```yaml
   wifi_ssid: "YourSSID"
   wifi_password: "YourPassword"
   api_key: "your-api-encryption-key"
   ota_password: "your-ota-password"
   ```

## How to Change Entities

Each tile on the display is driven by a Home Assistant sensor. To swap an entity, you need to change it in **two places**:

### 1. The sensor definition (top of the file)

Find the `sensor:` section and change the `entity_id` to your own Home Assistant entity:

```yaml
sensor:
  - platform: homeassistant
    id: pool_temp                              # internal ID used in the display code
    entity_id: sensor.pool_temperature         # ← change this to your entity
    name: "Pool Temperatur"
```

For binary sensors (on/off switches), look in the `binary_sensor:` section:

```yaml
binary_sensor:
  - platform: homeassistant
    id: poolpumpe_an
    entity_id: switch.pool_umwalzpumpe         # ← change this to your entity
    name: "Poolpumpe"
```

### 2. The display lambda (rendering code)

In the `display:` → `lambda:` section, each tile reads the sensor by its `id`. If you changed the `id` above, update it here too:

```cpp
fval(id(pool_temp).state, "%.1f", buf, sizeof(buf));
//        ^^^^^^^^^ must match the sensor id
```

Also update the label text and unit to match your new sensor:

```cpp
it.print(col[0]+12, row[0]+8, id(font_label), "Pool Temp");   // ← label
it.print(col[0]+188, row[0]+112, id(font_tiny), "°C");        // ← unit
```

### Adding a new sensor

1. Add a new `sensor:` entry with a unique `id` and the correct `entity_id`
2. Add the icon codepoint to the `glyphs:` list (see below)
3. Add a new tile block in the display lambda using the grid position system (`col[x]`, `row[y]`)

## How to Change Icons

This project uses **Material Design Icons (MDI)** rendered from a TTF font. Each icon is referenced by its Unicode codepoint.

### Current icon mapping

| Icon | Codepoint | MDI Name | Used for |
|------|-----------|----------|----------|
| 🌡️ | `\U000F1A5F` | `mdi:pool-thermometer` | Pool Temp |
| 🌡️ | `\U000F050F` | `mdi:thermometer` | Outdoor Temp |
| ⚗️ | `\U000F0668` | `mdi:flash` | Redox |
| 🧪 | `\U000F17C5` | `mdi:ph` | pH Value |
| 💧 | `\U000F1402` | `mdi:pump` | Pool Pump |
| ♨️ | `\U000F1A44` | `mdi:heat-pump` | Heat Pump |
| ⚡ | `\U000F1A57` | `mdi:pool` | Pool Energy |
| ☀️ | `\U000F0599` | `mdi:weather-sunny` | UV Index |
| 🔆 | `\U000F1A73` | `mdi:solar-power` | PV Production |
| 📶 | `\U000F051F` | `mdi:wifi-off` | WiFi Off |

### Step-by-step: Changing an icon

**1. Find your new icon codepoint**

Browse the full MDI icon library at:
👉 **https://pictogrammers.com/library/mdi/**

Search for an icon, click on it, and copy the **codepoint** (e.g. `F0599`).

**2. Register the glyph in the font section**

In the `font:` section under `glyphs:`, add or replace the codepoint. Prefix it with `\U000` and pad to 8 hex digits:

```yaml
glyphs:
  - "\U000F0599"   # weather-sunny
  - "\U000F0E0B"   # ← your new icon (e.g. mdi:home-thermometer = F0E0B)
```

**3. Use the icon in the display lambda**

Reference the same codepoint in the `it.printf()` call for the tile:

```cpp
it.printf(col[0]+50, row[0]+72, id(font_icon_big),
          TextAlign::CENTER, "\U000F0E0B");   // ← same codepoint
```

### Important notes about icons

- You **must** register every icon codepoint in the `glyphs:` list — ESPHome only includes glyphs that are explicitly listed to save memory.
- The codepoint format is `\U000XXXXX` where `XXXXX` is the hex code from the MDI website (prepend zeros to make 8 digits total after `\U`).
- If an icon doesn't show up, double-check that the codepoint matches between `glyphs:` and the lambda, and that your font file version includes that icon.

## Display Layout

![ePaper Dashboard](display.png)

## License

Feel free to use and adapt this configuration for your own ePaper dashboard.
