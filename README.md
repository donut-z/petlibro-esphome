# petlibro-esphome

> **Note on this fork:** This fork significantly enhances the firmware for the **Petlibro Air Smart Feeder (PLAF108)**. While the upstream repository provides basic pin mappings, this fork turns the feeder into a fully autonomous, production-ready smart feeder with scheduling, fail-safe motor unjamming, offline fallback routines, and flash-wear protection living directly on the ESP32 itself.

---

## What's changed in this fork (PLAF108)

### Quality-of-Life improvements & features

* **Autonomous On-Device Scheduling (8 Daily Feedings):**
* The entire feeding schedule runs directly on the ESP32 via on-device SNTP time, so no Home Assistant automations are required for daily feeding.
* 8 independently configurable feeding times (`HH:MM`) with portion counts (`0 = disabled`).
* Easily adjustable on the fly via the built-in ESPHome Web Server v3 UI or through Home Assistant entities (`time` and `number`).


* **Stall Detection & Auto-Unjamming Routine:**
* **Precision timing:** One normal dose takes ~3.5 s. The internal microswitch clicks once (~1.2 s) when the cam leaves the switch, and a second time (~3.5 s) when the next cam reaches it, marking the dose complete.
* **Stall detection:** If the first click does not register within 2.0 s, or the second click within 4.0 s, the motor is detected as stalled/jammed.
* **Auto-unjam:** The motor immediately reverses for a configurable duration (default: 1200 ms, adjustable from 200 ms to 1500 ms via slider), pauses briefly to relieve gear tension, and then turns forward again to complete the dose.
* **Fault reporting:** If the jam persists after unjamming, a fault is raised (physical red alarm LED + `feeder_fault` problem sensor in Home Assistant), the dispensing loop safely halts to protect the motor and gearbox, and the feeder tries again at the next scheduled feeding.
* Includes a dedicated manual "Unjam Motor" trigger button in HA and the Web UI.


* **Offline Fallback Schedule (WiFi / Internet Outage Protection):**
* If the feeder boots without WiFi or cannot reach an NTP time server, an automated offline emergency schedule kicks in: it feeds at configurable intervals (default: 4 portions every 6 hours, ~56 g/day at 3.5 g/portion, or automatically calculated from the planned daily total).
* WiFi and API reboot timeouts are disabled (`reboot_timeout: 0s`), meaning once the clock syncs, normal scheduled feedings continue uninterrupted even during prolonged internet or Home Assistant outages.


* **Daily Safety Cap & Overfeed Protection:**
* Configurable daily portion limit (`max_daily_portions`, default 18).
* Acts as a child/glitch safety net: scheduled feedings will never exceed this cap in a single calendar day, even if feeding times accidentally overlap.


* **Grams Calibration & Accurate Tracking:**
* Configurable calibration slider (`gram_per_portion`, default: 3.5 g).
* Live tracking of both portions and grams fed today, as well as planned daily totals.
* Formatted as clean text sensors in the Web Server UI.
* Timestamp and portion count of the last successful feeding ("Last Feeding: 18:30:00 (2 portions)").


* **Hopper Level & Power Monitoring:**
* Optical food sensor tracking with debounce filters (`Food Hopper Status`: "Enough kibble in stock" / "Attention: hopper almost empty / refill!").
* Power source monitoring (`Power Source`: Mains Adapter vs Backup Batteries) and battery voltage/percentage estimation.


* **Streamlined ESPHome Web Server v3:**
* Clean, organized UI divided into three clear cards: *Status & Manual Control*, *Feeding Schedule (8 Times)*, and *Settings & Emergency Protection*.
* All raw hardware test pins hidden (`internal: true`) for a clean, non-cluttered user experience.


* **Minimal Flash Wear (RAM-First Architecture):**
* All high-frequency operational states (in-progress flags, motor timings, sensor deduplication, active timeouts, unjam states) reside strictly in RAM.
* Only persistent configuration (schedule times/portions, calibration settings, and a daily-resetting portions counter) are written to flash NVS.
---

# petlibro-esphome

