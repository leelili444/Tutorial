---
description: Basics of IMU data and attitude estimation  
---

# Overview
This document summarizes the physical meaning of typical IMU signals (with notes for the ICM42688P), and outlines common, practical methods for computing attitude (orientation) from IMU data. It focuses on basics and engineering tips useful when implementing AHRS on embedded systems.

---

## IMU raw signals and their meaning
- Accelerometer (ax, ay, az)
  - Measures specific force along each sensor axis (m/s²) including gravity.
  - For many sensors raw counts → scale factor → units in g or m/s².
  - ICM42688P typical ranges: ±2, ±4, ±8, ±16 g (selectable).
  - When stationary, output ≈ gravity vector (|a| ≈ 9.80665 m/s²).

- Gyroscope (gx, gy, gz)
  - Measures angular rate about each axis (°/s or rad/s).
  - Typical ranges (ICM42688P): up to ±2000 °/s.
  - Integrating gyroscope rates yields orientation change; biases cause drift.

- Temperature
  - Used for temperature compensation of biases and scales.

- Time delta (dt)
  - Time between consecutive samples. Accurate dt (in seconds) is critical for numeric integration.

- Noise & error modes
  - White noise (measurement noise) → increases uncertainty.
  - Bias (offset) and bias instability → slow drift, must be estimated/compensated.
  - Scale factor errors and axis misalignment → systematic errors.
  - Vibration and aliasing → proper filtering and sampling needed.

---

## Units and conversions (practical)
- Gyro: degrees/sec → radians/sec: ω_rad = ω_deg * (π / 180).
- Accel: g → m/s²: a_mps2 = a_g * 9.80665.
- Ensure sensor registers are converted using datasheet scale factors for selected full-scale ranges.

---

## Coordinate frames and conventions
- Define clearly:
  - Sensor/body frame axes (x,y,z) and sign convention.
  - Navigation frame (e.g., NED, ENU, NWU) used by filters.
- Mismatched conventions cause sign errors in roll/pitch/yaw.

---

## Preprocessing and calibration
1. Bias (offset) estimation:
   - Measure mean when sensor is stationary to estimate constant gyro bias and accel bias.
   - For gyro: subtract estimated bias before integrating.
2. Scale & alignment:
   - Apply scale factors and a 3x3 calibration matrix if needed.
3. Temperature compensation:
   - Track bias vs temperature and compensate if necessary.
4. Filtering:
   - Low-pass accelerometer to reduce vibration effects.
   - Possibly low-pass or complementary high-pass on gyro to mitigate low-frequency drift tradeoffs.

---

## Basics of attitude estimation (concepts)
- Two main raw methods:
  1. Gyroscope integration
     - Integrate angular rates to update orientation (quaternion or DCM).
     - Pros: good short-term accuracy, high bandwidth.
     - Cons: drifts over time due to bias.
  2. Accelerometer (and magnetometer) correction
     - Accelerometer gives gravity direction → correct roll/pitch.
     - Magnetometer gives heading (yaw) relative to magnetic north.
     - Without magnetometer yaw is unobservable (drifts slowly).

- Sensor fusion idea:
  - Use gyro for fast dynamics (integration).
  - Use accelerometer (and magnetometer) as absolute reference to correct slowly-varying errors.
  - Filters: complementary filter, Mahony, Madgwick, Kalman/Extended Kalman Filter (EKF), or dedicated AHRS libraries (e.g., FusionAhrs).

---

## Simple math primitives

- Quaternion integration (continuous):
  - Represent angular velocity ω = [ωx, ωy, ωz] (rad/s) as quaternion 0 + ω.
  - q_dot = 0.5 * q ⊗ ω_quat
  - Discrete update (first-order): q_{k+1} = q_k + q_dot * dt ; then normalize q_{k+1}.

- Euler angles from accelerometer (approximate when no linear accel):
  - roll = atan2(ay, az)
  - pitch = atan2(-ax, sqrt(ay² + az²))
  - yaw is not observable from accel.

- Simple complementary filter (1st order):
  - compute roll/pitch from accel (slow absolute)
  - integrate gyro to get angle estimate (fast)
  - combined_angle = α * (angle_from_gyro) + (1 - α) * (angle_from_accel)
  - α in [0..1], choose near 0.98–0.995 for high trust in gyro short-term.

---

## Common AHRS algorithms
- Madgwick filter
  - Gradient-descent algorithm, computationally cheap, suitable for microcontrollers.
- Mahony filter
  - Error feedback control style, good performance and low complexity.
- EKF / Error-state Kalman Filter
  - Models sensor biases and noise, more complex but provides optimal estimates under Gaussian assumptions.
- Dedicated libraries (e.g., "Fusion" used in project)
  - Provide FusionAhrsUpdateNoMagnetometer, FusionOffset, etc. Use documented settings (gain, rejection constants).

---

## Practical tips for embedded implementation
- Always use consistent, accurate dt (seconds). dt jitter degrades integration.
- Keep critical sections short when sharing sensor structures between ISRs and tasks.
- Estimate and remove gyro bias before AHRS updates to reduce yaw drift.
- Normalize quaternions after updates to avoid numeric error accumulation.
- Use accelerometer rejection or recovery logic during high linear acceleration or vibration (many AHRS implementations include acceleration rejection thresholds).
- If no magnetometer is present, expect yaw to drift slowly; consider external references (GPS heading, visual odometry) for long-term yaw correction.
- Tune AHRS gain: higher gain → faster convergence to accel reference but more accel-induced noise; lower gain → smoother but slower correction.
- Use proper low-pass filters and anti-aliasing if sampling at lower rates than sensor bandwidth.

---

## Expected ICM42688P-specific notes
- Data rates: sensor supports high-rate sampling (hundreds to thousands Hz). Ensure DMA + ISR + RTOS tasks can handle chosen rate.
- Full-scale and register conversion: check datasheet for LSB-to-unit factors for chosen FS settings.
- Use FIFO/DMA to reduce CPU overhead; ensure decoding yields atomic snapshot (timestamp/dt).
- Watch for SPI timing and chip-select patterns when using DMA; ensure decode makes a consistent sample.

---

## Example minimal fusion flow (pseudocode)
- Acquire raw sample and dt
- Preprocess:
  - apply scale, remove biases, temperature compensation
  - convert gyro to rad/s
- Update gyro bias estimator (optional)
- AHRS update (quaternion):
  - q_dot = 0.5 * q ⊗ [0, ω]
  - q += q_dot * dt
  - if accel valid:
    - correct q using accel (Mahony/Madgwick step or EKF measurement update)
  - normalize q
- Convert q → Euler (roll, pitch, yaw)
- Publish telemetry

---
## Wrap up
Today we covered the basics of IMU raw signals, their physical meaning, common preprocessing steps, and foundational concepts for attitude estimation using sensor fusion techniques. This knowledge is essential for implementing robust AHRS systems on embedded platforms.