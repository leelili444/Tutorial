---
description: New project for MX Motion IMU board.
---

# Start with stm32CubeIDE
## New STM32 Project
* Use File -> New -> STM32 Project to create it
* The IDE pops up a 'Target selection' window. In the section 'Commercial Part Number', input the processor model 'STM32F405RGT6', and on the right side of the window, several MCUs are filtered out. Click on the line of STM32F405RGT6, and click on 'Next'.
![select target MCU](image.png)

*  Then, give a proper name to this project. You can leave the rest of the options at their default values. Click Finish and the IDE will open your project and open a tab called 'myIMU.ioc'. This tab is the window of STM32CubeMX Device Configuration Tool.  
![name the project](image-1.png)

## Configure the MCU
Now you can configure the MCU as per the [imu schematic](https://github.com/leelili444/IMU/blob/main/Hardware/Schematic.pdf) of MX Motion IMU board, which you can find in repo.

The first tab in the Device Configureation tool is the 'Pin out and configuration' of MCU. Some essential pins and peripherals used in the IMU board will be configured in the following steps.
![Device configuration tool](image-2.png)
### Clock configuration
There is a external clock in the form of a 8MHz active crystal oscillator on the IMU board, which is a  
connected with pin5, which are the input pin to connect with external oscillator.    

![connection Between the Crystal Oscillator and Processor](image-3.png)  
![OSC_in and OSC_out](image-4.png)
* RCC Mode and Configuration
  * In the Device Configuration Tool, expand the 'System Core' section and choose the 'RCC' (Reset and Clock Controller) peripheral.
  * In the active 'Mode' panel, change the HSE (High-Speed Clock) setting to Crystal/Ceramic Resonator.
  * It will enable and label the two oscillator pins as RCC_OSC_IN and RCC_OSC_OUT.  
![Set HSE to Crystal/Ceramic option](image-15.png)
* Clock Tree Configuration
  By default it still uses the HSI (High-speed Internal) as the main clock source, unless we customize the clock in the tab 'Clock Configuration' on the top.
  * Set the HCLK and PLL Source
  HCLK is the bus clock which frequency is up to 168 MHz for stm32f405. Set the HCLK to 168MHz; the tool will auto-adjust the parameters. Then, change the PLL Source to HSE and set input frequency with 16Mhz, and set the HCLK to 168MHz again.  
  ![Clock tree configuration](image-6.png)  

  Timer1, Timer2 and Timer3 are used in MX Motion IMU. Based on the datasheet, individual timer frequency shown as:

| Timer | Type | Connected Bus | Timer Frequency Used |
| :---: | :---: | :---: | :---: |
| TIM1, TIM8 | Advanced-control Timer | APB2 | 168 MHz |

![Timer-1 and Timer-8](image-7.png)


### SWD Configuration
MX Motion IMU uses Stlink for code debugging.Enable SWD as follows: 
* Under 'System Core' on the left side, click on SYS, and in the 'Debug' option in the Mode Configuration, select 'Serial Wire'. 
* Set the 'Timebase Source' option to 'TIM3'. This is recommended because when using FreeRTOS, a HAL-based timer provides more reliable and dedicated timekeeping than the default system tick (SysTick) timer.
* As soon as you do this, the PA13 and PA14 pins will be set as SWDIO and SWCLK.  
  ![SWD configuration](image-10.png)

## Enable FreeRTOS
* From the left, click FreeRTOS under 'Middleware and Software' and select CMSIS_V2 to enable FreeRTOS.    
![Enalble FreeRTOS](image-11.png)
* The IDE will pop up a window to remind you to set the Newlib：
  ![Newlib setting](image-12.png)
* Task
  There is a default task. We will use it to run the LED blinking.
  ![Default task](image-13.png)

## Code Generation
Press 'ctrl + S' to save the 'Device Configuration Tool', and the IDE will start to generate the code. 

We will look into the code more next time and build the first FreeRTOS task for IMU board.
  