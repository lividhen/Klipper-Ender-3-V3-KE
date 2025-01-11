# Klipper-Ender-3-V3-KE
Configs and stuff for running klipper on the Ender 3 V3 KE without the nebula pad.

---

## Info
These are hopefully mostly drag and drop. This printer uses a GD32F303RET6 which in almost all cases can be treated as an STM32F103 (not 303 as some people say). If you are using a different setup to me (which is not unlikely) you will have to look through and change anything that's needed. They are imperfect as creality seems to do a lot of things that... don't make a lot of sense. If you find any issues please open an issue to let me know! 

## My setup
I am using a pi 4 running mainsail is and kalico connected to the printer via Creality's serial to USB adapter ([Amazon](https://www.amazon.com/Creality-Sonic-Pad-Serial-Cable/dp/B0CFL5N319/ref=mp_s_a_1_1?crid=3KJQP9FARDMYA&dib=eyJ2IjoiMSJ9.pFnhUqBv4cuKFHbP5ICexNFIZgzGYcOXJHPROlFUslvK0fuS_mQXrdUSgCafDtjyxDuIaSFle6TBUwuxQqfihCQkfag_JSO_g23-OSvtQPOwpDblb_gt12PiqYPptFTUj94aAzxj58K2hR7oAsdKEZNfQRx2JJRr_ajKhjsJ-USxHYISjq5nwwu0n2Uerh7meeaJQkipmTiVT5Po0gLCLw.SCEXMVqklgPEOE0RnG69vfyV5OTgjC2vz_GVTm3R42Q&dib_tag=se&keywords=creality+serial+adapter&qid=1736591800&sprefix=creality+serial+adapter%2Caps%2C158&sr=8-1])).

---

# Updating the MCU
Please read ALL of this before even opening a terminal. If you find any issues please open an issue to let me know!

## Building klipper
To build klipper for the MCU on the printer's mainboard you can use the following and the instructions in the [klipper repo](https://github.com/Klipper3d/klipper):

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

Disabling SWD at startup seems to be optional. If using the 6 pin serial port on the board, instead use USART1 PA10/PA9 as the communication interface.

## Building katapult
The katapukt github can be found [here](https://github.com/Arksine/katapult).
I couldn't get the stock bootloader to load klipper no matter how hard I tried. Maybe its my specific board. Not sure. If you want to try, build klipper with a bootloader offset of 28kb and flash to offset 0x08007000 of the MCU using the SWD debug probe detailed later.  

Use the same settings as above but I recommend leaving disable SWD on startup OFF to make troubleshooting easier.

---
## Flashing
OK... this part is fun. Not user friendly. You will need a pi pico, pi debug probe, St-Link or GD-link as well as a soldering iron, a hot air station or a butt ton of dexterity and 3 hands. You will also need to install openocd and a debugger such as gdb on whatever device your debug probe will be plugged in to (probably your pi/other secondary MCU device).

I will give instructions on using a pi pico as a debug probe and a pi 4.

Install the pi [debug probe firmware](https://github.com/raspberrypi/debugprobe) to your pi pico. Solder wires to or plug wires into the pin headers of GND, GP2 and GP3 on your pi pico. Open up the bottom of your printer and on the control board look to the right of the header of the long rainbow ribbon cable used for serial connections you will see some pins. Four of those pins are used for SWD (serial wire debug). Solder GND on the pico to GND on the board, GP2 on the pico to CLK on the board, and GP3 on the pico to DIO on the board. You may also need a way to ground the NRST pin on the board.  

Now that your debug probe is set up use a micro USB to USB A cable to plug the pico in to your pi. cd to the directory containing gd32f3x.cfg and run `sudo openocd -f interface/cmsis-dap.cfg -f gd32f3x.cfg -c "gdb_memory_map disable"` to launch the openocd server just barely after you turn the printer on, as the version of klipper from creality disables SWD at startup. If it gives an error saying unable to read IDR try again or ground the NRST pin.  

In another terminal window or tab run `gdb`. I recommend doing this from your phone using ssh in case you need to fiddle with the NRST pin. In gdb run `target remote :3333` to connect to openocd which acts as a driver for the pico. if it says the connection was dropped or rejected, ground the NRST pin. After this run `mon reset halt`. If the NRST pin is still grounded it will say processor not halted, so you will have to release the NRST pin as or just after you run the command. If it says could not read IDR follow the same troubleshooting steps for starting openocd, but release NRST as or just after you run the command.  

I recommend making a backup of your current flash storage in case something goes wrong. You can use `dump binary memory </location/of/backuo.bin> 0x08000000 0x7000` or the dump MCU script in the scripts directory of klipper to do so. After you have your backup erase the flash of the microcontroller using `mon flash erase_address 0x08000000 0x7000`. The first address is the start of the flash memory on this device, and the second number is 448 KB (458752 bytes in decimal), which is more than enough space for our purposes. Next flash katapult to the beginning of the flash memory: `mon flash write_image </location/of/katapult.bin> 0x08000000`. Replace the path with where your katapult.bin file you built is stored. If it says wrote 0 bytes double check the location. After that you can run `mon reset` to restart the MCU again or manually power cycle your printer.  

Once you have finished that go back to your pi and cd to the katapult directory. Run the following command to install your build of klipper: `python scripts/flashtool.py -b 230400 -d /dev/ttyUSB0`. If your klipper build is not in the default place use -f to specify. Once that is done it should automatically reset and start klipper on the mcu. If it doesn't, manually turn the printer off and then back on again.  

Thou art now done with the task thou hast set out to complete. Probably. I mean, I don't know. Maybe your intent was to go to a Wendy's. How did you end up here?

---
## Updating in the future
Instead of going through all that in the future, you now have katapult! Just open your printer and quickly ground the NRST pin twice to enter katapult. From there you can update using flashtool like when you installed klipper/kalico/whatever your preferred firmware is.

## Disclaimer
I am not responsible if something goes wrong! Make backups! Only do to your hardware what you feel comfortable with!

---
## Sources and resources
pi debug probe: https://github.com/raspberrypi/debugprobe  
MCU info & datasheet: https://www.gigadevice.com/product/mcu/mcus-product-selector/gd32f303ret6  
gdb docs: https://www.sourceware.org/gdb/documentation/  
Openocd docs: https://openocd.org/doc/html/index.html  
Klipper build conf: https://forum.sovol3d.com/t/klipper-on-sv05/1702/2  

Using creality adxl: https://www.reddit.com/r/klippers/comments/1e5ocz5/guide_hooking_up_your_creality_adxl345gsensor_to/
