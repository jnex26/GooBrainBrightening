# GMLED-TMRF-MC / TG7111B LED Controller ESP32-C3 Replacement

This documents reverse engineering and replacement of a `GMLED-TMRF-MC` wireless LED controller daughterboard using an **ESP32-C3 SuperMini**.

The original controller uses a **TG7111B Bluetooth/RF MCU** and controls a two-channel LED driver through a device marked `2233B`.

The replacement retains the original LED power electronics and MOSFET drivers. Only the wireless/control daughterboard is replaced.

## ⚠️ Disclaimer and Safety Warning

I provide this information **without any warranty or guarantee of any kind**. Everything documented here is based solely on my own experience, measurements, reverse engineering and modifications performed on my particular device.

Your device may differ, even if it appears externally identical or uses the same PCB/controller markings.

If you choose to open, modify or otherwise work on your device, **you do so entirely at your own risk**.

This equipment is mains powered and parts of the circuit operate at **potentially lethal voltages**. Extreme precautions must be taken when working on or around the mains-voltage sections of the device. Dangerous voltages may also remain present after power has been disconnected due to charged capacitors.

Do not work on mains-powered equipment unless you understand the risks and have the appropriate knowledge, equipment and experience to do so safely.

The information in this repository should **not** be treated as a manufacturer-approved modification, safety certification, or guarantee that the same modification will be safe or suitable for another device.
---

## Original Hardware

The controller daughterboard is marked:

```text
GMLED-TMRF-MC
```

and contains a:

```text
TG7111B
```

wireless microcontroller.

The original controller advertises using Bluetooth Mesh and was observed with:

```text
Service UUID: 0x1827
Mesh Provisioning Service
```

The original controller was successfully detected as an:

```text
Unprovisioned Bluetooth Mesh Device
```

---

# Connector Pinout

The daughterboard connects to the main LED driver PCB through an 8-position interface.

The following was determined by voltage measurement and continuity tracing:

| Pin | Function | Notes |
|---:|---|---|
| 1 | ~11 V | Low-voltage supply |
| 2 | GND | Ground |
| 3 | 3.3 V | Existing controller 3.3 V rail/test point |
| 4 | Unused / unknown | No useful connection found |
| 5 | LED control channel 1 | Connects to `2233B` pin 1 |
| 6 | LED control channel 2 | Connects to `2233B` pin 2 |
| 7 | Unused / unknown | No useful connection found |
| 8 | NC | No connection found |

Pins **5 and 6** are the important control signals.

---

# Main LED Driver

The main PCB contains an SOIC-8 device marked:

```text
2233B
F2339E52
```

No reliable public datasheet has yet been identified for this device.

Tracing produced the following partial pinout:

| 2233B pin | Function |
|---:|---|
| 1 | Control input from daughterboard pin 5 |
| 2 | Control input from daughterboard pin 6 |
| 3 | ~11 V supply |
| 4 | GND |
| 5 | MOSFET gate drive through ~100 Ω resistor |
| 6 | MOSFET gate drive through ~100 Ω resistor |
| 7 | Not determined |
| 8 | Not determined |

Pins 5 and 6 each drive one of the LED channel MOSFETs.

The topology therefore appears to be approximately:

```text
                  Original TG7111B
                        │
                 ┌──────┴──────┐
                 │             │
               Pin 5         Pin 6
                 │             │
                 ▼             ▼
             2233B pin 1   2233B pin 2
                 │             │
                 │   2233B     │
                 │             │
             pin 5           pin 6
                 │             │
               ~100Ω         ~100Ω
                 │             │
                 ▼             ▼
              MOSFET        MOSFET
                 │             │
                 ▼             ▼
             LED CH 1      LED CH 2
```

This means the existing `2233B` and MOSFET power stage can be retained.

---

# ESP32-C3 Replacement

An **ESP32-C3 SuperMini** was used.

The two LED control channels are connected to:

```text
GPIO4 → LED control pin 5
GPIO5 → LED control pin 6
```

A **1 kΩ series resistor** was placed between each ESP32 GPIO and the LED controller input.

```text
ESP32-C3                     LED Driver
────────                     ──────────

GPIO4 ───── 1kΩ ───────────► Pin 5

GPIO5 ───── 1kΩ ───────────► Pin 6

GND ────────────────────────► Pin 2
```

GPIO4 and GPIO5 were chosen to avoid ESP32-C3 boot/strapping pins.

---

# ESP32 Power Supply

The original board provides approximately:

```text
11 V
```

on connector pin 1.

Rather than powering the ESP32 from the original controller's 3.3 V regulator, a separate **3.3 V buck converter** was used.

This avoids placing the ESP32's Wi-Fi/Bluetooth current demand on the original regulator.

```text
LED board
Pin 1
 ~11 V
   │
   ▼
┌───────────────┐
│ 3.3 V BUCK    │
└───────┬───────┘
        │
       3.3 V
        │
        ▼
   ESP32-C3 3V3

Pin 2 / GND
        │
        └────────► ESP32-C3 GND
```

The buck converter was tested at **12 V input** while powering an ESP32-C3 with both Bluetooth and Wi-Fi active and operated successfully.

---

# Decoupling

A **220 µF electrolytic capacitor** was fitted across the ESP32 supply.

```text
          3.3 V
            │
            ├────────► ESP32 3V3
            │
          + │
        220 µF
          - │
            │
GND ────────┴────────► ESP32 GND
```

The capacitor should be rated for at least **6.3 V**.

The ESP32-C3 SuperMini already contains local ceramic decoupling, so the 220 µF capacitor provides additional bulk capacitance for Wi-Fi/Bluetooth current spikes.

---

# Complete Wiring

```text
ORIGINAL LED BOARD                     ESP32-C3
──────────────────                     ─────────

Pin 1  ~11 V ──────► 3.3 V BUCK ─────► 3V3
                         │
                         │
                       220µF
                         │
Pin 2  GND ──────────────┴────────────► GND


Pin 5  CONTROL ◄────── 1kΩ ─────────── GPIO4

Pin 6  CONTROL ◄────── 1kΩ ─────────── GPIO5


Pin 3  3.3 V      NOT USED
Pin 4             NOT USED
Pin 7             NOT USED
Pin 8             NOT USED
```

---

# ESPHome

The following ESPHome configuration provides independent control of the two LED channels for testing:

```yaml
esphome:
  name: ceiling-light
  friendly_name: Ceiling Light

esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: esp-idf

logger:

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

mqtt:
  broker: 192.168.1.10
  username: !secret mqtt_user
  password: !secret mqtt_password
  topic_prefix: smarthome/lights/ceiling

output:
  - platform: ledc
    pin: GPIO4
    id: channel_1
    frequency: 1000 Hz

  - platform: ledc
    pin: GPIO5
    id: channel_2
    frequency: 1000 Hz

light:
  - platform: monochromatic
    name: "Test Channel 1"
    output: channel_1
    restore_mode: ALWAYS_OFF

  - platform: monochromatic
    name: "Test Channel 2"
    output: channel_2
    restore_mode: ALWAYS_OFF
```

This configuration has been tested and **successfully controls the LED channels**.

---

# CCT / Warm + Cool White Configuration

Once the two physical channels have been identified as warm and cool white, ESPHome can expose the fixture as a single colour-temperature light.

For example:

```yaml
output:
  - platform: ledc
    pin: GPIO4
    id: warm_pwm
    frequency: 1000 Hz

  - platform: ledc
    pin: GPIO5
    id: cool_pwm
    frequency: 1000 Hz

light:
  - platform: cwww
    name: "Ceiling Light"
    id: ceiling_light

    warm_white: warm_pwm
    cold_white: cool_pwm

    warm_white_color_temperature: 2700 K
    cold_white_color_temperature: 6500 K

    constant_brightness: true

    restore_mode: ALWAYS_OFF
```

If the warm and cool channels are reversed, simply swap:

```yaml
warm_white: cool_pwm
cold_white: warm_pwm
```

No hardware change is necessary.

---

# MQTT

ESPHome can communicate directly with an MQTT broker, allowing the light to be controlled by systems such as:

- Home Assistant
- Node-RED
- OpenHAB
- custom MQTT applications

Example:

```yaml
mqtt:
  broker: 192.168.1.10
  username: !secret mqtt_user
  password: !secret mqtt_password
  topic_prefix: smarthome/lights/ceiling
```

This allows the replacement controller to operate entirely locally without depending on the manufacturer's Bluetooth application or cloud services.

---

# Confirmed Findings

The following have been physically measured or continuity-tested:

- Original daughterboard is marked `GMLED-TMRF-MC`
- Controller uses a `TG7111B`
- Bluetooth Mesh provisioning advertisements are present
- Connector pin 1 is approximately 11 V
- Connector pin 2 is GND
- Connector pin 3 is 3.3 V
- Connector pin 5 connects to `2233B` pin 1
- Connector pin 6 connects to `2233B` pin 2
- `2233B` pin 3 is approximately 11 V
- `2233B` pin 4 is GND
- `2233B` pin 5 drives a MOSFET through approximately 100 Ω
- `2233B` pin 6 drives the second MOSFET through a similar resistor
- ESP32-C3 GPIO PWM control through the existing control inputs works
- ESP32-C3 operates successfully from a separate 3.3 V buck converter connected to the ~11 V rail

---

# Still Unknown

The exact identity and internal function of the IC marked:

```text
2233B
F2339E52
```

has not been established.

It appears to act as a dual-channel LED/MOSFET driver, but this should be considered an inference until a reliable datasheet is found.

The exact function of connector positions 4 and 7 has also not been established. No useful connection to the main LED driver was found during initial continuity testing. the devce worked fine without them. 

---

# Why Replace the Original Controller?

Replacing the TG7111B daughterboard with an ESP32-C3 provides:

```text
Original controller
        │
        X
        │
    ESP32-C3
        │
        ├── Wi-Fi
        ├── ESPHome
        ├── MQTT
        ├── Home Assistant
        ├── Node-RED
        └── fully local control
```

The original LED power supply, constant-current circuitry, MOSFETs and LED boards remain untouched.

Only the control layer is replaced.

---

## Disclaimer

This information was obtained by reverse engineering one particular light/controller combination.

Manufacturers may reuse PCB/controller names while changing circuit revisions.

**Verify voltages, ground, isolation and connector pinout on your own hardware before connecting an ESP32.**

Mains-powered LED drivers can contain lethal voltages even after power has been disconnected.
