This is an ATSC TV tuner box that follows a very common pattern in this space. On the outside, it has a minimal physical interface limited to an LCD indicating the current channel, three buttons to power on/off and change the channel, and one USB port for recording/watching media and firmware updates. This particular box, sold by Aluratek, also has a couple of games on it. I'm especially interested in modding them.

Internally, it's much more interesting. The processor is a SOC manufactured by MStar, a Mediatek subsidiary. The part number appears to be (from the u-boot printout) MST173B or MST173BD01A. The demodulator is also made by MStar; its part number is MSB1237. It has 32MB of DDR II SDRAM ([ESMT part number M14D5121632A](https://www.endrich.com/fm/2/M14D5121632A-2.5BG2K.pdf))  The tuner is under metal shielding, which I left intact since I didn't want to damage it. At the front of the board are three circuit pads marked GND, RX, TX. Since I lacked the proper tools to verify this ahead of time, on a hunch, I soldered a lead to the easiest pad (TX) and used a USB-to-TTL board set to 3.3V to see if anything would be printed out. Based on prior research of other MStar-based devices, it seemed reasonable to use a 115200 baud rate when connecting to it. The end result of this step was seeing LOTS of text.

The initial attempt to solder a lead to the RX pad failed, a fact I didn't fully grasp at first. Once I figured out that was the issue, further research led to how to interrupt the boot sequence. U-boot's `bootdelay` is set to zero by default, but there is still a way to interrupt the boot sequence. I've seen blog and forum posts suggesting that pressing Esc or Enter will do it for this particular board. Esc didn't do it in this case, but Enter did. The easiest way to do it is to connect to it using screen or something similar: 

`screen /dev/tty.usbserial-A50285BI 115200`

While still in the terminal window, press and hold Enter. This results in root access. 

```
BOOTSPILMLMLMLM76A6_OK!decomp
_done

Hello U-Boot


U-Boot 1.1.6 (May  8 2018 - 14:23:15)

Board: MSTAR KAPPA (CPU Speed 600 MHz)
DRAM:  64 X 0 MBytes
U-Boot is running at DRAM 0x87610000
Module: USB FAT FLASH SPI LOGO OSD ENV=SERIAL
Flash is detected (0x09A0, 0x5E, 0x40, 0x16)
In:    serial
Out:   serial
Err:   serial
MSVC00B000100100208768TH0000000T
[MDrv_PNL_Init][524]
MDrv_PNL_Init u32PnlRiuBaseAddr = BF200000
[MDrv_PNL_Init][535]
MDrv_PNL_Init u32PMRiuBaseAddr = BF000000
[MDrv_PNL_Init][545]
[MApi_PNL_Init][1118]
[_SetPanelParams][469]
[MDrv_PNL_SetPanelType][553]
[MDrv_PNL_SetPanelType][575]
[_MDrv_PNL_Init_LPLL][295]pstPanelInitData->u16Width=720, pstPanelInitData->u16Height=480
[_MDrv_PNL_Init_LPLL][297]u16HTotal=857,u16VTotal=524,pstPanelInitData->u16HTotal=857,pstPanelInitData->u16VTotal=524, u16DefaultVFreq=600
[_MDrv_PNL_Init_Output_Dclk][340]pstPanelInitData->u16Width=720, pstPanelInitData->u16Height=480
[_MDrv_PNL_Init_Output_Dclk][342]u16HTotal=857,u16VTotal=524,pstPanelInitData->u16HTotal=857,pstPanelInitData->u16VTotal=524, u16DefaultVFreq=600
[XC,Version] 00442327
 UTOPIA ASSERT: 0 /home/stb/PERFORCE/THEALE/utopia/project/kappa_nos_mips_mboot/../../mxlib/hal/kappa/xc/mhal_sc.c Hal_SC_set_ficlk 224
DAC eTiming =0
HDMITx eTiming =1
HDMITx eTiming =1
[Hal_VE_EnableDI][1049] bEnable = 0, bIsDNR2VE = 0
u32ReadBuffVirAddr = A0000000, u32IntBuffVirAddr = A0100000, u32OutBuffVirAddr = A0730000
verJPD_SetStatus >>>>>>>>>>> w:720,  h:576,  p:720

[GOP3, PID 0, TID 0x-1][Driver Version]: 0089, BuildNum: 0002, ChangeList: 00524916
 keypad_pressed is [0]
 ir_pressed is [0]

@@@[main_loop][3117]@@@
Hit any key to stop autoboot:  0

<< MStar >>#
```

From there, the `help` command reveals a long list of available commands. I had to increase the size of screen's scrollback buffer on my machine just to be able to scroll to see them all. 

 ## A selection of some of the commands that I've used or that I'd like to further explore
 
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
 
 ## Boot Settings
 
 `bootcmd=usb exit;spi_rdc 0x80B00000 0x7001C 0x28C385; LzmaDec 0x80B00000 0x28C38
5 0x80000180 0x81000000; go 0x80000224;`

Translates to: exit `usb`, read length of `0x28C385` from SPI flash address `0x7001C` and write to DRAM address `0x80B00000`, decompress the kernel, go to DRAM address `0x80000224` to start application.

## Environment Settings
```
<< MStar >># printenv
bootdelay=0
baudrate=115200
preboot=echo;echo Type "help" for more commands.
MS_BOARD=BD_MST173B_D01A
logo_cmd=boot_logo 0 0 1 1
bootcmd=usb exit;spi_rdc 0x80B00000 0x7001C 0x28C385; LzmaDec 0x80B00000 0x28C385 0x80000180 0x81000000; go 0x80000224;
info_exchange=spi
CUSTOMER_OUI=0x169B
AP_SW_MODEL=0x0001
AP_SW_VERSION=0x0001
HW_MODEL=0x0001
HW_VERSION=0x0001
panel_cmd=set_paneltype 6
RF_No=1
RF_MODULATION_IS_CH3=1
stdin=serial
stdout=serial
stderr=serial

Environment size: 459/65532 bytes
```

