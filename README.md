# Klipper-Ender-3-V3-KE
Configs and stuff for running klipper on the Ender 3 V3 KE and possibly SE² without the nebula pad.

---

## Info
These configs are hopefully mostly drag and drop. This printer uses a GD32F303RET6 which in almost all cases can be treated as an STM32F103 (not 303 as some people say). If you are using a different setup to me (which is not unlikely) you will have to look through and change anything that's needed to fit your setup. They are imperfect as creality seems to do a lot of things that... don't make a lot of sense. They were originally dumped from the nebula pad and modified to get working on more recent builds of klipper. If you find any issues please open an issue to let me know! 

## My setup
I am using a pi 4 running mainsail os and kalico connected to the printer via Creality's serial to USB adapter ([Amazon](https://www.amazon.com/Creality-Sonic-Pad-Serial-Cable/dp/B0CFL5N319/ref=mp_s_a_1_1?crid=3KJQP9FARDMYA&dib=eyJ2IjoiMSJ9.pFnhUqBv4cuKFHbP5ICexNFIZgzGYcOXJHPROlFUslvK0fuS_mQXrdUSgCafDtjyxDuIaSFle6TBUwuxQqfihCQkfag_JSO_g23-OSvtQPOwpDblb_gt12PiqYPptFTUj94aAzxj58K2hR7oAsdKEZNfQRx2JJRr_ajKhjsJ-USxHYISjq5nwwu0n2Uerh7meeaJQkipmTiVT5Po0gLCLw.SCEXMVqklgPEOE0RnG69vfyV5OTgjC2vz_GVTm3R42Q&dib_tag=se&keywords=creality+serial+adapter&qid=1736591800&sprefix=creality+serial+adapter%2Caps%2C158&sr=8-1])).

---

# Updating the MCU
Please read ALL of this before even opening a terminal. If you find any issues please open an issue to let me know! At least as of January 11th 2025 this is optional as the current creality firmware on the MCU is protocol compatible with the most recent build of klipper. You don't need to do this if you want to run 'stock' klipper!

## Building klipper
To build klipper for the MCU on the printer's mainboard you can use the following plus the instructions in the [klipper repo](https://github.com/Klipper3d/klipper):

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

Disabling SWD at startup is optional. If using the 6 pin serial port on the board, instead use USART1 PA10/PA9 as the communication interface³.

## Building katapult
The katapult github can be found [here](https://github.com/Arksine/katapult).
I couldn't get the stock bootloader to load a custom build of klipper no matter how hard I tried. Maybe it's my specific board. Not sure. If you want to try, build klipper with a bootloader offset of 28kb and flash to offset 0x08007000 of the MCU using the SWD debug probe detailed later.  

Use the same settings as above but I recommend leaving disable SWD on startup OFF to make troubleshooting easier.

---
## Flashing
OK... this part is fun. Not user friendly. You will need one of the following: a pi pico, pi debug probe, St-Link or GD-link as well as a soldering iron or a hot air station. Or a butt ton of dexterity and 3 hands. You will also need to install openocd and a debugger such as gdb on whatever device your debug probe will be plugged in to (probably your pi/other secondary MCU device).

I will give instructions on using a pi pico as a debug probe and a pi 4.

Install the pi [debug probe firmware](https://github.com/raspberrypi/debugprobe) to your pi pico. Solder wires to or plug wires into the GND, GP2 and GP3 headers on your pi pico. Open up the bottom of your printer and on the control board look to the right of the header that the long rainbow ribbon cable used for serial connections is plugged in to. You will see some pins. Four of those pins are used for SWD (serial wire debug). Solder GND on the pico to GND on the board, GP2 on the pico to CLK on the board, and GP3 on the pico to DIO on the board. You also need a way to ground the NRST pin on the board. You can use a jumper wire between NRST and GND or something similar.  

Now that your debug probe is set up, use a micro USB to USB A cable to plug the pico in to your pi. `cd` to the directory containing gd32f3x.cfg and run `sudo openocd -f interface/cmsis-dap.cfg -f gd32f3x.cfg -c "gdb_memory_map disable"` to launch the openocd server just barely after you turn the printer on, as the version of klipper from creality disables SWD at startup. If it gives an error saying unable to read IDR, ground the NRST pin and try again. Our goal is to establish a connection before the SWD interface gets disabled. If openocd establishes a connection and *then* starts complaining, that is fine. Leave it and move on. 

In another terminal window or tab run `gdb`. I recommend doing this from a phone using ssh in case you need to fiddle with the NRST pin. In gdb run `target remote :3333` to connect to openocd which acts as a driver for the pico. If it says the connection was dropped or rejected, ground the NRST pin and try again. After this release NRST run `mon reset halt` at about the same time or just after. If it says processor not halted you were too early running the command and the mcu was reset after you ran it. If it says could not read IDR you were too late running the command and the SWD interface was already disabled. Our goal is to halt the processor mid startup so it won't disable our interface.  

I ~~demand you make~~ recommend making a backup of your current mcu flash storage in case something goes wrong. You can use `dump binary memory </location/of/backup.bin> 0x08000000 0x7000` or the dump mcu script in the scripts directory of klipper to do so over the serial port. No backups, no sympathy. After you have your backup, erase the flash of the microcontroller using `mon flash erase_address 0x08000000 0x7000`. The first address is the start of the flash memory on this device, and the second number is 448 KB (458752 bytes in decimal)¹, which is more than enough space. Next flash katapult to the beginning of the flash memory: `mon flash write_image </location/of/katapult.bin> 0x08000000`. Replace the path with where your katapult.bin file you built is stored. If it says wrote 0 bytes double check the location. After that you can run `mon reset` to reset the mcu, or you can manually power cycle your printer to boot into katapult.  

Once you have finished that, go back to your pi and `cd` to the katapult directory. Run the following command to install your build of klipper: `python scripts/flashtool.py -b 230400 -d /dev/ttyUSB0`. If your klipper build is not in the default place use -f to specify its location. If you are using the serial port on the board you may have to change what device you are targeting depending on how it is connected. Once that is done it should automatically reset and start klipper on the mcu. If it doesn't, manually turn the printer off and then back on again. Once you have verified that your secondary mcu can see it you can shut down the printer, desolder anything you have soldered in there and close the printer back up.

Thou art now done with the task thou hast set out to complete. Probably. I mean, I don't know. Maybe your intent was to go to a Wendy's. How did you end up here?

---
## Updating in the future
Instead of going through all that in the future, you now have katapult! Just open your printer and briefly ground the NRST pin twice in quick succession to enter katapult. From there you can update using kiauh/flashtool without the debug probe.

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


## Notes
¹ I am not sure if this is the full size of the flash but it is more space than klipper takes up. Everything after *should* be 00s until you get to other registers.
² If someone wants to flash klipper on their SE and try these, please let me know if it works! the hardware is nearly identical so I hope it does.  
³ This could be wrong. If someone is doing this please let me know.
