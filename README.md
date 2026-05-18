# Budget Bionic Hand

**15-DOF EMG-controlled prosthetic hand. Total cost under $55. Built from zero electronics knowledge.**

---

## What it does

Reads raw muscle electrical signals (EMG) from a forearm electrode cuff, processes them through a 4-stage analog signal chain, classifies the gesture using a Random Forest running on the Arduino itself, and actuates 15 servo-driven fingers to match the intended pose.

Demo: [link to video -- to be added on project completion, target 10 September 2026]

**Timeline:** 7 April 2026 -- 10 September 2026 (~22 active weeks, ~270 hours total)

---
## Table of contents

- [System Overview](#System_Overview)
- [Signal Chain](#Signal_Chain)
- [Hardware](#Hardware)
- [Firmware architecture](#Firmware_architecture)
- [ML pipeline](#ML_pipeline)
- [Electrode placement](#Electrode_placement)
- [Repo structure](#repo-structure)
- [Build sequence](#build-sequence)
- [Key references To Learn From](#Key_references_To_Learn_From)
- [Results](#Results)
---
## System Overview

```
Forearm electrodes
      |
      v
[Stage 1] Instrumentation Amplifier   -- 101x gain, rejects common-mode noise
      |
      v
[Stage 2] Twin-T 50 Hz Notch Filter   -- kills UAE mains hum
      |
      v
[Stage 3] Sallen-Key Bandpass 20-500 Hz -- isolates EMG band
      |
      v
[Stage 4] Half-wave rectifier + RC envelope (tau = 47 ms)
      |
      v
Arduino Nano ADC (10-bit, 500 Hz sampling)
      |
      v
Random Forest classifier (4 features, exported to C++ via sklearn-porter)
      |
      v
PCA9685 I2C driver --> 15x SG90 servos --> InMoov 3D-printed hand
```

---

## Signal Chain

All op-amp stages use the **LM358P** (DIP-8, single supply 5 V).

The LM358P output swings only 0.05 V to 3.5 V on a 5 V supply. A 2.5 V DC bias is applied at every stage input so the AC signal can swing symmetrically without clipping.

### Stage 0 -- Bias circuit

Two matched 10 kohm resistors from 5 V to GND. Midpoint = 2.5 V.

```
V_bias = 5V x R2 / (R1 + R2) = 5V x 10k / 20k = 2.5 V
```

### Stage 1 -- 3-op-amp instrumentation amplifier

```
Gain = 1 + (2 x Rf / Rg) = 1 + (2 x 100k / 2k) = 101x
```

- Rf = 100 kohm x2 (feedback around OA1, OA2)
- Rg = 2 kohm (single resistor between OA1 IN- and OA2 IN-)
- OA3 difference amplifier: 4 matched 10 kohm resistors

CMRR depends on matching the four OA3 resistors. Select four from the kit using a multimeter -- target within 1% of each other.

### Stage 2 -- Twin-T 50 Hz notch filter

```
f_notch = 1 / (2pi x R x C)
R = 1 / (2pi x 50 x 100nF) = 31.8 kohm --> 33 kohm (E24 standard)
Centre resistor: R/2 = 15 kohm
Centre capacitor: 2C = 220 nF
Actual f_notch with 33 kohm: 48.2 Hz -- effective against 50 Hz hum
```

Followed by a unity-gain op-amp buffer to restore drive capability.

### Stage 3 -- Sallen-Key bandpass 20-500 Hz

High-pass (20 Hz):
```
C = 100 nF
R = 1 / (2pi x 20 x 100nF) = 79.6 kohm --> 82 kohm
Actual cutoff: 19.4 Hz
```

Low-pass (500 Hz):
```
C = 10 nF
R = 1 / (2pi x 500 x 10nF) = 31.8 kohm --> 33 kohm
Actual cutoff: 482 Hz
```

### Stage 4 -- Envelope detector

```
Half-wave rectifier: 1N4148 signal diode (stripe = cathode, faces output)
RC smoother: R = 4.7 kohm, C = 10 uF electrolytic
tau = R x C = 4700 x 0.00001 = 47 ms

Constraint: 1/500Hz = 2ms << tau << 1/20Hz = 50ms  (satisfied)
```

The output is a slowly-varying DC voltage proportional to muscle activation level. This feeds directly into Arduino A0.

---

## Hardware

### Bill of materials (under $55 total)

All components sourced from AliExpress. Total budget: under $55.

| Component | Marking / notes | Qty | Role |
|-----------|-----------------|-----|------|
| LM358P op-amp (DIP-8) | Notch = pin 1 end. Pin 8 = VCC, pin 4 = GND | 3 chips | OA1-OA6 across all stages |
| Resistor 10 kohm | Brown-black-orange-gold | 6 | 2x bias divider, 4x OA3 difference amp (matched) |
| Resistor 100 kohm | Brown-black-yellow-gold | 2 | INA feedback Rf (one per OA1, OA2) |
| Resistor 2 kohm | Red-black-red-gold | 1 | INA gain-set Rg -- changing this changes gain |
| Resistor 33 kohm | Orange-orange-orange-gold | 4 | 2x notch top arms, 2x LP Sallen-Key |
| Resistor 15 kohm | Brown-green-orange-gold | 1 | Notch centre (R/2 approximation) |
| Resistor 82 kohm | Grey-red-orange-gold | 2 | HP Sallen-Key (20 Hz cutoff) |
| Resistor 4.7 kohm | Yellow-violet-red-gold | 1 | Envelope RC time constant |
| Capacitor 100 nF ceramic | Marked 104 | 6 | 2x notch bottom arms, 2x HP filter, 2x chip decoupling |
| Capacitor 220 nF ceramic | Marked 224 | 1 | Notch centre (2C approximation) |
| Capacitor 10 nF ceramic | Marked 103 | 2 | LP Sallen-Key |
| Capacitor 10 uF electrolytic | POLARISED -- stripe = negative | 1 | Envelope smoother |
| Capacitor 100 uF electrolytic | POLARISED -- stripe = negative | 1 | Bulk 5 V rail decoupling |
| Capacitor 1000 uF electrolytic | POLARISED -- stripe = negative | 3 | Servo 5 V rail (prevents brown-out) |
| 1N4148 signal diode | Stripe = cathode = faces output | 1 | Half-wave rectifier Stage 4 |
| Arduino Nano | ATmega328P, 10-bit ADC, 32 KB flash | 1 | MCU, ADC sampling, classifier inference |
| PCA9685 I2C PWM driver | Address 0x40 default, 16 channels | 1 | Drives all 15 servos over I2C |
| SG90 micro servo | 180 deg range, ~2.5 kg/cm stall, 5 V | 15 | 5 fingers x 3 joints each |
| Copper tape electrode | Conductive adhesive, cut to ~2x2 cm | 3 | IN+, IN-, REF skin contacts |
| Conductive gel | Pharmacy ultrasound gel acceptable | 1 tube | Reduces skin impedance to under 10 kohm |
| Shielded cable | 3 runs ~50 cm each | 1 | Electrode leads -- outer braid connects to GND |

**Mechanical:** InMoov open-source hand (inmoov.fr). 3D-printed PLA, 30% infill, 3 perimeters, 0.2 mm layer height. Tendons: 20 lb nylon fishing line through PTFE tubing.

---

## Firmware architecture

Three concurrent tasks using a cooperative scheduler:

| Task | Rate | Trigger | Notes |
|------|------|---------|-------|
| ADC sampling | 500 Hz | Timer1 ISR | 2 ms period, 10-bit resolution |
| Servo update | 20 Hz | Main loop (50 ms) | Smooth interpolation -- step toward target each cycle |
| Haptic feedback | Event-driven | ISR flag | 200 ms pulse via 2N2222 transistor on detection of grip |

**PCA9685 driver:** Written from scratch using `Wire.h`. No third-party libraries. Implements `init()`, `setFrequency(50)`, `setPWM(channel, on, off)` against the NXP register map (datasheet sections 7-8).

**I2C address:** 0x40 (PCA9685 default). Decoupling: 3x 1000 uF electrolytic capacitors across the servo 5 V rail to prevent brown-out resets under simultaneous load.

**Gesture presets:** 6 poses defined as 15-element servo angle arrays (one angle per joint):

| Gesture | Description |
|---------|-------------|
| open | All fingers extended, hand flat |
| fist | All fingers fully flexed |
| pinch | Thumb + index flex, others extended |
| point | Index extended, others flexed |
| peace | Index + middle extended, others flexed |
| thumbs-up | Thumb extended, others flexed |

**Proportional control:** `map(analogRead(A0), rest_val, flex_val, 0, 180)`. Startup calibration: 2 s at rest samples `rest_val`, 2 s max flex samples `flex_val`.

**Ring buffer:** 8-sample moving average on ADC readings before mapping. Gesture hold: reading must be stable for 150 ms before gesture state changes (prevents false triggers).

---

## ML pipeline

### Data collection

- 500 ms windows at 500 Hz = 250 samples per window
- 50 labeled windows per gesture x 6 gestures = 300 total samples
- Collected via Python Serial capture script, saved to CSV
- 80/20 train/test split (240 train, 60 test)

### Feature extraction (4 features per window)

```python
import numpy as np

def extract_features(window):
    rms = np.sqrt(np.mean(window**2))
    zcr = np.sum(np.diff(np.sign(window)) != 0)
    wl  = np.sum(np.abs(np.diff(window)))   # waveform length
    mav = np.mean(np.abs(window))
    return [rms, zcr, wl, mav]
```

### Classifier

```python
from sklearn.ensemble import RandomForestClassifier

clf = RandomForestClassifier(n_estimators=10, max_depth=5, random_state=42)
clf.fit(X_train, y_train)
```

Hyperparameters chosen for Arduino Nano flash constraint (32 KB). `max_depth=5` limits exported C++ decision tree size. `n_estimators=10` keeps inference fast on 16 MHz ATmega328P.

- Exported to Arduino C++ via [sklearn-porter](https://github.com/nok/sklearn-porter)
- Fallback if flash too large: [EloquentArduino](https://github.com/eloquentarduino/EloquentArduino)

**Inference loop:** Every 250 ms, extract 4 features from ADC ring buffer, call exported classifier, trigger matching servo preset.

---

## Electrode placement

```
FOREARM (palm facing up, elbow left, wrist right)

+------------------------------------------+
|  ELBOW <----------------------- WRIST    |
|                                          |
|  [IN+]  [IN-]                   [REF]    |
|  2-3 cm apart, over muscle belly  bony   |
|  (inner forearm, palm side)       bump   |
+------------------------------------------+
```

- IN+ and IN- over the forearm flexor muscle belly, 2-3 cm apart along the muscle fibre direction
- REF on the bony wrist prominence (styloid process) -- no muscle underneath, picks up noise only

Skin prep: alcohol wipe, apply conductive gel under each pad. Target skin impedance below 10 kohm between electrodes.

---

## Repo structure

```
/
+-- firmware/
|   +-- main.ino          -- main loop, scheduler, gesture state machine
|   +-- pca9685.h         -- custom I2C driver (no library)
|   +-- classifier.h      -- exported Random Forest C++ (from sklearn-porter)
|   +-- gestures.h        -- 6 gesture servo angle arrays
|
+-- ml/
|   +-- collect_data.py   -- Serial capture + CSV labelling
|   +-- train.py          -- feature extraction, RF training, evaluation
|   +-- confusion_matrix.png
|   +-- dataset.csv
|
+-- signal_chain/
|   +-- falstad_ina.txt       -- Falstad circuit file for INA
|   +-- falstad_notch.txt     -- Falstad circuit file for twin-T
|   +-- falstad_bandpass.txt  -- Falstad circuit file for Sallen-Key
|   +-- derivations.md        -- Full component value derivations
|
+-- mechanical/
|   +-- bom.csv           -- complete component BOM with costs
|   +-- assembly_notes.md
|
+-- docs/
|   +-- technical_report.md
|   +-- build_guide.html
|   +-- roadmap.html
|
+-- README.md
```

---

## Build sequence

1. Bias circuit -- verify 2.5 V at midpoint before any op-amp work
2. Stage 1 INA -- verify burst activity on Serial Plotter on forearm flex
3. Stage 2 notch -- verify 50 Hz hum reduction at rest
4. Stage 3 bandpass -- verify flat signal during arm movement without flex
5. Stage 4 envelope -- verify smooth DC rise/fall tracking muscle activation
6. 3D print InMoov parts -- fingers first, palm overnight, forearm cuff
7. Mount servos at 90 deg neutral, route tendons, verify full ROM manually
8. PCA9685 driver -- verify all 15 servos respond without Arduino brown-out
9. Integrate ADC to servos -- proportional control, calibration routine
10. Collect ML dataset -- 300 labeled windows across 6 gestures
11. Train and export classifier -- verify accuracy on held-out test set
12. Live inference -- 6 gestures triggering correct hand poses

---

## Key references To Learn From

| Topic | Resource | URL |
|-------|----------|-----|
| Electronics fundamentals | All About Circuits textbook | allaboutcircuits.com/textbook |
| Op-amp and INA theory | Electronics-Tutorials op-amp series (parts 1-5) | electronics-tutorials.ws/opamp/opamp_1.html |
| LM358P | TI LM358 datasheet | ti.com/lit/ds/symlink/lm358.pdf |
| Op-amp applications | TI SLOA049 app note | ti.com/lit/an/sloa049b/sloa049b.pdf |
| Active filter design | Phil's Lab (YouTube) | youtube.com/@PhilsLab |
| I2C protocol | Ben Eater (YouTube) | youtube.com/@BenEater |
| PCA9685 register map | NXP PCA9685 datasheet | cdn-shop.adafruit.com/datasheets/PCA9685.pdf |
| PID control | Brett Beauregard -- Improving the Beginner's PID | brettbeauregard.com/blog/2011/04/improving-the-beginners-pid-introduction |
| Random Forests | StatQuest with Josh Starmer (YouTube) | youtube.com/@statquest |
| Hands-on ML | Kaggle Intro to Machine Learning | kaggle.com/learn/intro-to-machine-learning |
| EMG biology | Wikipedia -- Electromyography | en.wikipedia.org/wiki/Electromyography |
| Electrode placement standards | SENIAM | seniam.org |
| InMoov hand files | InMoov open-source project | inmoov.fr/hand-and-forarm |
| Circuit simulation | Falstad circuit simulator | falstad.com/circuit |
| Filter design verification | Analog Devices Filter Wizard | analog.com/en/design-center/design-tools-and-calculators/filter-wizard.html |
| Model export to C++ | sklearn-porter | github.com/nok/sklearn-porter |
| Embedded ML alternative | EloquentArduino | github.com/eloquentarduino/EloquentArduino |

---

## Results

| Metric | Value |
|--------|-------|
| Target total cost | under $55 (AliExpress sourced) |
| Gesture accuracy (test set) | TBD -- target Month 5 (Aug 2026) |
| Inference latency (on-device) | TBD -- target Month 5 |
| Envelope time constant (tau) | 47 ms |
| ADC sampling rate | 500 Hz (2 ms period) |
| Servo update rate | 20 Hz (50 ms period) |
| Gesture hold threshold | 150 ms stable before state change |
| INA gain | 101x (Rf=100k, Rg=2k) |
| Notch frequency | 48.2 Hz (33k / 100nF, targets 50 Hz) |
| HP cutoff | 19.4 Hz (82k / 100nF) |
| LP cutoff | 482 Hz (33k / 10nF) |
| Signal band passed | 19.4 Hz -- 482 Hz |
| Op-amp supply | 5 V single supply, 2.5 V bias |
| Op-amp output swing | 0.05 V -- 3.5 V (LM358P limitation) |
| ADC resolution | 10-bit (0--1023, 4.9 mV per step) |
| ML training samples | 300 (50 per gesture x 6 gestures) |
| ML features | 4 (RMS, ZCR, WL, MAV) |
| Random Forest depth | max_depth=5, n_estimators=10 |
| DOF | 15 (5 fingers x 3 joints) |
| Mechanical base | InMoov open-source hand |
| Print settings | PLA, 30% infill, 3 perimeters, 0.2 mm layers |
| Servo model | SG90 (~2.5 kg/cm stall torque, 180 deg) |
| Project start | 7 April 2026 |
| Target completion | 7 September 2026 |

