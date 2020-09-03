# unring

*fuck the police!*

# Journal

## 08312020

Experimented with some pre-compiled Micropython builds that include a driver for the OV2640 camera, but didn't have any luck getting it to initialize.  I think the board I'm using has the camera connected differently from the dev boards the firmware developers are using.

## 09012020

Taking a more organized stab at this, starting by picking one Micropython build to work with.  The selected firmware is [lemariva](https://github.com/lemariva/micropython-camera-driver)'s Micropython build described in [this blog post](https://lemariva.com/blog/2020/08/micropython-ov2640-camera-module-extended).  At some point I'll do a [custom Micropython build](https://lemariva.com/blog/2020/06/micropython-support-cameras-m5camera-esp32-cam-etc) for this board, but if I can get started without having to learn how to do that, it will be helpful.


So I've loaded the pre-compiled Micropython and now I'm going to try and get the recommended [uPyCam](https://github.com/lemariva/uPyCam) script running.

Using `rshell -p /dev/ttyUSB1` to manage the board now.

Trying a custom camera init:

camera.init(0, d0=39, d1=36, d2=23, d3=18, d4=15, d5=5, vsync=27, href=25, pclk=19, pwdn=26, xclk=32, siod=13, sioc=12, reset=-1)

## 09022020

After much experimentation I got the camera initialized corredtly:

```
           camera.init(
                0,
                xclk_freq=camera.XCLK_10MHz,
                d0=5,
                d1=14,
                d2=4,
                d3=15,
                d4=18,
                d5=23,
                d6=36,
                d7=39,
                vsync=27,
                href=25,
                pclk=19,
                pwdn=26,
                xclk=32,
                siod=13,
                sioc=12,
                reset=-1
            )
```

It appears to run fine off battery power as well, although there's no way to tell what the battery level is.  I think the next steps will be to figure that out, as well as the rest of the onboard hardware and publish that by extending the web API this camera demo uses.

### Other Hardware

* Charging chip: IP5306 I2C
* Display: SSD1306 I2C
    + SDA - 21
    + SCL - 33
* Infrared: AS312
    + pin 33
* Pushbutton (right) on pin 34 (other button tied to reset)

The display and charging chip are on the I2C bus.  Time to learn how to talk to that in Micropython...

No luck finding any devices using the .scan() method.


## 09032020

Did a little reading on I2C and tried scanning the bus again.  This time it came back with one device: `60` (3C hex).  

```

>>> from machine import Pin, I2C
>>> i2c = I2C(0, scl=Pin(22), sda=Pin(21), freq=400000)
>>> i2c.scan()
[60]

```


I'm not sure if this is the display or the charge controller though, so we'll have to experiment more.

Not easy finding docs for this chip.  I found some sample code for reading the charge level in (what looks to me like) C for another device:

```

int8_t getBatteryLevel()
{
  Wire.beginTransmission(0x75);
  Wire.write(0x78);
  if (Wire.endTransmission(false) == 0
   && Wire.requestFrom(0x75, 1)) {
    switch (Wire.read() & 0xF0) {
    case 0xE0: return 25;
    case 0xC0: return 50;
    case 0x80: return 75;
    case 0x00: return 100;
    default: return 0;
    }
  }
  return -1;
}
```

from: [https://github.com/m5stack/M5Stack/issues/74]

I guess it could be wiring/Arduino?

I *think* this means I read register 0xF0 (240 decimal) to get the charge level?

Here's what my first attempt looks like:

```
>>address = 60
>>bat_reg = 240
>>data = i2c.readfrom_mem(address, bat_reg, 1)
>>print(data)
b'F'
```

This could be right?  If 'F' means "max", that could be right since the battery has been plugged-in this whole time.  I can't unplug it without loosing my serial connection to the REPL, so I'll try unplugging the battery itself and see if it changes.

With the battery unplugged the `data` value is the same, so the results are inconclusive.  I tried another method of querying the device (`readfrom()`) but it also returns `F`.

It's possible this isn't even the right device.  Maybe we can try talking to it like it's the display and rule that out?

After initial experimentation with some sample code, it's clear that this device is the display and not the charge controller.  I'm not sure if that means that the charge controller isn't connected to the I2C bus, or if it's just not showing up when I scan the bus.  In any event, now that I've started playing with the display I might as well try and do something useful with it.

I'm working with the library and sample code provided here:

https://www.makerfabs.cc/article/micropython-esp32-tutorial-interfacing-0-96-inch-oled.html

It doesn't quite work out-of-the-box but I figure that might be due to pin assumptions and such.

Appears that this module only works with "software I2C".  I'm not sure what the disadvantage of that might be (I assume hardware I2C is preferable) but for now we'll go with what works.

```
>>> from machine import Pin, I2C
>>> import ssd1306
>>> oled_width = 128
>>> oled_height = 64
>>> i2c = I2C(scl=Pin(22), sda=Pin(21), freq=400000)
>>> oled = ssd1306.SSD1306_I2C(oled_width, oled_height, i2c)
>>> oled.text('Hello, World!', 0, 0)
>>> oled.show()
```

As you'd expect this displays "Hello, World!" on the OLED.  It's upside-down, but maybe we can fix that with some tweaks to the module or the initialization.

Now that we know a little bit about using the OLED, let's try and have it display the IP address when the webserver boots up.

Modified the `boot.py` script to display the IP address when WiFi connects, or display an error when it can't.  I'm not sure how much more time I want to spend on the display at this point so I'm switching-back to look at the camera.

For some reason the camera web UI stopped working.  I ended up deleting the `main.py` file, manually running the python that's in it and then the camera worked again.  I copied a fresh copy of main.py over and now it's happy, so I don't know what was going on there...





# References

* https://github.com/lemariva/uPyCam
* https://lemariva.com/blog/2020/06/micropython-support-cameras-m5camera-esp32-cam-etc
* https://lemariva.com/blog/2020/08/micropython-ov2640-camera-module-extended
* https://github.com/lemariva/esp32-camera
* https://github.com/lemariva/micropython-camera-driver
* https://github.com/tsaarni/esp32-micropython-webcam/blob/master/webcam.py
* https://github.com/shariltumin/esp32-cam-micropython/tree/master/esp32-cam-1-11-498
* https://github.com/Lennyz1988/micropython/releases/tag/v1
* https://github.com/tsaarni/esp32-micropython-webcam/blob/master/boot.py
* https://github.com/tsaarni/micropython-with-esp32-cam/wiki
* https://www.electronics-lab.com/ttgo-t-camera-esp32-cam-board-oled-ai-capabilities/
* https://github.com/LilyGO/ESP32-Camera
* 
