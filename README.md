# Smart Air Purifier ESP32-C6 (Project Fornuftig)
Smart controller for IKEA Förnuftig Air Purifiers (ESP32 version)

# Hardware

![Image](https://github.com/user-attachments/assets/0f3e21fe-031f-41c8-8963-674925d908cf)
New version of the fan controller is ready to test. A few changes have been made in the current 4.0 version 😎 

It is based on a pure ESP32-C6 chip that enables to leverage WIFI, Zigbee or Thread.

The previous voltage regulator was also replaced with a better one. The current AP63203WU has higher frequency (>1MHz) and requires cheaper and smaller-sized external components. 

In this version an SR1712F 4-gear 15mm D-shaft rotary switch was attached for manual operation.

![Image](https://github.com/user-attachments/assets/72a3cf68-f232-4449-a807-bd6899b04a16)

MCU programming: the board supports only UART programming via TX, RX pin headers on the righ side of the ESP module (no USB interface), thus USB to TTL converter is necessary to upload your firmware. It is recommended to supply the module via VIN and GND pins during initial setup. It is compatible with +5-24V supply (note: in serial bootloader mode you need to supply at least +5V. Standard +3v3 won't work as you may experience some voltage drop via the buck converter so make sure to provide enough voltage during flashing. Supplying +5V won't hurt the ESP as the VIN pin of the board is not directly connected to chip). 

The ESP32-C6 will enter the serial bootloader when GPIO9 (BOOT) is held low on reset. It is recommended to use jumper caps on the BOOT and RST pin headers.