A collection of alternative DIY open-source [ESPHome firmware](https://esphome.io) for [Petlibro](https://petlibro.com) series of smart cat/dog food feeders and water fountains devices.

Petlibro makes WiFi-connected pet food feeders and water fountains based on ESP8266/ESP32 microcontrollers from Espressif. Their default firmware requires that you use their Petlibro mobile application to create an account and it must via that be connected to their cloud (but use MQTT to communicate internally). 

This ESPHome firmware implements local LAN support with almost feature-parity as stock firmware without need for cloud/WAN connection.

# Factory firmware backup

Before flashing the firmware for either device, it is recommended to save a backup of the flash memory. The firmware installed from the factory includes data unique to your device, such as the internal ID and cryptographic keys used to communicate with the cloud server, so if you wish to be able to return your device to a state where it can communicate with the Petlibro cloud then you'll need to have made a backup.

This can be done by following the instructions for setting up the serial connection for your device in developer mode as per the instructions below, and then running the following `esptool` command:

```bash
esptool.py --before=default-reset --no-stub read-flash 0x0 ALL petlibro_factory_dump.bin
```

You are, of course, free to change the name of the output file as desired.

- The `--before=default-reset` and `--no-stub` flags may not be required, but do seem to make the process more reliable.
- If the read fails, try power cycling the device. In testing, it sometimes would fail to read the file part of the way through unless the device had been cleanly restarted immediately before the dump was initiated.

# Dockstream Smart Fountain (PLWF105)

An esphome firmware for PLWF105 automatic water bowl (also called the Dockstream Smart Fountain or the PETLIBRO App Monitoring Cat Water Fountain with Wireless Pump) by Petlibro. 

PLWF105 is available in either white-colored variant or black-colored variant of the same model:

* https://petlibro.com/products/dockstream-app-monitoring-water-fountain
  * https://www.amazon.com/dp/B0BSFB2D37

## About this ESPHome firmware for plwf105

This ESPHome firmware for plwf105 implements (almost) feature parity as the stock firmware but supports only local connections (instead of requiring cloud connection).

### Supported Features

- Turn pump on or off
- Red LED
- Yellow LED
- Scale (used to determine water level)
- Pump error indicator
- water consumption tracking
- water reporting in oz and ml
- configurable pump intervals
- integrated calibration process

## Flashing prereqs

If you carefully open the water bowl "base" you will see a PCB holding a `ESP32-C3-WROOM-03`, a `HX7111` and a voltage regulator. There is also one more chip on the board but I am unable to identify it.
![PCB](https://github.com/user-attachments/assets/cf67e89f-4cc1-4773-8e06-78d1bb700e36)

The PCB very conveniently contains 3 pins that we can use to flash our firmware. `GND`, `TX`, and `RX`.
The PC has a `VCC` pin that can be used to power to program, but your USB-Serial converter needs to be able to supply enough current. I suggest using the USB plug to power while programing.
The PCB also has two pads on the lower right corner labled `B` and `G`. These pads can be shorted to put the ESP chip in programmer mode.

## Flashing

0) You will need to generate a firmware .bin file from ESPHome based on the .yaml file in this repo.  

1) Plug the USB to serial converter into your computer and hook up to the pin holes as described below. Do NOT power the board/fountain yet.


`GND` -> `GND`

`TX`  -> `RX`

`RX`  -> `TX`

You don't even need to solder anything most likely. I was able to hold the pins in place with just tension.

2) Take a small wire and hold it on the `B` and `G` pads described above.
3) While still holding the wire in place, plug in the pumps USB cable.
4) At this point the board should be in programming mode.  If you left the top panel connected, the light should be a steady white.  If it's slowly flashing white you did not enter programming mode.
5) Flash the firmware you created above.  You can use use https://web.esphome.io/ to do the flashing.

If you run into problems or otherwise need to revert, you can reflash back to stock firmware using the .bin file in this repo.

## Result

Here is how it looks in HomeAssistant:

![Result](https://github.com/user-attachments/assets/24344f14-f331-4fa7-b6bd-9aeaddb32a11)

## Calibration

1) The water fountain will automatically enter calibration mode after a flash or after hitting "Start Calibration" in the home assistant UI
2) The fountain will now be in calibration mode (slow flashing yellow light).
3) With the water a the min fill line and fully assembled, hit the "wifi" button (or enter the number yourself in the home assistant UI)
4) The light should flash fast now, with the water a the max fill line and fully assembled, hit the "wifi" button (or enter the number yourself in the home assistant UI)


# Petlibro Air Smart Feeder (PLAF108 model)

An ESPHome firmware for PLAF108 automatic pet feeder (also called the Air Smart Feeder) by Petlibro.

PLAF108 is available in either white-colored variant or black-colored variant of the same model:

* https://petlibro.com/products/air-wifi-feeder
  * https://www.amazon.com/dp/B0CDC3WK46

## About this ESPHome firmware for PLAF108

This ESPHome firmware for PLAF108 implements (almost) feature parity as the stock firmware but supports only local connections (instead of requiring cloud connection).

### Supported Features

- Alarm Red LED
- Sensor for showing if feeder is on battery power
- Sensor for showing if battery is plugged in
- Sensor to enable the "Food Detection" sensor
- Sensor for showing if mains connected
- Switch to turn motor left
- Switch to turn motor right
- Sensor to show when the motor has done a quarter turn

### Unknowns

I do not know what GPIO0 and GPIO1 are for. I think it's for the DC-power current sensor to maybe determine battery charge. But, that's a guess.

## Flashing prereqs

The PCB very conveniently contains 4 pins that we can use to flash our firmware. `GND`, `TX`, `RX`, and `VCC`. The PCB also has two unlabled pin holes. These holes can be connected to each other put the ESP chip in programmer mode.

## Flashing

1) Plug the USB to serial converter into your computer and hook up to the pin holes as described below. Do NOT plugin the board yet.


`GND` -> `GND`

`TX`  -> `RX`

`RX`  -> `TX`

You don't even need to solder anything most likely. I was able to hold the pins in place with just tension.

2) Take a small wire and hold place it in the unmarked pin holes described above
3) While still holding the wire in place, plug in the board.
4) At this point the board should be in programming mode.
5) Flash the firmware in this repo.
