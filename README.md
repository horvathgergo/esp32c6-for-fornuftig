# esp32c3-for-fornuftig
Smart controller for IKEA Förnuftig Air Purifiers (ESP32 version)

# Hardware

New version of the fan controller is ready to test. A few changes have been made in the current 3.1 version 😎 

It is based on a pure ESP32-C3 wifi chip. For the current PCB prototype, the FH4 version of the chip was used (with 4MB internal memory), but left space on the board for external flash too. 

The previous voltage regulator was also replaced with a better one. The current AP63203WU has higher frequency (>1MHz) and requires cheaper and smaller-sized external components. 

In this version an SR1712F 4-gear 15mm D-shaft rotary switch was attached for manual operation.

![IMG_0087](https://github.com/horvathgergo/esp32c3-for-fornuftig/assets/44551566/d7020a72-bf55-40b3-8224-49519e7beffc)
