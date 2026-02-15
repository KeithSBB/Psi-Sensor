# Psi-Sensor

**PSI Sensor** is an ESP32-based, ESPHome-integrated multi-sensor device designed to measure environmental and electromagnetic phenomena often associated with reported paranormal activity.  

The sensor suite provides real-time data for:
- Temperature
- Relative humidity
- Barometric pressure
- Ambient sound level (dB SPL)
- Electromagnetic field strength (EMF, primarily 50/60 Hz AC)
- Quantum-like noise for true random number generation (TRNG)

All sensors are exposed in Home Assistant and can be used to influence automations or AI personalities. The companion project **[Barabashka](https://github.com/KeithSBB/barabashka)** demonstrates how these measurements can dynamically modify Grok AI responses in Home Assistant.

## Photos

**Assembled PSI Sensor**  
![Top level image](images/topAnnotated.png "The PSI Sensor")

**EMF Sensor Coil**  
![EMF Coil](images/emfcoil.png "EMF Sensor Coil")

**Inside the enclosure (top and bottom)**  
![Inside top and bottom](images/opentopbottom.png)

**Under the top lid**  
![Under top lid](images/opentop.png)

**Bottom half internals**  
![Inside bottom](images/openbottom.png)

## Hardware Components

| Qty | Component | Notes / Link |
|-----|-----------|--------------|
| 1 | ESP-WROOM-32 ESP32 30-pin development board + breakout shield | Main MCU, Wi-Fi, dual-core, plenty of GPIO |
| 1 | GY-BME280 5 V compatible breakout (temperature, humidity, pressure) | I²C interface |
| 1–6 | INMP441 omnidirectional I²S digital microphone module | One primary mic for sound level; extras can be used for noise TRNG or directional array |
| 1 | QWORK demonstration induction coil (primary + secondary + iron core) | Used as EMF pickup coil |
| 1 | LMV358 dual op-amp breakout | Low-voltage rail-to-rail op-amp for EMF signal amplification |
| 1 | KBP206 bridge rectifier (2 A, 600 V) | Full-wave rectification of induced AC to DC for ESP32 ADC |
| – | Resistors, capacitors, prototyping PCB, enclosure, wires, 5 V power supply | As needed for signal conditioning and mounting |

## Subsystem Details

### 1. System on a Chip – ESP32-WROOM-32
- Dual-core Tensilica LX6 processor, 240 MHz, 520 KB SRAM  
- Integrated Wi-Fi + Bluetooth for OTA updates via ESPHome  
- Multiple ADC channels (18 total, 12-bit), I²C, I²S, and plenty of GPIO  
- Powered via USB or 5 V external supply; 3.3 V logic throughout

### 2. Temperature, Humidity & Barometric Pressure – BME280
- Combined digital sensor (Bosch)  
- Temperature: ±0.5 °C accuracy, –40…+85 °C  
- Humidity: ±3 % RH  
- Pressure: ±1 hPa  
- Connected via I²C (default address 0x76 or 0x77) to ESP32 GPIO 21 (SDA) and 22 (SCL)  
- 5 V tolerant breakout used; powered directly from 5 V rail with 3.3 V logic level shifting handled by the module

### 3. Ambient Sound – INMP441 I²S Microphone
- Digital 24-bit I²S output, 44.1 kHz sample rate capable  
- High dynamic range (61 dB SNR) and omnidirectional pattern  
- Primary microphone connected to ESP32 I²S pins:  
  - SCK → GPIO 14  
  - WS → GPIO 15  
  - SD → GPIO 32  
  - L/R = GND (left channel)  
- ESPHome calculates RMS sound level and reports as dB SPL (A-weighted approximation)

### 4. Electromagnetic Field (EMF) Detector
Custom analog front-end built around the induction coil:

1. **Pickup coil** – Secondary winding of the QWORK demonstration coil (several hundred turns) acts as a search coil. Changing magnetic fields (especially 50/60 Hz power-line hum or anomalous fluctuations) induce a small AC voltage.
2. **Amplification** – LMV358 configured as non-inverting AC amplifier (gain ≈ 100–500, adjustable with feedback resistor). Single-supply operation from 5 V with bias at ~2.5 V.
3. **Rectification** – KBP206 full-wave bridge rectifier converts amplified AC to pulsating DC.
4. **Smoothing & buffering** – RC low-pass filter (e.g., 10 kΩ + 4.7 µF) creates a DC level proportional to EMF strength.
5. **ADC input** – Smoothed DC fed to ESP32 ADC1_CH4 (GPIO 32 or similar spare channel). ESPHome reads raw ADC value and converts to approximate mG or arbitrary units.

This design is sensitive to low-frequency magnetic fields and is a classic “ghost hunter” style EMF meter.

### 5. Quantum Noise Random Number Generator (TRNG)
Paranormal enthusiasts often look for anomalies in truly random processes. We provide a hardware-derived entropy source:

- One or more spare INMP441 microphones are run at maximum gain with the input shorted or pointed at a quiet area.  
- The least-significant bits of the raw I²S audio stream contain thermal and shot noise that is effectively unpredictable.  
- ESPHome’s `random` component extracts entropy and exposes a binary sensor that flips based on true random bits, plus a sensor for raw entropy quality.

Alternative: ADC noise from an unused channel or reverse-biased transistor junction can be added if desired.

## ESPHome Configuration (example)

```yaml
esphome:
  name: psi-sensor
  platform: ESP32
  board: esp32dev

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

api:
ota:

i2c:
  sda: 21
  scl: 22

i2s_audio:
  i2s_lrclk_pin: 15   # WS
  i2s_bclk_pin: 14    # SCK

sensor:
  # BME280
  - platform: bme280
    temperature:
      name: "PSI Temperature"
    pressure:
      name: "PSI Pressure"
    humidity:
      name: "PSI Humidity"
    address: 0x76
    update_interval: 10s

  # Sound level (primary mic)
  - platform: i2s_audio
    i2s_audio_id: mic_primary
    microphone:
      number: 0
      name: "Primary Mic"
    adc_pin: 32   # SD
    sample_rate: 44100
    bits_per_sample: 24
    channel: left
    pdm: false
    filters:
      - lambda: return x * 0.00002;  # rough calibration to volts
    on_raw_value:
      then:
        - sensor.template.publish:
            id: sound_level_db
            state: !lambda 'return 20 * log10(id(rms).state) + 94;'  # approximate dB SPL

  # EMF analog reading (ADC)
  - platform: adc
    pin: 33
    name: "PSI EMF Raw"
    update_interval: 500ms
    filters:
      - multiply: 0.0012   # scale to approximate mG (calibrate empirically)
    unit_of_measurement: "mG"

  # Optional TRNG entropy sensor
  - platform: random
    name: "PSI True Random Bit"
    binary: true

binary_sensor:
  - platform: template
    name: "PSI Paranormal Anomaly Flag"
    lambda: |-
      if (id(emf_raw).state > 5.0 || id(temperature).state < 15) {
        return true;
      } else {
        return false;
      }
