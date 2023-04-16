# ESPhome Ventoy boot selector
 Use ESPhome to boot you PC and choose which OS to boot via the COM port (RS232 UART).

![Screenshot 2023-04-16 at 17 15 46](https://user-images.githubusercontent.com/6509533/232326230-1ae06160-2678-4d83-b552-7d3397352faa.jpg)

# Overview
I have a PC with multiple operating systems installed. To achieve this I have an external SSD drive attached via USB, the drive is setup with the Ventoy installer and each OS has it's own VHDX on the same drive. Upon boot Ventoy finds each of the VHDX files and presents them for selection. This all works flawlessly, I have Win10, Win11 and Ubuntu and others working great.



## The Problem
I would like to use this PC headlessly with no screen, keyboard or mouse attached. But I still want to control which OS is booted from the Ventoy bootloader.

I would also like to turn the PC on and off, and have a sensor in Home Assitant updated with the current status of the PC.

## Solution
Ventoy supports mirroring the boot menu selection screen to the motherboard COM port (UART), to do this you just need to add a line to the ventoy.json file like this:

```
{
    "control":[
        { "VTOY_MAX_SEARCH_LEVEL": "0" },
        { "VTOY_VHD_NO_WARNING": "1" }
    ],
    "theme":{
        "display_mode": "serial_console",
        "serial_param": "--unit=0 --speed=115200"
    }
}
```

Using an ESP32 module I connected a COM port module like this:

![wiring-diagram](https://user-images.githubusercontent.com/6509533/232055543-6ee5cb83-e1e6-4b02-ba04-a77a95813440.jpg)

My PC motherboard has a COM port socket but no cable to plug into it, so bought this adaptor cable.

![Screenshot 2023-04-14 at 14 14 53](https://user-images.githubusercontent.com/6509533/232055584-6382896a-dad0-480b-9000-c0747c3a1cd6.jpg)

Lastly I flashed ESPhome with is config file which presents button you can press in HomeAssistant to move the cursor up / down and press return. You can also set the correct boot position and turn on the PC, the ESPhome device detects when Ventoy has booted to the menu and will move the cursor to the correct position and press enter.

```yaml
esphome:
  name: rs232-test
  friendly_name: rs232-test

esp32:
  board: esp32dev
  framework:
    type: arduino

# globals:
#   - id: my_global_string
#     type: std::string
#     restore_value: no  # Strings cannot be saved/restored
#     initial_value: '"Global value is"'


# Enable logging
logger:
  level: VERBOSE #makes uart stream available in esphome logstream

# Enable Home Assistant API
api:
  encryption:
    key: ""

ota:
  password: ""

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Rs232-Test Fallback Hotspot"
    password: ""

captive_portal:

uart:
  baud_rate: 115200
  tx_pin: 17
  rx_pin: 16
  id: UART2
  stop_bits: 1
  data_bits: 8
  parity: NONE
  debug:
    direction: RX
    dummy_receiver: true
    after:
      delimiter: ";78H\e[m"
      #delimiter: "\n"
      #bytes: 98
    sequence:
      lambda: !lambda |-
        UARTDebug::log_string(direction, bytes); 

        std::string str(bytes.begin(), bytes.end()); // converts above to a string 

        // VenToy boot menu last line of loaded text screen
        if (str.find("                                      \e[5;78H") != std::string::npos) 
        {
          ESP_LOGD("custom", "INFO: VenToy Menu has just loaded");
          // get menu item position from HA
          ESP_LOGI("main", "Boot position number: %f", id(boot_position).state);

          if ((id(boot_position).state) == 1.000000)
          {
            ESP_LOGD("custom", "INFO: boot position is 1");
            id(return_key).press();
            // after boot reset boot option to zero to a new choice must be made on next boot
            auto call = id(boot_position).make_call();
            call.set_value(0);
            call.perform();
          }

          if ((id(boot_position).state) == 2.000000)
          {
            ESP_LOGD("custom", "INFO: boot position is 2");
            id(down_arrow).press();
            id(return_key).press();
            // after boot reset boot option to zero to a new choice must be made on next boot
            auto call = id(boot_position).make_call();
            call.set_value(0);
            call.perform();
          }

          if ((id(boot_position).state) == 3.000000)
          {
            ESP_LOGD("custom", "INFO: boot position is 3");
            id(down_arrow).press();
            id(down_arrow).press();
            id(return_key).press();
            // after boot reset boot option to zero to a new choice must be made on next boot
            auto call = id(boot_position).make_call();
            call.set_value(0);
            call.perform();
          }
        }


# these buttons send keyboard commands over the com port
button:
  - platform: template 
    name: "1.UP arrow"
    id: up_arrow
    on_press:
      - uart.write: "\x1b[A"

  - platform: template
    name: "2.DOWN arrow"
    id: down_arrow
    on_press:
      - uart.write: "\x1b[B"

  - platform: template
    name: "3.Return"
    id: return_key
    on_press:
      - uart.write: "\r\n"

# these buttons are optional but allow you to choose and set the boot os position using a button on HomeAssistant
  - platform: template 
    name: "boot os 1"
    id: pos1
    on_press:
      number.set:
        id: boot_position
        value: 1

  - platform: template 
    name: "boot os 2"
    id: pos2
    on_press:
      number.set:
        id: boot_position
        value: 2

  - platform: template 
    name: "boot os 3"
    id: pos3
    on_press:
      number.set:
        id: boot_position
        value: 3

# This is the boot menu position number that you want to boot
# In my case I set this is home assistant, when esphome starts
# it checks to see if it's in the Ventoy menu then selects
# boot position and presses return
number:
  - platform: template
    name: "boot position"
    id: boot_position
    optimistic: true
    min_value: 0
    max_value: 10
    step: 1
```

## Notes:

It would be better to parse each line on the boot menu and return it to Home Assistant as a drop down selector, but that seemed like a lot of work. If you do this let me know :-)

Ventoy does not work with USB COM ports so far as I can tell. This would be super useful, and there is even a GRUB module for two types of USB COM port (FTDI)adaptor, but I've tested both and can't get either of them to work YMMV.
