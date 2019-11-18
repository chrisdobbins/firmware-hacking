This is an ATSC TV tuner box that follows a very common pattern in this space. On the outside, it has a minimal physical interface limited to an LCD indicating the current channel, three buttons to power on/off and change the channel, and one USB port for recording/watching media and firmware updates. This particular box, sold by Aluratek, also has a couple of games on it. I'm especially interested in modding them.

Internally, it's much more interesting. The processor is a SOC manufactured by MStar, a Mediatek subsidiary. The part number appears to be (from the u-boot printout) MST173B or MST173BD01A. The demodulator is also made by MStar; its part number is MSB1237. It has 32MB of DDR II SDRAM (ESMT part number M14D5121632A[https://www.endrich.com/fm/2/M14D5121632A-2.5BG2K.pdf] )  The tuner is under metal shielding, which I left intact since I didn't want to damage it. At the front of the board are three circuit pads marked GND, RX, TX. Since I lacked the proper tools to verify this ahead of time, on a hunch, I soldered a lead to the easiest pad (TX) and used a USB-to-TTL board set to 3.3V to see if anything would be printed out. Based on prior research of other MStar-based devices, it seemed reasonable to use a 115200 baud rate when connecting to it. The end result of this step was seeing LOTS of text.

The initial attempt to solder a lead to the RX pad failed, a fact I didn't fully grasp at first. Once I figured out that was the issue, further research led to how to interrupt the boot sequence. U-boot's `bootdelay` is set to zero by default, but there is still a way to interrupt the boot sequence. I've seen blog and forum posts suggesting that pressing Esc or Enter will do it. Esc didn't do it in this case, but Enter did. The easiest way to do it is to connect to it using screen or something similar: 

`screen /dev/tty.usbserial-A50285BI 115200`

While still in the terminal window, press and hold Enter. This results in root access. From there, the `help` command reveals a long list of available commands. I had to increase the size of screen's scrollback buffer on my machine just to be able to scroll to see them all. 

 A selection of some of the commands that I've used or that I'd like to further explore:
 
 ```
 env2flash - read environment parameter file and restore it to flash
envbin - read out environment parameter and store it to usb disk
[...]
fatinfo - print information about filesystem
fatload - load binary file from a dos filesystem
 fatwrite - write binary file to a dos filesystem
 flinfo  - print FLASH memory information
 [...]
 printenv- print environment variables
 [...]
 spi_gfo - SPI get flash info
 [...]
 usb     - USB sub-system
 [...]
 usbboot - boot from USB device
 ```
 Of further note: `help <command name> [sub command]` is there. It's more helpful for some things than for others.
 
 Boot Settings
 
 `bootcmd=usb exit;spi_rdc 0x80B00000 0x7001C 0x28C385; LzmaDec 0x80B00000 0x28C38
5 0x80000180 0x81000000; go 0x80000224;`

Translates to: exit `usb`, read length of `0x28C385` from SPI flash address `0x7001C` and write to DRAM address `0x80B00000`, decompress the kernel, go to DRAM address `0x80000224` to start application.

More Findings From Poking Around

As mentioned above, `envbin` copies what `help` calls the environment parameter file from flash to persistent storage connected via USB. Looking at the printout from running it, it appears to execute `spi_rdc` to get the data from flash and save it to a buffer, then use `fatwrite` to send the contents to a file named `env.bin` on USB. The file is always named `env.bin`, which is important.

To restore from a backup file on USB to flash, use `env2flash`. The file must be named `env.bin`. This appears to work similarly to `envbin` in that it's a wrapper for the commands to read from USB and rewrite to flash.
