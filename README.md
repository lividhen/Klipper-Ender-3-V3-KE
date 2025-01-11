# Klipper-Ender-3-V3-KE
Configs and stuff for running klipper on the Ender 3 V3 KE without the nebula pad.
## Building klipper
This is optional at least as of 1/11/2025 as the version of klipper running on the MCU is protocol compatible with the latest build of klipper.  
To build klipper for the MCU on the printer's mainboard you can use the following:

```
[*] Enable extra low-level configuration options
    Micro-controller Architecture (STMicroelectronics S
    Processor model (STM32F103)  --->
[ ] Only 10KiB of RAM (for rare stm32f103x6 variant)
[*] Disable SWD at startup
    Bootloader offset (8KiB bootloader)  --->
    Clock Reference (8 MHz crystal)  --->
    Communication interface (Serial (on USART2 PA3/PA2)
(230400) Baud rate for serial port
()  GPIO pins to set at micro-controller startup
```

Disabling SWD at startup seems to be optional. If using katapult set bootloader offset to 8kb.

## Flashing
OK... this part is fun. Not user friendly. You will need a pi pico, pi debug probe, St-Link or GD-link. You will also need to install openocd and a debugger such as gdb on whatever device your debug probe will be plugged in to (probably your pi/other secondary MCU device).

I will give instructions on using a pi pico as a debug probe and a pi 4.

Solder wires to 
