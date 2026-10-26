# Kogan-KAWFPAC07YA-Airconditioner-Tuya-to-ESPHome

This repo has the config and some generalized instructions used on two seperate Kogan KAWFPAC07YA Portable Air Conditioners to flash ESPHome, I have since intergrated both to my Home Assistant, with it successfully (and happily) controlling the three of the four "Mode" settings, Cool, Fan, and Dry (excluding 'Sleep'), the temperature, Power On/Off, and the Fan Speed. 
All states report back to Home Assisstant accurately excluding the ambient temperature displayed while the device is on standby. Also i have not yet intergrated the timer. I will keep this repo updated as I make more progress, feel free to contribute, it seems pretty straightforward.
The Wi-Fi module in mine is a WBR1 (Realtek RTL8720CF) on a JWBR2S carrier board with Tuya firmware. It unplugged from the AC's mainboard by a 4-pin connector, sitting about halfway down the left side of the unit if you looked at it from the front..

There are 6 holes running down the side of the JWBR2S:
TXD, RXD, Log_TX, Log_RX, GND, 3.3V,
And there are three more holes on the back of the board, RST, A_0 and A_1.                      

-TXD / RXD (PA14 / PA13): UART0, the link to the AC's MCU (9600 baud).
-Log_TX / Log_RX: UART2, used for flashing and for the ESPHome logger.
-A_0: boot pin. High at reset = download mode.
-A_1: test mode i think. I didn't use it.

!!Flashing!!

You need a USB-UART adapter with 3.3 V logic, a 3.3 V supply for the module, and I used ltchiptool (https://github.com/libretiny-eu/ltchiptool).
For the adapter I used an ESP32S3 Chip with ESP USB Bridge (https://github.com/espressif/esp-usb-bridge) firmware, and just some jumper leads coming from the pins on the chip to the contacts on the AC module. 

Wiring:

Carrier: Connect to:                       
Log_TX   adapter RX                       
Log_RX   adapter TX                       
GND      adapter GND                      
3.3V     3.3 V supply                     
A_0      3.3 V (held high the whole time) 
RST      tap briefly to GND to reset      

Don't let anything pull RXD (PA13) low, or the chip won't enter download mode. 
How to enter download mode:
1. Connect A_0 to 3.3 V and keep it connected for the whole session. Pulling it mid-read kills the dump; a blob of Blu Tack holding the wire works well.
2. Tap RST to GND, resets the device.
3. Run ltchiptool. On the log you should see something like 'Download Image over *the adapter*'.
Back up the stock firmware, dudes.
Do this first, it's the only way back. Also the dump holds the units MAC and Tuya keys so dont post it.
The dump should be exactly 2,097,152 bytes, and `ltchiptool flash file` should report it as valid.

Cmds i used to build/flash:

cp secrets.yaml.example secrets.yaml 
esphome compile ac-configname.yaml
ltchiptool flash write -d /dev/ttyUSB0 firmware.uf2

NOTE!:
Probably leave the wire on A_0 throughout the flash, I honestly just used blutack. Removing had mixed results however mostly stopped the flash.

After completion, verify thru logs it seems to be working (I added it to my HA before putting it back in the AC), remove the A_0 wire, reset, and plug the module back into the AC. It joined my Wi-Fi and Home Assistant discovered it pretty well immediately. Later updates I've just done OTA.

# Configuration notes

These are already handled in `aircon.yaml`, but they're worth knowing if you adapt it:

UART0 pins: The `wbr1` board definition didn't define UART0, so the config adds the build flags `-DPIN_SERIAL0_RX=13u` and `-DPIN_SERIAL0_TX=14u` (see [esphome#13596](https://github.com/esphome/esphome/issues/13596)).
Logger: The logger stays on UART2 (the Log pins) and the MCU is on UART0, so there's no need for `baud_rate: 0`.
Current temperature: The unit has no current-temperature datapoint. Without one, ESPHome's Tuya climate code bails out and the mode is never read, so `on_boot` sets `current_temperature` to 0.
Mode changes from HA: The MCU doesn't echo mode changes made over Wi-Fi I dont think, so `on_control` waits 1 s and then sends the Tuya status query (`55 AA 00 08 00 00 07`) to read the state back.

## Datapoints

DP                Function            Values               
-1                -Power               bool                 
-2                -Target temperature  °C                   
-4                -Mode                0 cool, 1 fan, 2 dry 
-5                -Fan speed           0 low, 1 high        
-103              -Timer                      
-106              -Fault               bitmask              
-19, 104, 105, 107  Unknown, probably the missing ones.                    

### License

MIT