## More Findings From Poking Around

As mentioned above, `envbin` copies what `help` calls the environment parameter file from flash to persistent storage connected via USB. Looking at the printout from running it, it appears to execute `spi_rdc` to get the data from flash and save it to a buffer, then use `fatwrite` to send the contents to a file named `env.bin` on USB. The file is always named `env.bin`, which is important. 

```
<< MStar >># envbin
(Re)start USB...
Check USB port[0]:
Host type:2
scanning bus for devices... 1 USB Device(s) found
       scanning bus for storage devices... bulk max packet size: ep 200 ep2 200
usb_stor_Bulk_max_lun: 0
1 Storage Device(s) found
buff:spi_rdc 0x80000000 0x3E0000 0x10000
offset 0x3E0000, size 0x10000
Start read 10000 data from serial device...
crc:DE697461,crcNew:DE697461
buff:fatwrite usb 0 0x80000000 env.bin 0x10000
file env.bin found
gfd[0].ent_offset 7 7 ent_cluser:2file exist, and will be recreate
fd:0, cur_cluster:2, cursect:29970, offset:7
start_cluster= 5
entry = 5
offset = 5 bufnum = 0
entry = 6
offset = 6 bufnum = 0
entry = 7
offset = 7 bufnum = 0
entry = 8
offset = 8 bufnum = 0
entry = 9
offset = 9 bufnum = 0
entry = 10
offset = 10 bufnum = 0
entry = 11
offset = 11 bufnum = 0
entry = 15
offset = 15 bufnum = 0
entry = 16
offset = 16 bufnum = 0
entry = 17
offset = 17 bufnum = 0
entry = 18
offset = 18 bufnum = 0
entry = 19
offset = 19 bufnum = 0
entry = 20
offset = 20 bufnum = 0
entry = 21
offset = 21 bufnum = 0
entry = 22
offset = 22 bufnum = 0
entry = 23
offset = 23 bufnum = 0
fd:0, cur_cluster:2, cursect:29970, offset:7
#set getno = 16 and clust_size = 8
##
ok
```

To restore from a backup file on USB to flash, use `env2flash`. The file must be named `env.bin`. This appears to work similarly to `envbin` in that it's a wrapper for the commands to read from USB and rewrite to flash.

### binwalk

Using `binwalk` with default settings yielded nothing despite many signatures being found. In fact, the only thing that did reveal anything of worth was using `-I`, which includes results deemed invalid by other scans.

```
chris$ binwalk -I ENV.BIN

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
16516         0x4084          VxWorks symbol table, little endian, first entry: [type: function, code address: 0xF002E00, symbol address: 0x6E006F00],,,,,,,,,
17721         0x4539          EFS2 Qualcomm filesystem super block, little endian, , NOR version 0x7400, 5206351 blocks, 0x205249 pages per block, 0x719b2dc4 bytes per page
17849         0x45B9          EFS2 Qualcomm filesystem super block, little endian, , NOR version 0x7400, 5206351 blocks, 0x204148 pages per block, 0x719b2e3f bytes per page
19643         0x4CBB          Intel x86 or x64 microcode, sig 0x006e0072, pf_mask 0x6c002e00, 006F-00-75, rev 0x6a4100, size 1694529024
19707         0x4CFB          Intel x86 or x64 microcode, sig 0x00740073, pf_mask 0xffffff00, 0073-00-79, rev 0x5f4200, size 4294967040
19803         0x4D5B          Intel x86 or x64 microcode, sig 0x00720065, pf_mask 0xffffffff, 0075-00-73, rev 0x5f4200, size 4294967040
19899         0x4DBB          Intel x86 or x64 microcode, sig 0x006f0069, pf_mask 0x7900, 0070-00-72, rev 0x5f4200, size 4294967040
19995         0x4E1B          Intel x86 or x64 microcode, sig 0xffffffff, pf_mask 0xffffffff, 0065-00-00, rev 0x724300, size 4294967040
20123         0x4E9B          Intel x86 or x64 microcode, sig 0xffff0000, pf_mask 0xffffffff, 0073-00-74, rev 0x6f4300, size 4294967040
20251         0x4F1B          Intel x86 or x64 microcode, sig 0x00680063, pf_mask 0xff000000, 0068-00-5F, rev 0x744200, size 4294967040
20347         0x4F7B          Intel x86 or x64 microcode, sig 0x006e006f, pf_mask 0xffffffff, 0074-00-69, rev 0x614200, size 4294967040
20443         0x4FDB          Intel x86 or x64 microcode, sig 0x00650063, pf_mask 0xffffffff, 0061-00-6E, rev 0x684300, size 4294967040
20571         0x505B          Intel x86 or x64 microcode, sig 0xffff0000, pf_mask 0xffffffff, 006F-00-6E, rev 0x694200, size 4294967040
20731         0x50FB          Intel x86 or x64 microcode, sig 0x00640074, pf_mask 0x74005f00, 0068-00-75, rev 0x734100, size 1694526720
21625         0x5479          EFS2 Qualcomm filesystem super block, little endian, , NOR version 0x7400, 5206351 blocks, 0x204148 pages per block, 0x719b2a7d bytes per page
22652         0x587C          Intel x86 or x64 microcode, sig 0x0f006500, pf_mask 0x610064, 6F00-72-00, rev 0x740073e5, size 7536741
```
