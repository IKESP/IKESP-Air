# IKESP-AQ
An upgrade to the IKEA Vindriktning PM2.5 sensor to include eCO<sub>2</sub>, tVOC, relative humidity, temperature and barometric pressure sensing, and to report it all back to Home Assistant using an ESP8266.

## About
After using a few different ways to monitor air quality in my home, I was looking for something that could monitor CO<sub>2</sub>, VOC and PM2.5 levels. Getting an accurate measurement of humidity and temperature was also important to ensure accuracy of the other sensors, and to satisfy my own curiosity. Being familiar with many of the off the shelf solutions thanks to my day job, I struggled to find something that performed as I wanted, at a reasonably low price, and in an unobtrusive and visually appealing package. I was aware of the IKEA Vindriktning particle counters, and was please with the at a glance representation of the pm2.5 levels in a given room, and in the small size and low price it presents. What I was unhappy with was the lack of any other sensors. The newer Vindstyrka units provide temperature and VOC levels, but have no way of measuring the other key variables mentioned above. I was also not the biggest fan of their onboard display, or the requirement to add either an IKEA hub and their app to monitor the measurements remotely or to add a zigbee stick for communication with Home Assistant. Enter the IKESP-AQ project. Opening a Vindriktning unit, I was pleased to see that not only was the particle counter was the expected Cubic PM1006, which is [well supported](https://esphome.io/components/sensor/pm1006.html) on Home Assistant, but it also appears like whoever designed it has modification in mind. There are test pads available on the board for the UART of the sensor to which we can easily solder an ESP8266, giving us WiFi and an easy way to connect our device, as well as a way to add the other sensors required to meet my needs.

## Hardware
Next thing to do was decide on which sensors to use. I figured the first place to start was finding a temperature and humidity probe. There are many excellent choices for this category, I was initially interested in using the DHT22 (I discounted the DHT11 due to its poor performance at below freezing temperatures, as I plan to use this for some outdoor sensors too) but instead decided to use the BME280 (not the BMP280 which lacks humidity sensing) thanks to its included pressure sensor. Its use of the I<sup>2</sup>C protocol will also make it easier to package into the small space of the IKEA enclosure. That ticks off most of the missing values, leaving just VOC and CO<sub>2</sub>, preferably using the I<sup>2</sup>C protocol. The obvious choice is the CCS811. While it lacks a dedicated CO<sub>2</sub> measurement, instead giving an equivalent CO<sub>2</sub> that is calculated from the tVOC reading given by the MOX sensor. While this isn't ideal, it is more than accurate enough for our purposes. We can improve the accuracy of this value by giving it an external temperature reading, and by performing a simple calibration as detailed further below. In future I will probably look at comparing the BME280 and CCS811 to the BME680, however in my initial testing the 680 seemed to have a much harder time accounting for shifts in temperature, with rapid temperature changes leading to poor accuracy in CO<sub>2</sub> measurements.

Thankfully the decision of what board to base this on is easy. Having previously used the ESP8266 based Wemos/Lolin D1 Mini, I had one in my desk draw as part of another project. I was able to test-fit it into the enclosure where it fits with one small modification, which isn't visible from the outside and can be done in seconds with a pair of side cutters. It easily provides the required inputs for the I<sup>2</sup>C and UART interfaces of the sensors, the 3.3v for the CCS811 and BME280 boards, and can be easily powered by the 5v rail of the Vindriktning board, meaning no need to add extra power connectors or do more modification to the case or unit.

## Software
Software is simple. [ESPHome](https://esphome.io/) allows us to easily expose the sensors to Home Assistant, and to manage the D1 Mini wirelessly (after initial setup) meaning there's no need to open the enclosure to update the config.

## Flashing the ESP8266
Flashing the controller is easy. To start, we need a .bin file from ESPHome. Log into the web interface, add a new device and name it. Select ESP8266, allow it to create an encryption key for the device, then click "cancel" when asked how you want to install the yaml on your device. Instead, select "Edit" and copy the api and ota sections, which should look similar to below, into a text editor for later.

```
api:
  encryption:
    key: "XXXXXXX"

ota:
  - platform: esphome
    password: "XXXXXXX"
```

Return to the ESPHome web interface, delete the contents of the yaml file, and copy the contents of [example.yaml](example.yaml) into it. Copy the api and ota section from your text editor into the yaml file to replace the incomplete parts. Next, we need to provide your WiFi SSID and password. If you're using secrets, ensure that they match the !secret names in the yaml. If you don't wish to use secrets, you can enter the SSID and password as plain text, though this is not advised. If you wish to change the password of the fallback AP, which will automatically broadcast itself if the device fails to connect to WiFi within 60s of booting to allow you to reconnect the device, you can do that here either by replacing the plain text password, or by using a secret to set a new password.

Next, we can change the name of the device in the substitutions section. This should usually allow you to easily identify which this device from any others in your home, and make it much easier to identify the sensor outputs in home assistant when multiple devices are in use.

After modifying the file, we need to write it to our micro-controller. The first time we do this on each device it has to be connected by USB, though in future we should be able to update the device wirelessly. First, we can download the required bin file from ESPHome by selecting `Install > Manual Download` and when prompted, select a location to save the file. We then need to use a tool to flash the controller. You can use any tool you like to do this. Out of ease, I use ESPTool on Ubuntu, thanks to its easy install via apt. IF you need help with this step, you can refer to the [guide from the Espressif docs.](https://docs.espressif.com/projects/esptool/en/latest/esp8266/esptool/flashing-firmware.html)

## Preparing the Hardware
Wiring is fairly simple. The first step is to create what we'll refer to as the I<sup>2</sup>C bundle. We can stack the BME280 on top of the CCS811 using a double height pin header. If you didn't receive one with your parts, you may have what you need anyway, by taking an ordinary pin header and snapping it into 2 4-pin long sections. Remove the pins from one section, and push the plastic onto the long side of the other 4 pin section. Solder it to the sensor boards, connecting the VIN/VCC, GND, SCL, and SDA pins of the 2 sensors to create a stack. We then need to solder a small wire to bridge the WAK pin on the CCS811 to ground. WE then solder 4 wires, each about 1.5-2" long, to the 4 pins.

Next, we can prepare the D1 mini. WE need to add wires for the PM1006 sensor, and for 5V and ground connections to the IKEA board. The 5V and ground wires should be done to the matching pads on the D1 Mini, as well as wires on D4 (for the fan connection on the IKEA board) and D5 (for the TX pin of the PM1006). Once these wires hve been soldered, we can attach the IKEA board. The 5V and ground pins can be attached to their corresponding pads at the bottom of the IKEA board, beside the connector for the PM1006. The TX wire can be added to the REST pad on the same row. The final wire is for detecting when the fan comes on, which corresponds to when the sensor is taking a reading. This can be useful, as we are not requesting the unit to take readings, just listening to the results when it reports the results. This should be attached to the FAN+ pad on the row of 5 pads at the centre of the board.

For quick reference, please see the below table. While it doesn't matter too much which of the digital pins on the D1 we use, for the provided code to match we should follow the below table:

|D1 Mini | Connection|
|---|---|
|5V | IKEA 5V|
|GND | IKEA GND|
|D4 | IKEA Fan|
|D5 | PM1006 TX|
|3.3V | I<sup>2</sup>C VIN |
|GND | I<sup>2</sup>C VGND |
|D1 | I<sup>2</sup>C SCL |
|D2 | I<sup>2</sup>C SDA |

## Connecting to Home Assistant
Connecting to Home Assistant should also be painless. Ideally, once flashed with ESPHome and online it should appear in your integrations screen, asking you if you want to configure a new integration. If not, you can manually add the device but selecting Add Integration, searching for ESPHome, and adding the device using its IP address and port (default should be 6053). For more help, see the [guide on esphome.io](https://esphome.io/guides/getting_started_hassio.html).
