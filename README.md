# Home Assistant Custom Component for PAX BLE Fans

## Installation

Download using HACS (manually add repo) or manually put it in the `custom_components` folder.

## Supported devices

| Device | Notes |
|---|---|
| PAX Calima | Fully supported |
| PAX Levante 50 | Fully supported |
| Vent-Axia Svara | Same hardware as Calima |
| Vent-Axia Svensa | Same hardware as PureAire Sense |

If you have a similar fan not listed here, give it a try and report back.

## Adding a device

The integration supports automatic Bluetooth discovery for Calima and Levante. If discovery doesn't work, add it manually through the integration configuration.

If you have trouble connecting, power-cycle the fan. The Bluetooth interface can hang if the fan has been connected to another app or device recently.

For Svensa-specific setup instructions, see [svensa.md](svensa.md).

## PIN code

A PIN is required to control the fan. Without it, sensor values can still be read.

| Device | How to get the PIN |
|---|---|
| Calima / Svara | Printed on the motor (remove from base) |
| Levante 50 | Power-cycle using the side switch just before adding — PIN is discovered automatically. If this fails, follow the Svensa instructions. |
| Svensa | Not printed on the device. See [svensa.md](svensa.md). |

## Entities

### Sensors

These update on every poll (default every 60 seconds).

| Entity | Unit | Description |
|---|---|---|
| Humidity | % | Relative humidity measured by the onboard sensor. Shows 0 when humidity is very low. |
| Temperature | °C | Temperature at the fan's PCB. Reads 2–5 °C higher than actual room temperature due to self-heating from the motor. |
| Light | lx | Light level at the fan's light sensor inside the duct. Accuracy depends heavily on installation. |
| RPM | rpm | Current fan speed in revolutions per minute. |
| Flow | m³/h | Estimated airflow derived from RPM using a linear approximation. Accuracy depends on installation (duct length, bends, etc.). |
| State | — | What is currently driving the fan: `No trigger`, `Trickle ventilation`, `Humidity ventilation`, `Light ventilation`, `Boost`, or `Switch`. |

### Controls

| Entity | Description |
|---|---|
| BoostMode | Turns boost on or off. When turned on, the fan runs at **BoostMode Speed** for **BoostMode Time** seconds. |

### Configuration

These are read from the device on startup and refreshed every 24 hours. Changes made in the PAX app will appear in HA after the next refresh (or on restart).

#### Fan speeds

| Entity | Range | Description |
|---|---|---|
| Fanspeed Humidity | 800–2400 rpm | Fan speed when the humidity sensor triggers ventilation. |
| Fanspeed Light | 800–2400 rpm | Fan speed when the light sensor triggers ventilation. |
| Fanspeed Trickle | 800–2400 rpm | Continuous background (trickle) ventilation speed. |

> **Note:** Setting any speed below 800 rpm may stall the fan. Whether this causes damage is unknown — use with care.

#### Boost mode

> These two values are **local to Home Assistant** and are not read from or written to the device until boost is activated. They will reset to defaults (2400 rpm / 600 s) if you re-add the device.

| Entity | Range | Description |
|---|---|---|
| BoostMode Speed | 1000–2400 rpm | Fan speed during boost. |
| BoostMode Time | 60–900 s | How long boost runs before the fan returns to normal. |

#### Sensor triggers

| Entity | Options | Description |
|---|---|---|
| Sensitivity Humidity | Off, Low, Medium, High sensitivity | How readily the humidity sensor activates ventilation. Off disables humidity-triggered ventilation entirely. |
| Sensitivity Light | Off, Low, Medium, High sensitivity | How readily the light sensor activates ventilation. Off disables light-triggered ventilation. |

#### Light sensor timing

| Entity | Options | Description |
|---|---|---|
| LightSensorSettings DelayedStart | No delay, 5 min, 10 min | How long after the light turns on before the fan starts. Useful to avoid the fan kicking in for brief light use. |
| LightSensorSettings RunningTime | 5, 10, 15, 30, 60 min | How long the fan continues to run after the light turns off. |

#### Schedule

| Entity | Description |
|---|---|
| TrickleDays Weekdays | Whether trickle (continuous background) ventilation runs on weekdays. |
| TrickleDays Weekends | Whether trickle ventilation runs on weekends. |
| SilentHours On | Enables silent hours. During this period, humidity and light triggers are suppressed — the fan won't activate automatically. Trickle ventilation and boost are not affected. |
| SilentHours Start Time | When silent hours begin. |
| SilentHours End Time | When silent hours end. |

#### Automatic cycles

| Entity | Options | Description |
|---|---|---|
| Automatic Cycles | Off, 30 min, 60 min, 90 min | Forced ventilation cycle interval. When enabled, the fan runs a full ventilation cycle at this interval regardless of sensor triggers. |

#### Heat distribution (Calima / Svara / Levante only)

These settings are only active when **Mode** is `HeatDistributionMode` (see Diagnostic below). In this mode the fan acts as a heat distributor, running at different speeds based on temperature.

| Entity | Range | Description |
|---|---|---|
| HeatDistributorSettings TemperatureLimit | 5–50 °C | The temperature threshold that determines which fan speed is used. |
| HeatDistributorSettings FanSpeedBelow | 800–2400 rpm | Fan speed when temperature is **below** the limit. |
| HeatDistributorSettings FanSpeedAbove | 800–2400 rpm | Fan speed when temperature is **above** the limit. |

### Diagnostic

| Entity | Values | Description |
|---|---|---|
| Mode | See below | The operating mode configured on the fan. This is set via the PAX app and determines which features are active. |

| Mode value | Description |
|---|---|
| `MultiMode` | All sensors active (humidity, light, trickle). Default mode for bathroom use. |
| `DraftShutterMode` | Fan is connected to a draft shutter. |
| `WallSwitchExtendedRuntimeMode` | Fan controlled by a wall switch; continues running for a set time after the switch is turned off. |
| `WallSwitchNoExtendedRuntimeMode` | Fan controlled by a wall switch; stops immediately when switch is off. |
| `HeatDistributionMode` | Fan acts as a heat distributor, speed is based on temperature. Activates the HeatDistributor configuration entities. |

## Availability

Entities will show as **unavailable** after two consecutive failed connection attempts. This is expected when:
- The PAX app is open and connected (the fan supports only one BLE connection at a time).
- The fan is out of Bluetooth range.
- The fan has lost power.

HA will automatically reconnect and restore entity availability once the connection is re-established — no restart required.

## Good to know

- Configuration parameters are read from the device on startup and refreshed every 24 hours, so changes made from the PAX app will appear in HA within that window (or immediately after an HA restart).
- Fast scan interval is the polling interval used immediately after a write. It allows quick feedback when you change a setting, and reverts to the normal interval after 10 reads.
- If the PAX app and HA are both trying to connect at the same time, one of them will fail — the fan only supports a single BLE connection. Close the app after making changes to let HA reconnect.

### Bluetooth proxy (ESP32)

If your Home Assistant instance does not have Bluetooth, you can use an ESP32 running ESPHome as a Bluetooth proxy:

```yaml
esp32:
  board: esp32dev

bluetooth_proxy:
  active: true

esp32_ble_tracker:
```

Flash the config, then add the PAX device through the HA UI as normal.

## Thanks

- [@PatrickE94](https://github.com/PatrickE94/pycalima) for the Calima BLE driver
- [@MarkoMarjamaa](https://github.com/MarkoMarjamaa/homeassistant-paxcalima) for a good starting point for the HA implementation
