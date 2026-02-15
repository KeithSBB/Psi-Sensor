# Psi-Sensor

## Introduction

Paranormal research, spanning over a century from the Society for Psychical Research (founded 1882) to modern instrumental transcommunication (ITC) and field investigations, has repeatedly documented environmental correlates with reported anomalous events:

- **Cold spots** – localized temperature drops of 5–20 °C with no obvious cause
- **EMF spikes** – transient magnetic field fluctuations, especially in the 0.1–20 Hz range and at power-line frequencies
- **Barometric micro-changes** – small, rapid pressure shifts sometimes preceding poltergeist-like activity
- **Electronic voice phenomena (EVP)** – unexplained voices or sounds captured on audio recorders
- **Infrasound and atmospheric correlates** – low-frequency sound and humidity/pressure interactions that can induce unease or hallucinations in humans

While no single sensor has ever conclusively proven the existence of supernatural entities, multi-sensor correlation has become a cornerstone of serious field work. Simultaneous spikes across several independent channels are considered stronger evidence than isolated readings.

This **PSI Sensor** is designed explicitly for that multi-parameter approach. By combining precision temperature/humidity/pressure (BME280), high-resolution digital audio (INMP441), and a sensitive AC magnetic field detector (induction coil + LMV358 amplification) on a single ESP32 platform, it provides Home Assistant with a continuous, time-synced data stream that can reveal correlated anomalies in real time.

A particularly intriguing theoretical lens is the **Trickster hypothesis**, most thoroughly explored by George P. Hansen in *The Trickster and the Paranormal* (2001). Hansen argues that paranormal phenomena often exhibit characteristics of the archetypal trickster: deceptive, boundary-crossing, anti-structural, and stubbornly resistant to controlled scientific replication. Phenomena appear when scrutiny is low, vanish under rigid protocols, and sometimes seem to “play” with investigators or equipment. A multi-sensor device like the PSI Sensor acknowledges this elusive nature by collecting broad, passive data rather than forcing a narrow hypothesis. The hope is that patterns—or deliberate disruptions—may emerge across the combined channels that would be missed by single-axis “ghost meters.”

Ultimately, the PSI Sensor is both a practical environmental monitor and an open-ended experiment: objective data for skeptics, correlated anomalies for investigators, and (via projects like Barabashka) a way to let an AI assistant “feel” its surroundings in a uniquely paranormal-aware way.

**PSI Sensor** is an ESP32-based, ESPHome-integrated multi-sensor device designed to measure environmental and electromagnetic phenomena often associated with reported paranormal activity.

The sensor suite provides real-time data for:
- Temperature
- Relative humidity
- Barometric pressure
- Ambient sound level (dB SPL)
- Electromagnetic field fluctuations (EMF, AC-coupled)
- Optional true random number generation from microphone noise

