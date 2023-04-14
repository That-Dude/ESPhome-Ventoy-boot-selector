# ESPhome Ventoy boot selector
 Use ESPhome to set the Ventoy boot option via RS232 UART

# Overview
I have a PC with multiple operating systems installed. To achieve this I have an external SSD drive attached via USB, the drive is setup with the Ventoy installer and each OS has it's own VHDX on the same drive. Upon boot Ventoy finds each of the VHDX files and presents them for boot. This all works flawlessly, I have Win10, Win11 and Ubuntu working great.

## The Problem
I would like to use this PC in headless mode with no screen, keyboard or mouse attached. But I still want to control which OS is booted from the Ventoy bootloader.

## Solution
Ventoy supports mirroring the boot menu selection screen to the motherboard COM port (UART), to do this you just need to add a line to the ventoy.json file like this:


Using an ESP32 module I connected a COM port module like this:


My PC motherboard has a COM port socket but no cable to plug into it, so bought this adaptor cable.


Lastly I flashed ESPhome with is config file which presents button you can press in HomeAssistant to move the cursor up / down and press return. You can also set the correct boot position and turn on the PC, the ESPhome device detects when Ventoy has booted to the menu and will move the cursor to the correct position and press enter.

## Notes:

I would be better to parse each line on the boot menu and return it to Home Assistant as a drop down selector, but that seemed like a lot of work. If you do this let me know :-)

Ventoy does not work with USB COM ports so far as I can tell. This would be super useful, and there is even a GRUB module for two types of USB COM port adaptor, but I've tested both and can't get either of them to work YMMV.