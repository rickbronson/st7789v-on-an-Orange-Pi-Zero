  st7789v on an Orange Pi Zero
==========================================


Here we will get working a st7789v TFT 240x320 LCD on an Orange Pi Zero

TFT 240x320 LCD:

https://www.aliexpress.us/item/3256805435465934.html?spm=a2g0o.store_pc_aeChoiceSimplified.0.0.174e3ac1xPZ1Lt&pdp_npi=4%40dis%21USD%21US%20%246.13%21US%20%243.98%21%21%2143.88%2128.53%21%402101f6bf17103590116682709d0f68%2112000033780517242%21sh%21US%21108352091%21&gatewayAdapt=glo2usa

  It seems it's hard to find out who actually makes this thing, the closest thing I could find was:

http://hiletgo.com/ProductDetail/2157216.html

Orange Pi Zero:

http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/details/Orange-Pi-Zero-LTS.html


Here is hookup photo showing what you need to connect:

![alt text](https://github.com/rickbronson/st7789v-on-an-Orange-Pi-Zero/blob/master/docs/hardware/orangepi-only-hookup3.png "hookup")

I installed this verson of Armbian on the Orange Pi Zero:

Armbian_24.2.1_Orangepizero_jammy_current_6.6.16_minimal.img.xz

Steps for install:

 - On your Linux box do:

```
sudo apt-get install minicom pv xz-utils
```

- Insert SD card into your Linux box (change sdX for your SD card below):
 
```
file=Armbian_24.2.1_Orangepizero_jammy_current_6.6.16_minimal.img.xz
pv $file | xzcat | sudo dd of=/dev/sdX bs=1M
sync
```

 - Remove and re-insert the SD card, it should auto mount the /boot dir, update the armbianEnv.txt (make sure /media/$USER/armbi_root/boot exists):

```
ls -al /media/$USER/armbi_root/boot/ # make sure this exists before going any further
sudo cp /media/$USER/armbi_root/boot/armbianEnv.txt /media/$USER/armbi_root/boot/armbianEnv.txt.sav
sudo bash -c "cat << EOF > /media/$USER/armbi_root/boot/armbianEnv.txt
verbosity=7
bootlogo=false
console=both
disp_mode=1920x1080p60
overlay_prefix=sun8i-h3
overlays=spi-spidev spi-add-cs1 usbhost2 usbhost3 i2c1 uart2
param_spidev_spi_bus=1
param_spidev_spi_cs=0
extraargs="fbcon=map:8"
rootdev=UUID=abe40633-309f-4e11-8664-afcb88cccd6c
rootfstype=ext4
user_overlays=st7789v
usbstoragequirks=0x2537:0x1066:u,0x2537:0x1068:u
EOF"
umount /media/$USER/armbi_root
```

 - Hook up the USB-Serial adapter as shown in the hookup, insert the SD card into the Zero and do:
```
minicom -b 115200 -w -D /dev/ttyUSB0 -C ~/minicom-orangepi-zero.cap
```

 - Log into and setup up the Zero, then do this so you can sudo without password (if you want):

```
USR=`id -u -n`
su - -c "sed -i -e \"\\\$a${USR}     ALL = NOPASSWD: ALL\" /etc/sudoers"
```

 - Add some modules that you will need:

```
sudo bash -c 'cat << EOF >> /etc/modules-load.d/modules.conf
g_serial
fbtft
fb_st7789v
EOF'
```

 - Get some packages that you will need:

```
sudo apt update
sudo apt install fbi ssh
```

 - Create the overlay and add it in:

```
sudo bash -c 'cat << EOF > /boot/overlay-user/st7789v.dts
/dts-v1/;
/plugin/;

/ {
  compatible = "allwinner,sun8i-h3";

  fragment@0 {
    target = <&spi1>;
    __overlay__ {
      #address-cells = <1>;
      #size-cells = <0>;
      pinctrl-names = "default", "default";

      spidev@1 {
        compatible = "sitronix,st7789v";
        reg = <0>; // NOTE SPI chip select number
        pinctrl-1 = <&st7789v_pins>;
        spi-max-frequency = <32000000>; /* NOTE: the value in /boot/armbianEnv.txt overrides this */
				spi-cpol;
								custom = <1>;
								bgr;
				/* 				bgr; tried rgb bgr grb */
        spi-cpha;
				width = <240>;
        height = <320>;
				rotate = <90>; // NOTE 90/270 for horizontal orientation
        fps = <25>;
        buswidth = <8>;
				txbuflen = <16384>; // NOTE supposed to speed up overall transfer and make SPI device more available for other uses
        // NOTE obviously your pins will vary but last 0 or GPIO_ACTIVE_HIGH is important
        reset-gpios = <&pio 0 11 1>;  /* active LOW */
        dc-gpios = <&pio 0 6 0>;
        led-gpios = <&pio 0 1 0>;
        debug = <0>; // NOTE high value like 9 will print lots of debug info to dmesg
				};
    };
  };

  fragment@1 {
    target = <&pio>;
    __overlay__ {

      st7789v_pins: st7789v_pins {
        pins = "PA1", "PA6", "PA11";
        function = "gpio_out", "gpio_out", "gpio_out";
      };
    };
  };

};
EOF'
sudo armbian-add-overlay /boot/overlay-user/st7789v.dts
```

 - Then reboot

```
reboot
```

 - Try

```
dmesg | egrep 'st7789|tft|spi'
```

 - You should see something like:

```
[   12.296784] fbtft: module is from the staging directory, the quality is unknown, you have been warned.
[   12.312719] fb_st7789v: module is from the staging directory, the quality is unknown, you have been warned.
[   12.331914] SPI driver fb_st7789v has no spi_device_id for sitronix,st7789v
[   12.339254] fb_st7789v spi1.0: fbtft_property_value: width = 240
[   12.345420] fb_st7789v spi1.0: fbtft_property_value: height = 320
[   12.357009] fb_st7789v spi1.0: fbtft_property_value: buswidth = 8
[   12.363192] fb_st7789v spi1.0: fbtft_property_value: debug = 0
[   12.369146] fb_st7789v spi1.0: fbtft_property_value: rotate = 90
[   12.369174] fb_st7789v spi1.0: fbtft_property_value: fps = 25
[   12.369192] fb_st7789v spi1.0: fbtft_property_value: txbuflen = 16384
[   12.849337] graphics fb0: fb_st7789v frame buffer, 320x240, 150 KiB video memory, 16 KiB buffer memory, fps=25, spi1.0 at 32 MHz
[   14.541388] systemd[1]: Starting Load/Save Screen Backlight Brightness of backlight:fb_st7789v...
[   14.643085] systemd[1]: Finished Load/Save Screen Backlight Brightness of backlight:fb_st7789v.
```

  It seems that "SPI driver fb_st7789v has no spi_device_id for sitronix,st7789v" is just a warning and not an error.
	
  -Try

```
lsmod | egrep "fb|g_serial"
```

 - You should see something like:

```
fb_st7789v             12288  0
fbtft                  36864  1 fb_st7789v
g_serial               12288  0
libcomposite           49152  2 g_serial,usb_f_acm
```

  -Try

```
cat /proc/device-tree/soc/spi\@1c69000/status
```

 - You should see:

```
okay
```
  -Try

```
ls -l /sys/bus/spi/devices
```

 - You should see:

```
lrwxrwxrwx 1 root root 0 Dec 31  1969 spi1.0 -> ../../../devices/platform/soc/1c69000.spi/spi_master/spi1/spi1.0
```

  -Try

```
cat /sys/devices/platform/soc/1c69000.spi/spi_master/spi1/spi1.0/backlight/fb_st7789v/bl_power
```

 - This should show current backlight power on LCD

  -Try this to turn backlight on:

```
sudo bash -c 'echo 0 > /sys/devices/platform/soc/1c69000.spi/spi_master/spi1/spi1.0/backlight/fb_st7789v/bl_power'
```

  If the backlight didn't come on, check the wiring.

  -Try this to put random dots on the screen:

```
sudo head -300 /dev/urandom > /dev/fb0
```

  - Everytime you run the above, you should see the number of interrupts increase:

```
cat /proc/interrupts | grep spi
```

  If you don't see anything on the screen, check the wiring.
	
  -Try this to put an image on the screen:

```
sudo fbi -d /dev/fb0 -T 1 orangepi-hookup-340x240.png
```

  The colors will look strange, haven't quite figured that out (might have something to do with "#define HSD20_IPS" in drivers/staging/fbtft/fb_st7789v.c)
	
 - Here is a picture of the hardware:

![alt text](https://github.com/rickbronson/st7789v-on-an-Orange-Pi-Zero/blob/master/docs/hardware/IMG_20240313_152858580_HDR.jpg "hookup")
