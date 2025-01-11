# Klipper-Ender-3-V3-KE
Configs and stuff for running klipper on the Ender 3 V3 KE without the nebula pad.

---

## Info
These are hopefully mostly drag and drop. If you are using a different setup to me (which is not unlikely) you will have to look through and change anything that's needed. They are imperfect as creality seems to do a lot of things that... don't make a lot of sense.

## My setup
I am using a pi 4 running mainsail is and kalico connected to the printer via Creality's serial to USB adapter ((Amazon)[]).

---

# Updating the MCU
Please read ALL of this before even opening a terminal.

## Building klipper
To build klipper for the MCU on the printer's mainboard you can use the following and the instructions in the (klipper repo)[]:

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

Disabling SWD at startup seems to be optional. If using the 5 pin serial port on the board, instead use USART1 PA10/PA9 as the communication interface.

## Building katapult
Github can be found (here)[].
I couldn't get the stock bootloader to load klipper no matter how hard I tried. Maybe its my specific board. Not sure. If you want to try, build klipper with a bootloader offset of 28kb and flash to offset 0x08007000 of the MCU using the SWD debug probe detailed later.  

Use the same settings as above but I recommend leaving disable SWD on startup OFF to make troubleshooting easier.

---
## Flashing
OK... this part is fun. Not user friendly. You will need a pi pico, pi debug probe, St-Link or GD-link as well as a soldering iron, a hot air station or a butt ton of dexterity and 3 hands. You will also need to install openocd and a debugger such as gdb on whatever device your debug probe will be plugged in to (probably your pi/other secondary MCU device).

I will give instructions on using a pi pico as a debug probe and a pi 4.

Install the pi (debugger firmware)[] to your pi pico. Solder wires to or plug wires into the pin headers of GND, GP2 and GP3 on your pi pico. Open up the bottom of your printer and on the control board look to the right of the header of the long rainbow ribbon cable used for serial connections you will see some pins. Four of those pins are used for SWD (serial wire debug). Solder GND on the pico to GND on the board, GP2 on the pico to CLK on the board, and GP3 on the pico to DIO on the board. You may also need a way to ground the NRST pin on the board.  

Now that your debug probe is set up use a micro USB to USB A cable to plug the pico in to your pi. cd to the directory containing gd32f3x.cfg and run ` ` to launch the openocd server just barely after you turn the printer on, as the version of klipper from creality disables SWD at startup. If it gives an error saying unable to read IDR try again or ground the NRST pin.  

In another terminal window or tab run `gdb`. I recommend doing this from your phone using ssh in case you need to fiddle with the NRST pin. In gdb run `target remote :3333` to connect to openocd which acts as a driver for the pico. if it says the connection was dropped or rejected, ground the NRST pin. After this run `mon reset halt`. If the NRST pin is still grounded it will say processor not halted, so you will have to release the NRST pin as or just after you run the command. If it says could not read IDR follow the same troubleshooting steps for starting openocd, but release NRST as or just after you run the command.  

I recommend making a backup of your current flash storage in case something goes wrong. You can use ` ` or the dump MCU script in the scripts directory of klipper to do so. After you have your backup erase the flash of the microcontroller using `mon flash erase_address 0x08000000 0x7000`. The first address is the start of the flash memory on this device, and the second number is " in bytes, which is more than enough for our purposes. Next flash katapult to the beginning of the flash memory: `mon flash write_image </location/of/katapult.bin> 0x08000000`. Replace the path with where your katapult.bin file you built is stored. If it says wrote 0 bytes double check the location. After that you can run `mon reset` to restart the MCU again or manually power cycle your printer.  

Once you have finished that go back to your pi and cd to the katapult directory. Run the following command to install your build of klipper: ` `. Once that is done it should automatically reset and start klipper on the mcu. If it doesn't, manually turn the printer off and then back on again.  

Thou art now done with the task thou hast set out to complete. Probably. I mean, I don't know. Maybe your intent was to go to a Wendy's. How did you end up here?

---
## Updating in the future
Instead of going through all that in the future, you now have katapult! Just open your printer and quickly ground the NRST pin twice to enter katapult. From there you can update using flashtool like when you installed klipper/kalico/whatever your preferred firmware is.

## Disclaimer
I am not responsible if something goes wrong! Make backups! Only do to your hardware what you feel comfortable with!
