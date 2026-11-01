# DashIO

So, what is **DashIO**? It is a quick effortless way to connect your IoT device to your phone, tablet or iPad using the free **Dash IoT** app. It allows easy setup of controls such as Dials, Text Boxes, Charts, Graphs, Notifications..., from your IoT device. You can define the look and layout of the controls on your phone from your IoT device. There are three methods to connect to your phone; Bluetooth Low Energy (BLE), TCP or MQTT.

What is **dash** then? **dash** is an IoT platform based on an MQTT server with extra bits added in to allow you to store data, manage your devices, send notifications, share your devices, and save your **Dash** app setup.

## Discord Community

[![](https://img.shields.io/discord/1313341785430429747?color=5865F2&logo=Discord&style=flat-square)](https://discord.gg/fqvhNV3GQB)

Be a part of the DashIO community by joining our [Discord Server](https://discord.gg/fqvhNV3GQB)

## Documentation

For the **DashIO** SAMD NINA library documentation: <a href="https://dashio.io/guide-arduino-samd">https://dashio.io/guide-arduino-samd</a>

For the big picture on **DashIO**, take a look at our website: <a href="https://dashio.io">dashio.io</a>

For all documentation and software guides: <a href="https://dashio.io/documents">dashio.io/documents</a>

## Requirements

- **ArduinoBLE** 2.0.0 or later, and **WiFiNINA** 2.x (Arduino's official library, not WiFiNINA_Generic). Both use the **Arduino\_SpiNINA** driver, which the Arduino IDE installs with them.
- **NINA firmware 3.0.0 or later** on the board (Nano 33 IoT, MKR WiFi 1010). From ArduinoBLE 2.0.0, BLE talks to the NINA module over SPI, which needs this firmware. To update it in the Arduino IDE, use **Tools → Firmware Updater**. With older firmware, BLE fails to start with "Please ensure your NINA firmware is version 3.0.0 or higher".

## Dash IoT Application

The **Dash** app is free and available for both Apple and Android devices. Use it to create beautiful and powerful user interfaces to you IoT devices.


<img src="https://dashio.io/wp-content/uploads/2020/11/IMG_4154.jpeg" width="200" />

<img src="https://dashio.io/wp-content/uploads/2020/12/IMG_4203.jpeg" width="600" />

## Release Notes

For previous releases, please refer to <a href="https://github.com/dashio-connect/arduino-dashio">github.com/dashio-connect/arduino-dashio</a>, which is now obsolete.

### 1.3.0 (30 October 2026)

- Use the official **WiFiNINA** library instead of **WiFiNINA_Generic**. ArduinoBLE 2.x and WiFiNINA_Generic each include their own copy of the NINA SPI driver, so a sketch using both failed to link with "multiple definition of `SpiDrv::...`" errors.
- Requires ArduinoBLE 2.0.0 or later, and NINA firmware 3.0.0 or later (see Requirements).
- With NINA firmware 3.0.0 or later, BLE and WiFi can run at the same time. BLE with TCP works well. BLE with the dash MQTT connection works but runs very slowly, because of the NINA module's limitations when it encrypts while BLE is running. See the software guide.
- Added **MDNS_Generic** to the library dependencies (it was used but not listed).
- Removed an unused global mDNS object, saving some RAM.
- BLE writes longer than 20 bytes from the phone work with ArduinoBLE 1.4.1 or later. In the Android **Dash** app, the **BLE 20 Byte Writes (Arduino SAMD)** setting can be turned off for faster messaging.

### 1.2.3 (23 January 2025)

- Improve BLE incoming message processing to handle longer messages correctly.

### 1.2.2 (12 January 2025)

- Fix BLE incoming message errors associated with some Android phones/tablets.

### 1.2.1 (1 October 2024)

- Updated library properties

### 1.2.0 (30 September 2024)

- Release to Arduino library
