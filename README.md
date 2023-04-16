# ESPhome Ventoy boot selector
 Use ESPhome to boot you PC and choose which OS to boot via the COM port (RS232 UART).

![Screenshot 2023-04-16 at 17 15 46](https://user-images.githubusercontent.com/6509533/232326230-1ae06160-2678-4d83-b552-7d3397352faa.jpg)

# Overview
I have a PC with multiple operating systems installed. To achieve this I have an external SSD drive attached via USB, the drive is setup with the Ventoy bootloader and each OS has it's own VHDX on the same drive which VenToy can boot directly in bare metal style.

When the PC starts the Ventoy bootloader finds each of the VHDX files and presents them for selection. This all works flawlessly, I have Win10, Win11 and Ubuntu and others working great.

I also wanted to turn the PC on/off remmotely and integrate a sensor showing the on/off state. For this I used a couple of 1k resistors connected to pc817 opto-couplers, one is attache the motherboard header pins which turn the PC on and off and the other is connected the LED pins, this is how I'm detecting if the PC is on or off.

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

# ************************************** LOCAL WEB SERVER **************************************
web_server:
  local: true
  port: 80
  auth:
    username: admin
    password: !secret web_server_password

# ************************************ UART for RS232 module ***********************************

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

        std::string str(bytes.begin(), bytes.end()); // converts above to a string called str ?

        // VenToy boot menu last line of loaded text screen
        if (str.find("                                      \e[5;78H") != std::string::npos) 
        {
          ESP_LOGD("custom", "INFO: VenToy Menu has just loaded");
          // update HomeAssistant Boot Status
          id(text_status).publish_state("Ventoy menu");
          
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
            id(text_status).publish_state("booting OS...");
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
            id(text_status).publish_state("booting OS...");
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
            id(text_status).publish_state("booting OS...");
          }
        }

        // Set boot position from boot OS by typing: echo boot1 > com1
        if (str.find("boot1") != std::string::npos)
        {
            // set next boot to position1
            auto call = id(boot_position).make_call();
            call.set_value(1);
            call.perform();
            ESP_LOGI("main", "Boot position number: %f", id(boot_position).state);
        }

        if (str.find("boot2") != std::string::npos)
        {
            // set next boot to position1
            auto call = id(boot_position).make_call();
            call.set_value(2);
            call.perform();
            ESP_LOGI("main", "Boot position number: %f", id(boot_position).state);
        }

        if (str.find("boot3") != std::string::npos)
        {
            // set next boot to position1
            auto call = id(boot_position).make_call();
            call.set_value(3);
            call.perform();
            ESP_LOGI("main", "Boot position number: %f", id(boot_position).state);
        }

        if (str.find("status:") != std::string::npos)
        {
            std::string str1 = "";
            std::string str3 = str;
            std::string::size_type pos = str3.find(" ");
            str1 = str3.substr(pos + 1); // the part after the space

 
            // report OS status to HomeAssistant
            // requires a process running os the OS to write the status to the com port
            id(text_status).publish_state(str1);
        }


# ************************************** BUTTONS **************************************

# Create a series of buttons becuase they look better on the HA front end

# these buttons send keyboard commands over the com port
# https://notes.burke.libbey.me/ansi-escape-codes/
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

  - platform: template
    name: "Power Button Toggle"
    id: power_button_short_press
    icon: "mdi:toggle-switch-outline"
    on_press:
      - logger.log: "PC power button - short press"
      - switch.turn_on: power_short_press

  - platform: template
    name: "Power Hard Shutdown"
    id: power_button_long_press
    icon: "mdi:toggle-switch-outline"
    on_press:
      - logger.log: "PC power on - long press"
      - switch.turn_on: power_long_press


# ************************************** SWITCHES **************************************

switch:
  - platform: gpio
    name: "powershortpress"
    internal: true # hide from HA - we're using the button above to tigger this switch
    pin: 14   # Power button output pin
    id: power_short_press
    inverted: no
    on_turn_on:
    - delay: 80ms
    - switch.turn_off: power_short_press

  - platform: gpio
    name: "powerlongpress"
    internal: true # hide from HA - we're using the button above to tigger this switch
    pin: 14   # Power button output pin
    id: power_long_press
    inverted: no
    on_turn_on:
    - delay: 10000ms
    - switch.turn_off: power_long_press


# ************************************** NUMBER **************************************

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

# ************************************** BINARY SENSORS **************************************

binary_sensor:
  - platform: gpio
    name: "PC Power Status"
    pin:
      number: 27
      mode: INPUT_PULLUP
      inverted: true
    filters:
      - delayed_off: 30ms

# ************************************** TEXT SENSORS **************************************

text_sensor:
  - platform: template
    name: "Boot status"
    #icon: "mdi:water-percent"
    id: text_status

```

## Optional extras
Once an OS has booted the COM port can be used to send status messages to the ESPhome device, these are shown in HomeAssistant. I've created a simple script that runs at start up and posts status updates every 5 seconds.

```dos
@ECHO OFF
REM Windows Batch file to report hostname to the COM port, where my ESPhome
REM device uses the info update Home Assistant with the currently booted OS
REM os-status.cmd

REM The next line makes the script run minimized
if not DEFINED IS_MINIMIZED set IS_MINIMIZED=1 && start "" /min "%~dpnx0" %* && exit

MODE COM1:115200,N,8,1
set host=%COMPUTERNAME%

:loop
ECHO | set /p dummyName="status: %host%" > COM1
timeout /t 5 /nobreak > NUL
goto loop

ECHO "The loop was broken"
pause
exit
```

It's easy to create a scripts that reboot the PC into a different OS:

```
REM Reboots the PC into the OS at positoin 1 in the Ventoy menu boot1.cmd
MODE COM1:115200,N,8,1
ECHO "boot1" > COM1
timeout /t 10
shutdown.exe /r /t 00
ECHO | set /p dummyName="status: rebooting %host%" > COM1
```

## Notes:
It would be better to parse each line on the boot menu and return it to Home Assistant as a drop down selector, but that seemed like a lot of work. If you do this let me know :-)

Ventoy does not work with USB COM ports so far as I can tell. This would be super useful, and there is even a GRUB module for two types of USB COM port (FTDI)adaptor, but I've tested both and can't get either of them to work YMMV.