All sensors are exposed in Home Assistant and can be used to influence automations or AI personalities. The companion project **[Barabashka](https://github.com/KeithSBB/barabashka)** shows how these measurements can dynamically modify Grok AI responses.

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

| Qty | Component | Notes |
|-----|-----------|-------|
| 1 | ESP-WROOM-32 30-pin development board + breakout shield | Main MCU, Wi-Fi, 3.3 V logic |
| 1 | GY-BME280 breakout | Temperature, humidity, pressure (I²C) |
| 1–6 | INMP441 omnidirectional I²S digital microphone | Primary for sound level; extras optional for noise TRNG |
| 1 | QWORK demonstration induction coil (primary + secondary) | EMF pickup (secondary winding used) |
| 1 | LMV358 dual op-amp breakout | Amplifies tiny AC voltage from coil |
| – | Prototyping PCB, enclosure, wires, resistors/capacitors for biasing | Minimal additional passives for op-amp bias and coupling |

**Important power note**: The entire system runs on **3.3 V**.  
- ESP32 board is powered via USB (internal regulator provides 3.3 V).  
- BME280 and all INMP441 modules are powered directly from the ESP32 3.3 V rail.  
- LMV358 is also powered from 3.3 V (rail-to-rail op-amp).

## Subsystem Details

### 1. System on a Chip – ESP32-WROOM-32
- Dual-core Tensilica LX6, 240 MHz, integrated Wi-Fi/BT
- 3.3 V logic throughout
- ADC1 channels used for analog inputs
- I²C and I²S peripherals for sensors

### 2. Temperature, Humidity & Barometric Pressure – BME280
- Bosch combined digital sensor
- Temperature: ±0.5 °C
- Humidity: ±3 % RH
- Pressure: ±1 hPa
- I²C interface (default address 0x76 or 0x77)
- Connected to GPIO 21 (SDA) and GPIO 22 (SCL)
- Powered from ESP32 3.3 V rail

### 3. Ambient Sound – INMP441 I²S Microphone
- Digital 24-bit I²S omnidirectional microphone
- Powered from 3.3 V
- Primary microphone pins:
  - SCK → GPIO 14
  - WS  → GPIO 15
  - SD  → GPIO 32
  - L/R = GND (left channel)
- ESPHome computes approximate dB SPL from RMS

### 4. Electromagnetic Field (EMF) Detector – AC-Coupled
The EMF subsystem is deliberately minimal:

1. **Pickup coil** – Secondary winding of the QWORK induction coil induces a tiny AC voltage proportional to changing magnetic fields.
2. **Direct connection to op-amp** – Coil leads go straight to one channel of the LMV358 configured as a high-gain non-inverting AC amplifier (gain ≈ 500–2000).
3. **Single-supply biasing** – Voltage divider biases non-inverting input to ~1.65 V; coupling capacitor blocks DC.
4. **Direct to ESP32 ADC** – Op-amp output connects directly to an ADC1 pin (e.g., GPIO 33). ESPHome samples the amplified AC waveform and can compute RMS or peak values.

This preserves fast transients often sought in paranormal investigations.

### 5. Optional Quantum Noise / True Random Number Generator
Extra INMP441 microphones provide high-entropy noise for true random bits—a nod to the idea that genuine randomness may itself fluctuate in the presence of anomalous influence.

## ESPHome Configuration (example)

```yaml
esphome:
  name: psi-sensor
  friendly_name: Psi Sensor

esp32:
  board: esp32dev
  framework:
    type: esp-idf


# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "bfDQrylQjaRIXEHp4zVZH14PNZ+lTxwDGOPy7JDHyeA="

ota:
  - platform: esphome
    password: "e157b91834728b632aec73aee40bffdb"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Psi-Sensor Fallback Hotspot"
    password: "Y7YlNB97ntWF"

captive_portal:

    
i2c:
  sda: GPIO21
  scl: GPIO22
  scan: true   # Helpful on first boot - shows if BME280 is detected

i2s_audio:
  - id: i2s_bus
    i2s_lrclk_pin: GPIO25  # WS
    i2s_bclk_pin: GPIO32  # SCK

microphone:
  - platform: i2s_audio
    id: psi_mic
    i2s_din_pin: GPIO33  #SD
    adc_type: external
    pdm: false
    channel: left
    bits_per_sample: 32bit  # Revert to 32bit but keep Arduino framework
    use_apll: false         # Turn OFF APLL; it can cause sync drift on some ESP32 revisions

globals:
  - id: emf_prev_raw
    type: float
    restore_value: no
    initial_value: '0.0'
  - id: emf_prev_filtered
    type: float
    restore_value: no
    initial_value: '0.0'
  - id: emf_envelope
    type: float
    restore_value: no
    initial_value: '0.0'
  - id: emf_bin_max
    type: float
    initial_value: '0.0'
  - id: bit_sum
    type: int
    initial_value: '0'
  - id: trial_count
    type: int
    initial_value: '0'
  - id: psi_z
    type: float
    initial_value: '0.0'


interval:
  # interval value determines bin size
  - interval: 1s
    then:
      - lambda: |-
          // Publish the peak that was collected over the previous ~1 s
          id(emf_bin_peak_sensor).publish_state(id(emf_bin_max));
          // Reset for the next bin (safe because rectified output should be >= 0)
          id(emf_bin_max) = 0.0;

  - interval: 4ms
    then:
      lambda: |-
        uint16_t raw = id(eb_avalanche).state;

        // Faster-converging dynamic threshold – starts low to match your observed avg (~1935)
        // and tracks quickly to eliminate sustained positive bias
        static uint16_t running_avg = 1900;  // Initial guess based on your recent stats (~1920–1940)
        running_avg = (running_avg * 3 + raw) / 4;  // Alpha = 1/4 (~25% weight on new sample)
        // This converges in ~10–20 seconds instead of minutes/hours

        bool bit = (raw > running_avg);  // Auto-centered decision

        id(bit_sum) += bit;
        id(trial_count)++;

        if (id(trial_count) >= 200) {
          float z = (static_cast<float>(id(bit_sum)) - 100.0f) / 7.071f;
          id(psi_z) = z;

          // Optional debug log every trial – helps confirm centering
          ESP_LOGD("psi_rng", "Trial: ones=%d/200 z=%.3f raw_avg~%u", id(bit_sum), z, running_avg);

          if (fabs(z) > 3.5f) {
            ESP_LOGW("psi_rng", "Psi event: Z=%.3f (ones=%d/200)", z, id(bit_sum));
          }

          id(bit_sum) = 0;
          id(trial_count) = 0;
        }

sensor:
  # Temp, Press, Humidity
  - platform: bme280_i2c
    address: 0x76   # Change to 0x76 if your module uses that (rare on GY-BME280)
    temperature:
      id: psi_sensor_temperature
      name: "Psi Sensor Temperature"
      oversampling: 16x
      unit_of_measurement: "°C"
      icon: "mdi:thermometer"
    pressure:
      id: psi_sensor_pressure
      name: "Psi Sensor Pressure"
      oversampling: 16x
      unit_of_measurement: "hPa"
      icon: "mdi:gauge"
    humidity:
      id: psi_sensor_humidity
      name: "Psi Sensor Humidity"
      oversampling: 16x
      unit_of_measurement: "%"
      icon: "mdi:water-percent"
    update_interval: 60s

  # Sound
  - platform: sound_level
    microphone: psi_mic
    passive: false
    measurement_duration: 250ms  #  window for stable readings
    rms:
      name: "Psi Sensor Sound Level (RMS dB)"
      icon: "mdi:volume-medium"
      unit_of_measurement: "dB"
      accuracy_decimals: 1
      filters:
        - sliding_window_moving_average:
            window_size: 10
            send_every: 5
        - throttle: 10s
    peak:
      id: psi_sensor_sound_peak_db
      name: "Psi Sensor Sound Peak (dB)"
      icon: "mdi:volume-high"
      unit_of_measurement: "dB"
      accuracy_decimals: 1
      filters:
        - throttle: 10s

  # EMF Sensors      
  # Raw EMF strength (relative, higher = stronger fluctuating fields)
  - platform: adc
    id: emf_raw_internal          # Renamed for clarity
    pin: GPIO34
    attenuation: auto
    update_interval: 50ms         # Fast enough to catch transients (20 samples/sec)
    accuracy_decimals: 3

  - platform: template
    #name: "EMF High-Pass Filtered"
    id: emf_hp_filtered
    unit_of_measurement: "V"  # Or arbitrary units
    icon: "mdi:waveform"
    accuracy_decimals: 3
    update_interval: 50ms  # Must match or be slower than raw update
    lambda: |-
      float alpha = 0.90;  // Lower = stronger high-pass (blocks slower components like residual 60 Hz)
      if (!id(emf_raw_internal).has_state() || isnan(id(emf_raw_internal).state)) {
        return {};
      }
      float current_raw = id(emf_raw_internal).state;
      if (isnan(id(emf_prev_filtered))) {
        id(emf_prev_filtered) = 0.0;
      }
      float filtered = alpha * (id(emf_prev_filtered) + current_raw - id(emf_prev_raw));
      id(emf_prev_raw) = current_raw;
      id(emf_prev_filtered) = filtered;
      return filtered;

    on_value:
      then:
        - lambda: |-
            if (!std::isnan(x) && x > id(emf_bin_max)) {
              id(emf_bin_max) = x;
            }

  # Optional: Derived "EMF Anomaly" sensor for alerts (triggers on sudden changes)
  - platform: template
    id: emf_anomaly_level
    name: "EMF Anomaly Level"
    unit_of_measurement: "V"
    device_class: "voltage"
    lambda: |-
      static float last = 0.0;
      float current = id(emf_bin_peak_sensor).state;     // read current value of the ADC sensor
      float delta = fabs(current - last);    // use fabs() for floating-point
      last = current;
      return delta;
    update_interval: 1s

  - platform: template
    name: "EMF Bin Peak"
    id: emf_bin_peak_sensor
    unit_of_measurement: "V"  # Change if you use mV, arbitrary units, etc.
    device_class: voltage
    accuracy_decimals: 3
    update_interval: never  # We manually publish every x s

  # Quantum Noise Sensors

  - platform: adc
    pin: GPIO35                # ADC1_CH7 - WiFi-safe, input-only, excellent for noise
    id: eb_avalanche
    #name: "Avalanche Raw Noise"  # Optional: makes it visible in HA as a debug sensor
    update_interval: 100ms     # Frequent polling for fast RNG sampling (adjust 50-200ms)
    attenuation: 12db          # Full ~0-3.3V range on ESP32 (matches your LMV358 output)
    raw: true                  # Returns raw 0-4095 ADC counts (best for bit extraction)
    accuracy_decimals: 0

  - platform: template
    name: "Psi RNG Z-Score"
    id: psi_z_score
    lambda: 'return id(psi_z);'
    unit_of_measurement: "σ"
    device_class: "voltage"
    accuracy_decimals: 3
    update_interval: 2s
    # Optional smoothing for HA graphs
    filters:
      - sliding_window_moving_average:
          window_size: 5
          send_every: 1

  - platform: template
    name: "Avalanche Raw Stats (10s)"
    update_interval: 100ms         # Run lambda every 100 ms
    lambda: |-
      static uint32_t sum = 0;
      static uint16_t count = 0;
      static uint16_t min_val = 4095;
      static uint16_t max_val = 0;

      uint16_t raw = id(eb_avalanche).state;
      sum += raw;
      count++;
      if (raw < min_val) min_val = raw;
      if (raw > max_val) max_val = raw;

      if (count >= 100) {
        float avg = static_cast<float>(sum) / count;
        ESP_LOGD("raw_stats", "Raw 1s: min=%u max=%u avg=%.1f", min_val, max_val, avg);
        sum = 0; count = 0; min_val = 4095; max_val = 0;
      }
      return NAN;

    # This should compute a score based on correlation of events 
  - platform: template
    name: "Psi Coincidence Score"
    id: psi_coincidence_score
    unit_of_measurement: ""
    icon: mdi:radioactive
    accuracy_decimals: 1
    update_interval: 5s                    # Faster than 10s → ~100 s window, more responsive
    lambda: |-
      #include <algorithm>

      const int WINDOW_SIZE = 20;           // 20 samples × 5 s = 100 s rolling window

      static float buf_rng[WINDOW_SIZE] = {0};
      static float buf_emf_env[WINDOW_SIZE] = {0};   // New: envelope-based strength
      static float buf_emf_hp[WINDOW_SIZE] = {0};    // Optional: keep rapid changes (lower weight)
      static float buf_sound[WINDOW_SIZE] = {0};
      static float buf_temp[WINDOW_SIZE] = {0};
      static float buf_humid[WINDOW_SIZE] = {0};
      static float buf_press[WINDOW_SIZE] = {0};

      static int idx = 0;

      float rng_z = fabs(id(psi_z));
      float emf_hp = fabs(id(emf_hp_filtered).state);                // ← Secondary (rapid only)
      float sound_peak = id(psi_sensor_sound_peak_db).state;
      float temp = id(psi_sensor_temperature).state;
      float humid = id(psi_sensor_humidity).state;
      float press = id(psi_sensor_pressure).state;

      const float mean_sound = -70.0;
      const float mean_temp = 22.0;
      const float mean_humid = 50.0;
      const float mean_press = 1013.0;

      // Normalization — tune the / X.0f divisors after watching live values
      float norm_rng = std::min(std::max(rng_z / 10.0f, 0.0f), 1.0f);
      float norm_emf_hp = std::min(std::max(emf_hp / 0.3f, 0.0f), 1.0f);     // Lower scale for rapid ripple
      float norm_sound = std::min(std::max((sound_peak - mean_sound + 40.0f) / 40.0f, 0.0f), 1.0f);
      float norm_temp = std::min(std::max(fabs(temp - mean_temp) / 10.0f, 0.0f), 1.0f);
      float norm_humid = std::min(std::max(fabs(humid - mean_humid) / 25.0f, 0.0f), 1.0f);
      float norm_press = std::min(std::max(fabs(press - mean_press) / 10.0f, 0.0f), 1.0f);

      buf_rng[idx] = norm_rng;
      buf_emf_hp[idx] = norm_emf_hp;            // ← Secondary
      buf_sound[idx] = norm_sound;
      buf_temp[idx] = norm_temp;
      buf_humid[idx] = norm_humid;
      buf_press[idx] = norm_press;

      idx = (idx + 1) % WINDOW_SIZE;

      float max_score = 0.0f;
      for (int i = 0; i < WINDOW_SIZE; i++) {
        float score =
          buf_rng[i] * 1.5f +
          buf_emf_hp[i] * 0.6f +                  // ← Lower weight on rapid ripple
          buf_sound[i] * 0.8f +
          buf_temp[i] * 0.4f +
          buf_humid[i] * 0.3f +
          buf_press[i] * 0.3f;

        if (score > max_score) max_score = score;
      }

      float final_score = std::min(max_score * 22.0f, 100.0f);   // Slight multiplier bump to recover old range

      return final_score;

binary_sensor:
  - platform: template
    name: "Psi RNG Anomaly"
    id: psi_event_active
    lambda: 'return fabs(id(psi_z)) > 3.5;'  # 3.5σ for sensitivity (your plot shows plenty)
    device_class: safety                  # Red shield icon when active
    filters:
      - delayed_off: 30s                  # Hold alert longer for events


output:
  - platform: ledc
    pin: GPIO26          # Red anode via resistor
    id: pwm_red
    frequency: 1000Hz    # 1 kHz is fine for LEDs (smooth, no flicker)
  - platform: ledc
    pin: GPIO27          # Green anode via resistor
    id: pwm_green
    frequency: 1000Hz
  - platform: ledc
    pin: GPIO14                                                                 # Blue anode via resistor
    id: pwm_blue
    frequency: 1000Hz

light:
  - platform: rgb
    name: "Phenomenon Alert LED"
    id: phenomenon_led   # Optional, useful if you reference it in lambdas/automations
    red: pwm_red
    green: pwm_green
    blue: pwm_blue
    restore_mode: ALWAYS_ON     # Keeps last state after reboot/power cycle
    gamma_correct: 2.8          # Default – good for perceived brightness balance
    default_transition_length: 1s  # Smooth fades for color changes
    effects:
      - strobe:                 # Your strobe effect
          name: "Strobe Alert"
          colors:
            - state: true
              brightness: 100%
              red: 100%
              green: 0%
              blue: 0%
              duration: 100ms
            - state: false
              duration: 100ms
      - random:                 # Add more as needed
      - flicker:
      - pulse:
