# WCS1500 Hall Effect Current Sensor with ESP32

A non-invasive DC/AC current measurement system using the **WCS1500 Hall-effect sensor** and an **ESP32** microcontroller. Measures up to **±200A DC** without breaking the circuit — just pass the wire through the sensor's 9mm hole.

![WCS1500 Hall Effect Current Sensor Module](https://i.ibb.co/jHYsv7z/HW-671-013.jpg)


## How it works

The WCS1500 contains a linear Hall-effect IC inside a housing with a 9mm through-hole. When current flows through a wire passed through the hole, the resulting magnetic field is detected and converted into a proportional analog voltage on the **Aout** pin.

At zero current, the output sits at **VDD / 2** (ratiometric). As current increases in one direction the voltage rises, and in the other direction it drops. The relationship is linear:

```
V_out = (VDD / 2) + (I × Sensitivity)
```

With VDD = 4.6V (typical USB-powered ESP32 breakout board):
- Zero current output: 4.6 / 2 = **2.3V**
- Sensitivity: 11 mV/A × (4.6 / 5.0) = **10.12 mV/A** (ratiometric scaling)
- At +100A: 2.3 + (100 × 0.01012) = **3.312V**
- At -100A: 2.3 - (100 × 0.01012) = **1.288V**

The output is **ratiometric**, meaning it scales proportionally with the supply voltage. The sensitivity value of 11 mV/A is specified at 5V — at other supply voltages, multiply by VDD / 5.0.



## Components

| Component | Purpose |
|-----------|---------|
| **WCS1500 module** | Hall-effect current sensor (Winson) |
| **ESP32 breakout board** (30-pin) | Microcontroller with 12-bit ADC |
| **10kΩ resistor** | Voltage divider (high side) |
| **6.8kΩ resistor** | Voltage divider (low side) |
| **LED (built-in or external)** | Calibration status indicator |
| **220Ω resistor** | Current limiting for external LED (optional) |



## WCS1500 module pinout

Looking at the module PCB, the pins from left to right are:

| Pin | Function |
|-----|----------|
| **VCC** | Power supply (3.0–12V, typically 5V) |
| **Dout** | Digital overcurrent alarm output (TTL) |
| **GND** | Ground |
| **Aout** | Analog voltage output (proportional to current) |

> **Important:** Connect **Aout** to the ESP32 ADC, not Dout. The Dout pin is a digital overcurrent alarm that sits at 0V during normal operation — connecting it instead of Aout will result in constant 0V readings.



## WCS1500 sensor specifications

| Parameter | Min | Typical | Max | Unit |
|-----------|-----|---------|-----|------|
| Supply voltage (VDD) | 3.0 | — | 12 | V |
| Supply current | — | 3.5 | 6.0 | mA |
| Zero current output (at 5V) | 2.4 | 2.5 | 2.6 | V |
| Sensitivity | 10 | 11 | 12 | mV/A |
| DC current range (at 5V) | — | ±200 | — | A |
| AC current range (at 5V, RMS) | — | 150 | — | A |
| Bandwidth | — | 23 | — | kHz |
| Conductor through-hole | — | 9.0 | — | mm |
| Isolation voltage | — | 4000 | — | V |
| Temperature drift | — | ±0.2 | — | mV/°C |
| Output voltage range | 0.3 | — | VDD-0.3 | V |

Source: [WCS1500 Datasheet (Winson)](https://www.graylogix.in/wp-content/uploads/2022/02/WCS1500.pdf)



## Wiring

### Why a voltage divider is needed

At 5V supply, the WCS1500 output ranges from 0.3V to 4.7V. The ESP32 ADC can only handle 0–3.3V — anything above that risks damaging the pin. A resistor divider scales the sensor output to a safe range.

### Voltage divider

```
WCS1500 Aout ─── 10kΩ ───┬─── ESP32 GPIO34
                          │
                        6.8kΩ
                          │
                         GND
```

Divider ratio: 6800 / (10000 + 6800) = **0.4048**

This maps the sensor's 0.3–4.7V output to approximately 0.12–1.90V at the ESP32 pin.

### Full connections

| WCS1500 Module | Connect to |
|----------------|------------|
| VCC | ESP32 breakout 5V rail |
| GND | ESP32 GND |
| Aout | Through voltage divider → ESP32 GPIO34 |
| Dout | (Optional) ESP32 GPIO35 for overcurrent alarm |

The calibration status LED uses **GPIO2**, which is the built-in LED on most ESP32 30-pin development boards — no extra wiring required. If your board doesn't have a built-in LED on GPIO2, wire an external LED through a 220Ω current-limiting resistor:

```
ESP32 GPIO2 ─── 220Ω ─── LED(+) ─── LED(-) ─── GND
```

### Notes

- **GPIO34** is an input-only ADC pin on the ESP32 — it cannot be used as an output, which makes it a safe choice for analog reading.
- Place a **0.1µF ceramic capacitor** between VCC and GND close to the sensor module to reduce noise.
- Pass the current-carrying wire through the centre of the 9mm hole for best accuracy.
- If current reads negative when expecting positive, flip the wire direction through the hole.



## How the code works

### Startup calibration

With no current flowing, the code takes 500 readings from the sensor and averages them to establish the zero-current offset voltage. This compensates for component tolerances, supply voltage variation, and ESP32 ADC non-linearity. The expected offset is approximately VDD / 2, but the actual measured value through the divider and ADC may differ slightly.

### LED status indicator

A status LED on GPIO2 provides a clear visual signal of when it's safe to apply power to the circuit being measured:

| LED state | Meaning | Action |
|-----------|---------|--------|
| **Blinking** (~4 Hz) | Calibration in progress | Do **NOT** apply load current |
| **Rapid flashing** (10× fast) | Zero offset out of range — wiring error | Check wiring and reset |
| **Solid ON** | Calibration complete | Safe to energise the circuit |

This is especially useful when running the system without a serial monitor attached, or in field deployments where you need an unambiguous "ready" signal before switching on the load.

### Reading the sensor

Each reading averages 100 rapid ADC samples to reduce noise. The raw ADC value is converted to a pin voltage, then divided by the divider ratio to recover the actual sensor output voltage before the resistor divider.

### Current calculation

The current is calculated by subtracting the calibrated zero offset from the sensor voltage and dividing by the ratiometric sensitivity:

```
I = (V_sensor - V_zero) / Sensitivity
```

### Moving average filter

A sliding window of 50 samples smooths out ADC noise. Each new reading replaces the oldest value in the circular buffer. The filter introduces a slight delay but provides much more stable readings.

### Serial output

The code prints three values:
- **Sensor V** — the recovered WCS1500 output voltage (before the divider)
- **Current (A)** — the instantaneous calculated current from the latest reading
- **Filtered (A)** — the moving average current over the last 50 readings



## Calibration

### Automatic zero offset

The code auto-calibrates at startup. Ensure **no current is flowing** through the WCS1500 during the first 3 seconds after power-on.

### Measuring actual VDD

For best accuracy, measure the voltage between VCC and GND on the sensor module with a multimeter and update the `VDD` constant in the code. USB-powered ESP32 boards typically supply 4.5–4.8V on the 5V rail, not a full 5.0V.

### Temperature drift

The WCS1500 has a temperature drift of ±0.2 mV/°C. At 10.12 mV/A sensitivity, this is approximately ±0.02 A/°C. Over a 30°C temperature swing, the drift is roughly ±0.6A — negligible for high-current monitoring but worth noting for precision applications.



## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Constant 0V / 0.3V reading | Dout connected instead of Aout | Swap the wire to the Aout pin on the module |
| Zero offset reads 0.000V | Sensor not powered or Aout not reaching GPIO34 | Measure VDD at the sensor (should be ~5V) and Aout with a multimeter (should be ~2.5V) |
| Zero offset far from VDD/2 | Wrong VDD constant in code | Measure actual VDD with a multimeter and update the constant |
| Readings very noisy | Missing bypass capacitor or poor connections | Add 0.1µF cap near sensor; check breadboard connections |
| Current always negative | Wire direction through sensor hole is reversed | Flip the wire 180° through the hole |
| Readings saturate / clamp | Current exceeds ±200A or divider resistors are wrong | Verify resistor values; check that current is within sensor range |
| LED never turns solid ON | Calibration failed sanity check (rapid flashing) | Check sensor wiring, VDD value in code, and ensure no current during startup |
| LED doesn't light at all | Wrong GPIO for your board variant | Try GPIO5 or wire an external LED to a free GPIO and update `LED_PIN` |



## Resolution and accuracy

With the ESP32's 12-bit ADC and the voltage divider:

| Parameter | Value |
|-----------|-------|
| ADC resolution | 12-bit (4096 counts) |
| Voltage per ADC count | 0.806 mV |
| Sensor voltage per count (after divider recovery) | 1.99 mV |
| Current per ADC count | ~0.2A |
| Practical accuracy | ±1–2A (limited by ESP32 ADC noise) |

For higher resolution, consider upgrading to an **ADS1115** external 16-bit ADC (I²C), which provides approximately 0.017A per count with the same sensor.



## References

- [WCS1500 Datasheet (Winson)](https://www.graylogix.in/wp-content/uploads/2022/02/WCS1500.pdf)
- [WCS1500 Module — Micro Robotics (South Africa)](https://www.robotics.org.za/WCS1500-MOD)
- [ESP32 ADC Documentation (Espressif)](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/adc.html)

---

## License

MIT License. See [LICENSE](LICENSE) for details.
