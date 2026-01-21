# MSP (Multiwii Serial Protocol) V2

MSP is a binary protocol used for communication between flight controllers and various peripherals in the RC (Radio Control) hobbyist community, particularly in drones and multirotors. MSP V2 is an updated version of the original MSP protocol, offering enhanced features and capabilities.

## Key Features of MSP V2
- **Binary Protocol**: MSP V2 uses a binary format for data transmission, which is more efficient than text-based protocols.
- **Extended Command Set**: MSP V2 includes a wider range of commands and data types compared to its predecessor, allowing for more complex interactions.
- **Parameter Management**: It supports reading and writing configuration parameters, enabling dynamic adjustments to flight controller settings. In our IMU project, this allows for real-time tuning of sensor fusion parameters and calibration settings.
- **Telemetry Support**: MSP V2 can transmit telemetry data, providing real-time feedback on system status, sensor readings, and other critical information.

## Integration with IMU Data Processing
In our IMU data processing project, MSP V2 is utilized to facilitate communication between the GUI and the IMU board. This integration allows for:
- **Real-time Data Transmission**: The IMU can send real-time arhs and raw data to GUI.
- **Configuration Updates**: Flight controllers can send configuration commands to the IMU module to adjust settings such as sampling rates, filter parameters, and calibration data.
- **Telemetry Feedback**: The IMU can provide telemetry data back to the GUI, allowing for monitoring of sensor health and performance.

## Parameters saving
There is no external flash chip on the board, so the parameters are saved in the internal flash memory of the STM32 microcontroller. The parameters are stored in a dedicated section of the flash memory, which is reserved for non-volatile data storage. This allows the parameters to persist even when the board is powered off.

When a parameter is updated via MSP, the new value is written to the flash memory. The next time the board is powered on, it reads the parameters from flash and applies them to the IMU data processing algorithms.

## Update breakdown
- **MSP Command Implementation**: Implemented MSP V2 commands for reading and writing parameters related to IMU data processing. Use parameter to select different communication protocols via serial port.
- **Flash Memory Management**: Developed routines for saving and retrieving parameters from the STM32 internal flash memory.
- **GUI Integration**: Updated the GUI to support MSP V2 communication for parameter management and telemetry display.
- **Testing and Validation**: Conducted extensive testing to ensure reliable communication and parameter persistence across power cycles.