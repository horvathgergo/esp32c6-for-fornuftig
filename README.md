# Smart Air Purifier ESP32-C6 (Project Fornuftig)
Smart controller for IKEA Förnuftig Air Purifiers (ESP32 version)

# Hardware

![Image](https://github.com/user-attachments/assets/0f3e21fe-031f-41c8-8963-674925d908cf)
New version of the fan controller is ready to test. A few changes have been made in the current 4.0 version 😎 

It is based on an ESP32-C6-WROOM-1N8 module that enables to leverage WIFI 6, Bluetooth 5, Zigbee 3.0 or Thread.

The previous voltage regulator was also replaced with a better one. The current AP63203WU has higher frequency (>1MHz) and requires cheaper and smaller-sized external components. 

In this version an SR1712F 4-gear 15mm D-shaft rotary switch was attached for manual operation.

![Image](https://github.com/user-attachments/assets/72a3cf68-f232-4449-a807-bd6899b04a16)

MCU programming: the board supports only UART programming via TX, RX pin headers on the righ side of the ESP module (no USB interface), thus USB to TTL converter is necessary to upload your firmware. It is recommended to supply the module via VIN and GND pins during initial setup. It is compatible with +5-24V supply (note: in serial bootloader mode you need to supply at least +5V. Standard +3v3 won't work as you may experience some voltage drop via the buck converter so make sure to provide enough voltage during flashing. Supplying +5V won't hurt the ESP as the VIN pin of the board is not directly connected to chip). 

The ESP32-C6 will enter the serial bootloader when GPIO9 (BOOT) is held low on reset. It is recommended to use jumper caps on the BOOT and RST pin headers.

# ESPHOME Configuration

Although the ESP32-C6 chip is not officially supported by ESPHome yet, you can use its Wi-Fi capabilities without any issues by defining the board type and esp-idf framework version (tested only with ESPhome 2025.01). In the future, ESPHome is expected to support Zigbee on the C6 chip as well. For now, you can only use Espressif Matter SDK to develop Thread-based firmware.

Example config:

```yaml

substitutions:
  friendly_name: Air Purifier
  fan_name: air_purifier

esphome:
  name: air-purifier

esp32:
  board: esp32-c6-devkitc-1
  flash_size: 8MB
  variant: esp32c6
  framework:
    type: esp-idf
    version: "5.3.1"
    platform_version: 6.9.0

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "YOUR KEY"

ota:
  - platform: esphome
    password: "YOUR PASSWORD"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Air Purifier Fallback Hotspot"
    password: "YOUR PASSWORD"

captive_portal:
    

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
