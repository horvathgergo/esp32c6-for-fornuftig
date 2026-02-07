# Smart Air Purifier ESP32-C6 (Project Fornuftig)
Smart controller for IKEA Förnuftig Air Purifiers (ESP32 version)

# Hardware

![Image](https://github.com/user-attachments/assets/0f3e21fe-031f-41c8-8963-674925d908cf)

Fan Controller v4.0 – Ready for Testing 😎

A new version of the fan controller is now ready for testing. Several improvements have been introduced in **version 4.0**.

The board is based on the **ESP32-C6-WROOM-1N8** module, enabling support for **Wi-Fi 6, Bluetooth 5, Zigbee 3.0, and Thread**.

The previous voltage regulator has been replaced with a better alternative. The new **AP63203WU** operates at a higher switching frequency (>1 MHz) and requires **smaller and more cost-effective external components**.

This version also adds an **SR1712F 4-position, 15 mm D-shaft rotary switch** for manual operation.

![Image](https://github.com/user-attachments/assets/72a3cf68-f232-4449-a807-bd6899b04a16)

MCU programming: The board supports UART programming only via the TX and RX pin headers on the right side of the ESP module (there is no USB interface on the board). Therefore, a **USB-to-TTL converter is required** to upload firmware. USB-TTL converters are cheap and broadly available.

It is recommended to power the module via the VIN and GND pins during the initial setup. The board is compatible with a +5–24V supply.

Note: In serial bootloader mode, you must **supply at least +5V**. A standard +3.3V supply will not work, as voltage drop across the buck converter may prevent proper operation. Make sure to provide sufficient input voltage during flashing. Supplying +5V is safe, as the **VIN pin is not directly connected to the ESP chip**.

he ESP32-C6 enters serial bootloader mode when **GPIO9 (BOOT)** is held low during reset. It is recommended to use jumper caps on the **BOOT** and **RST** pin headers.

Flashing procedure:
1. Connect the USB-to-TTL converter to the board.
2. Power the device.
3. Place a jumper cap on the BOOT pin.
4. Place a jumper cap on the RST pin (both pins are now held low).
5. Remove the jumper cap from the RST pin.
6. Flash the firmware.
7. If flashing is successful, remove the jumper cap from the BOOT pin.
8. Restart the device.

# Components

| Designator | Items                              | Designation              |
|------------|------------------------------------|--------------------------|
| IC1        | ESP32-C6-WROOM-1-N8                | ESP Modul                |
| C2         | 22uF, >6V, 1206                    | Decoupling capacitor     |
| C4         | 100nF, >6V, 0805                   | Decoupling capacitor     |
| R1         | 10K, 0805                          | IO9 resistor             |
| R4         | 10K, 0805                          | EN resistor              |
| R5         | 10K, 0805                          | IO8 resistor             |
| C1         | 10uF, >30V, 1206                   | Buck input capacitor     |
| C19        | 22uF, >6V, 1206                    | Buck output capacitor    |
| C11        | 22uF, >6V, 1206                    | Buck output capacitor    |
| C3         | 100nF, >6V, 0805                   | Buck bootstrap capacitor |
| L1         | 4.7uH, L_6.7x7.0_H2.8              | Buck inductor            |
| U1         | TSOT-23-6                          | AP63203WU                |
| J1         | JST_XH 2P                          | Power                    |
| J2         | JST_XH 4P                          | Fan                      |
| F1         | 10A, 1206                          | Fuse                     |
| J3         | PinHeader_1x02_P2.54mm_Vertical    | UART                     |
| J4         | PinHeader_1x02_P2.54mm_Vertical    | RST                      |
| J5         | PinHeader_1x02_P2.54mm_Vertical    | BOOT                     |
| S1         | SR1712F                            | SR1712F switch           |

# BOM and CPL

This hardware project was primarily designed for manual assembly.
Due to multiple requests, I have uploaded the BOM and CPL files to this repository. Please note that these files are provided for reference only. I have not tested this design with automated assembly or PCB manufacturing services, so there may be inaccuracies or missing details.
If you plan to use these files for automated assembly, please review and verify them carefully before placing any orders.

Please also note that no suitable rotary switch or SMD fuse was found in the JLCPCB parts library, so these parts may require manual assembly.

# ESPHOME Configuration

The ESP32-C6 chip is now officially supported by ESPHome, so you can use its Thread capabilities without any issues. It has been tested with ESPhome version 2025.12 using the following example.

Example config:

```yaml

substitutions:
  friendly_name: Air Purifier
  fan_name: air_purifier

esphome:
  name: air-purifier

esp32:
  board: esp32-c6-devkitc-1
  framework:
    type: esp-idf

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "YOUR KEY"

ota:
  - platform: esphome
    password: "YOUR PASSWORD"

#THREAD
network:
  enable_ipv6: true

openthread:
  tlv: "YOUR TLV"
    

output:
  - platform: ledc
    pin: 18 
    frequency: 150 Hz
    id: pwm_output
    min_power: 0.5
    max_power: 0.5
    zero_means_zero: true

fan:
  - platform: speed
    output: pwm_output
    name: "$friendly_name"
    id: "$fan_name"
    restore_mode: RESTORE_DEFAULT_OFF
    on_speed_set:
      lambda: |-
        if(id($fan_name).speed != id(fan_speed) && id($fan_name).state){
          id(fan_speed) = id($fan_name).speed;
          id(set_fan_freq).publish_state(id($fan_name).speed);
        }
    on_turn_on:
      lambda: |-
       id(set_fan_freq).publish_state(id($fan_name).speed);
    on_turn_off:
      - output.ledc.set_frequency:
          id: pwm_output
          frequency: !lambda 'return int(0);'
      - logger.log: "Fan Turned Off!"

sensor:
  - platform: pulse_counter
    pin: 
      number: 19
      mode: INPUT_PULLUP
    unit_of_measurement: 'RPM'
    accuracy_decimals: 0
    id: fan_tach
    name: $friendly_name Fanspeed #HA entity
    filters:
      - multiply: 0.1

  - platform: template
    id: set_fan_freq
    name: "$friendly_name Fan Frequency"  #HA entity
    filters:
      - multiply: 3
      - lambda: |-
          if (id($fan_name).state){
            if(x > 300) { // limit max frequency
              return 300;
              
            } else if(x < 36 && x > 3) { // limit low frequency
              return 36;
              
            } else if(x <= 3) {
              return 0;
              
            } else {
              return x;
              
            }
          } else {
            return 0;
          }
    on_value:
      then:
        - output.ledc.set_frequency:
            id: pwm_output
            frequency: !lambda 'return int(x);'
        - logger.log: "on_value called"
        - lambda: |- 
            // send current status to homeassistant
            if(x) {
              auto call = id($fan_name).turn_on();
              call.set_speed(int(x)/3);
              call.perform();
            } else {
              auto call = id($fan_name).turn_off();
              call.perform();
            }

binary_sensor:
  - platform: template
    id: 'knob_0'
    filters:
      - delayed_on_off: 300ms
    lambda: |-
      if (!id(knob_1).state && !id(knob_2).state && !id(knob_3).state) {
        return true;
      } else {
        return false;
      }
    on_press: 
      lambda: |-
        if(!id(speed_0)){
          auto call = id($fan_name).turn_off();
          call.perform();
        }else{
          auto call = id($fan_name).turn_on();
          call.set_speed(id(speed_0));
          call.perform();
        }

  - platform: template                      #Ha entity
    name: "$friendly_name Speed 1"
    lambda: |-
      if (id(set_fan_freq).state >= 60 and id(set_fan_freq).state < 100 and id($fan_name).state) {
        return true;
      } else {
        return false;
      }
      
  - platform: template                      #Ha entity
    name: "$friendly_name Speed 2"
    lambda: |-
      if (id(set_fan_freq).state >= 100 and id(set_fan_freq).state < 200 and id($fan_name).state) {
        return true;
      } else {
        return false;
      }

  - platform: template                      #Ha entity
    name: "$friendly_name Speed 3"
    lambda: |-
      if (id(set_fan_freq).state >= 200 and id($fan_name).state) {
        return true;
      } else {
        return false;
      }      


  - platform: gpio
    id: 'knob_1'
    pin:
      number: 12
      mode: INPUT_PULLUP
      inverted: true
    filters:
      - delayed_on_off: 100ms
    on_press: 
      then:
        - fan.turn_on:
            id: $fan_name
            speed: !lambda 'return id(speed_1);'
    
    
  - platform: gpio
    id: 'knob_2'
    pin:
      number: 11
      mode: INPUT_PULLUP
      inverted: true
    filters:
      - delayed_on_off: 100ms
    on_press: 
      then:
        - fan.turn_on:
            id: $fan_name
            speed: !lambda 'return id(speed_2);'
  
  
  - platform: gpio
    id: 'knob_3'
    pin:
      number: 10
      mode: INPUT_PULLUP
      inverted: true
    filters:
      - delayed_on_off: 100ms
    on_press:
      then:
        - fan.turn_on:
            id: $fan_name
            speed: !lambda 'return id(speed_3);'

globals:
  - id: fan_speed
    type: int
    restore_value: yes
    initial_value: '0'
    
  - id: speed_0
    type: int
    restore_value: yes
    initial_value: '0'
    
  - id: speed_1
    type: int
    restore_value: yes
    initial_value: '33'
    
  - id: speed_2
    type: int
    restore_value: yes
    initial_value: '66'
    
  - id: speed_3
    type: int
    restore_value: yes
    initial_value: '100'
