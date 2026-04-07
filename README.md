# Parkinson's Motion Monitor

Real-time embedded movement monitoring system for Parkinson's disease symptoms using an STM32 IoT development board, an LSM6DSL IMU, FFT-based signal analysis, and Bluetooth Low Energy telemetry.

This project detects three motion states from wearable inertial data:

- Resting tremor in the 3-5 Hz band
- Dyskinesia in the 5-7 Hz band
- Freezing of gait (FOG) using step cadence and low-variance gait interruption logic

## Why this project matters

This repository demonstrates practical embedded ML-adjacent engineering rather than a notebook-only prototype. The system handles sensor initialization, interrupt-driven sampling, real-time windowed processing, stateful symptom detection, BLE broadcasting, and device-side status indication on constrained hardware.

For employers, the value is in the implementation:

- Real-time firmware in C++ on `mbed` with `PlatformIO`
- Interrupt + polling fallback for reliable sensor ingestion
- FFT-based frequency-domain classification with CMSIS-DSP
- State-machine-based gait freeze detection
- BLE GATT service design and characteristic updates
- Modular embedded architecture across acquisition, DSP, detection, comms, and UI layers

## System overview

### Hardware

- Board: `STM32 DISCO-L475VG-IOT01A`
- IMU: `LSM6DSL` accelerometer + gyroscope
- Transport: `I2C` at 400 kHz
- Wireless: `Bluetooth Low Energy`

### Software stack

- Framework: `mbed`
- Build system: `PlatformIO`
- DSP library: `CMSIS-DSP`
- Language: `C++`

## Detection pipeline

1. The firmware configures the LSM6DSL IMU for 52 Hz accelerometer and gyroscope sampling.
2. New samples arrive through the IMU interrupt line, with a polling fallback if interrupts stall.
3. Magnitude data from accelerometer and gyroscope streams is collected into a 3-second window.
4. The signal is normalized, Hann-windowed, zero-padded to 256 samples, and processed with a real FFT.
5. Frequency peaks are evaluated in symptom-specific bands:
   - Tremor: `3-5 Hz`
   - Dyskinesia: `5-7 Hz`
6. Detection results are smoothed with consecutive-window confirmation and EMA intensity tracking.
7. FOG logic runs as a separate gait state machine using step count, cadence, and motion variance.
8. Confirmed outputs are published over BLE and reflected in LED behavior.

## Repository structure

`src/main.cpp`
Application entry point, startup diagnostics, main loop, BLE event dispatch, and runtime status reporting.

`src/sensor.cpp`
LSM6DSL configuration, I2C register access, interrupt handling, raw sample ingestion, and step detection.

`src/signal_processing.cpp`
Window processing, FFT setup, spectral analysis, adaptive thresholds, and tremor/dyskinesia intensity estimation.

`src/fog_detection.cpp`
Freezing-of-gait state machine and gait transition logic.

`src/ble_comm.cpp`
BLE initialization, advertising, GATT service creation, and notification updates.

`src/led_control.cpp`
On-device visual feedback patterns for normal, tremor, dyskinesia, and FOG states.

`include/config.h`
Hardware registers, DSP window sizing, thresholds, and BLE configuration declarations.

## BLE data model

The firmware exposes three BLE characteristics:

- `TREMOR:<value>`
- `DYSK:<value>`
- `FOG:<value>`

Where:

- Tremor and dyskinesia intensities are scaled to `0-1000`
- FOG status is `0` or `1`

The advertised device name is currently `PD_Detector`.

## Build and flash

### Prerequisites

- `PlatformIO` installed locally
- STM32 ST-LINK access
- `STM32 DISCO-L475VG-IOT01A` board

### Commands

```bash
pio run
pio run -t upload
pio device monitor -b 115200
```

The current `platformio.ini` environment is:

- `disco_l475vg_iot01a`

## Tuning notes

Important configuration values are defined in `include/config.h`:

- Sample rate: `52 Hz`
- Window size: `156` samples
- FFT size: `256`
- Detection confirmation: `3` consecutive windows
- Step threshold and gait timing constants for FOG detection

These values can be adjusted depending on sensor placement, subject motion patterns, and desired sensitivity.

## What I would improve next

- Add logged BLE packet examples and a small mobile dashboard
- Add unit tests for the gait state machine and detection threshold logic
- Store timestamped symptom events for later analysis
- Validate thresholds on recorded datasets instead of hand-tuned motion trials
- Add power and latency measurements for wearable deployment
