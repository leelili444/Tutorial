# Hardware specification
## Main components of MX Motion IMU
|Category|Designator|Part Number|Core Function|
|:-|:-|:-|:--- |
| **Main MCU** | U200 | **STM32F405RGT6** | High-performance ARM Cortex-M4 microcontroller, responsible for real-time control and data processing. |
| **Inertial Sensor** | U2 | **ICM-42688-P** | High-end 6-axis IMU (Gyroscope + Accelerometer), used for attitude and motion measurement. |
| **Bus Communication** | U6 | **TJA1051T/3** | High-speed **CAN Bus Transceiver**, used for industrial/vehicular network communication. |
| **Non-Volatile Memory** | U4 | **AT24C256** | 256 Kbit **EEPROM**, used for storing configuration parameters and calibration data. |
| **Power Management** | U1 | **LD39150DT33-R** | **LDO Regulator**, provides a stable **3.3V** supply for the digital circuitry. |
| **Clock Source** | Y200 | **16 MHz Crystal** | **External Clock Source** for the MCU, providing high-precision timing for the system and peripherals. |

## Online manuals
We mainly need the following manuals:
* [STM32F405RGT6](https://www.st.com/resource/en/datasheet/dm00037051.pdf)
* [ICM42688-P](https://invensense.tdk.com/download-pdf/icm-42688-p-datasheet/)
  