# Nebula Light
## Credits go to:
Based on the idea of [3ative - How-to make a 'Galaxy Projector LED Nebula Light' smart and work with Home Assistant](https://youtu.be/YwHWbcuztuY).

## Differences
3ative uses the "dumb" nebula light. Meaning no wifi, no tuya integration, contolling the light is done by using a regular IR remote control. 

I am using the smart version of the nebula light, which includes the use of the Tuya IOT platform. 

## Why go through all the trouble!?
I am not a big fan of cheap and "smart" devices living inside my network. They are a potential security risk as security is not a big priority for companies like Tuya. 

That is why I wanted to flash ESPhome on the device and be sure I have full control of the code running on the nebula light. And so the story begins...

## Tuya WB3S 
Unfortunately Tuya switched from a regular ESP8266 chip to another chip, designed by Tuya itself, called WB3S:
 
![Tuya WB3S][img1]

The pin out of the device is exactly the same as a ESP12:
![Tuya and ESP12][img2]

Swapping the WB3S with an ESP12 is straight forward:
- Gently remove the Tuya chip using hot air gun or solder station
- Remove remaining solder and clean PCB using some alcohol
- solder ESP12 on the PCB
- Add resistor between pin GPIO15 and GND to force the ESP12 in normal boot mode. Any resistor between 2K to 10K will do the job.

## Detailed description:
First you will need to open the case by removing four screws on the side of the device. After that you will be able to open the device by removing the top part of the case:
![Opening the device][img3]

Remove the 3 screws that hold the PCB in place and disconnect the cables running to  the RGB led, laser, motor and the button.

Next remove the PCB from the case.
![PCB front][img4]
![PCB back][img5]

Remove the Tuya WB3S chip with your prefered way. I personaly used a hot air gun. Once the WB3S is removed, remove any excess solder and clean the PCB with some alcohol.

Flash ESP12 module with code [below](##ESPHome-config) 
I used a ESP8266 burner for this.

Solder the ESP12 into place and add a resistor between GPIO15 and GND
![ESP12][img6]

Now you can test the ESP12 by connecting all wires
![Testing][img7]
If all works, put the case back together and enjoy you ESPHome enabled nebula light!

![end result][img8]


## ESP12 pin configuration
| GPIO          | Function
| ------------- |-------------
| GPIO0         | LED outside case
| GPIO4         | Red
| GPIO5         | Laser
| GPIO12        | Green
| GPIO13        | Motor
| GPIO14        | Blue
| GPIO15        | LED outside case
| GPIO16        | Button

## Tools used
- [Nebula light, white](https://www.aliexpress.com/item/1005001427043343.html?spm=a2g0s.9042311.0.0.27424c4dFY0lak)
- [Nebula light, black](https://www.aliexpress.com/item/1005001635598503.html?spm=a2g0s.9042311.0.0.27424c4dfXN5os)
- [Solder station](https://www.aliexpress.com/item/32963583169.html?spm=a2g0s.9042311.0.0.27424c4dfXN5os)
- [Helping hand](https://www.aliexpress.com/item/4000811748567.html?spm=a2g0s.9042311.0.0.27424c4dfXN5os)
- [Development Board Test Burning Fixture Tool Downloader](https://www.aliexpress.com/item/32953841956.html?spm=a2g0s.9042311.0.0.27424c4dfXN5os)
- [ESP12F](https://www.aliexpress.com/item/32863572642.html?spm=a2g0s.9042311.0.0.27424c4dfXN5os)

## ESPHome config
For some reason, and I don't know why, if the motor is stopped the brightness of the laser is effected. By powering the motor at very low power (15 percent) the brightness of the laser is consistant and will not rotate the laser or RBG led. 

```YAML
substitutions:
  name: <INSERT PREFFERED NAME>
  
esphome:
  name: ${name}
  platform: ESP8266
  board: esp12e

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  domain: !secret domain_iot
  power_save_mode: none

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "${name}HotSpot"
    password: !secret wifi_fallback_password
    
captive_portal:

# Enable logging
logger:
  level: INFO

# Enable Home Assistant API
api:
  password: !secret api_password

ota:
  password: !secret ota_password

# Enable Web server.
web_server:
  port: 80

light:
  - platform: rgb
    name: ${name}_light
    id: rgb_light
    red: red
    green: green
    blue: blue
    restore_mode: ALWAYS_OFF
    effects:
      - flicker:
          name: Flicker
          alpha: 95%
          intensity: 2.5%
      - random:
          name: Random
          transition_length: 2.5s
          update_interval: 3s
      - random:
          name: Random Slow
          transition_length: 10s
          update_interval: 5s

  - platform: monochromatic
    name: ${name}_laser
    id: laser
    output: laser_pwm
    restore_mode: ALWAYS_OFF
  - platform: monochromatic
    name: ${name}_bled
    id: bled
    output: bled_pwm
    restore_mode: ALWAYS_OFF
    internal: true
  - platform: monochromatic
    name: ${name}_rled
    id: rled
    output: rled_pwm
    restore_mode: ALWAYS_OFF
    internal: true

fan:
  platform: speed
  name: "${name}_Motor"
  id: motor
  output: motor_pwm
  restore_mode: ALWAYS_OFF

output:
  - platform: esp8266_pwm
    id: red
    pin: GPIO4
    inverted: true
  - platform: esp8266_pwm
    id: green
    pin: GPIO12
    inverted: true
  - platform: esp8266_pwm
    id: blue
    pin: GPIO14
    inverted: true

  - platform: esp8266_pwm
    id: laser_pwm
    pin: GPIO5
    # max_power: 80%
    # frequency: 2000 Hz
    inverted: true
  - platform: esp8266_pwm
    id: motor_pwm
    pin: GPIO13
    min_power: 15%

  - platform: esp8266_pwm
    id: bled_pwm
    pin: GPIO0
    inverted: true
  - platform: esp8266_pwm
    id: rled_pwm
    pin: GPIO15
    inverted: true

binary_sensor:
  - platform: gpio
    pin: 
      number: GPIO16
      mode: INPUT_PULLDOWN_16
      inverted: true
    name: ${name}_button
    on_press:
      then:
        - light.turn_on: 
            id: rgb_light
            brightness: 30%
        - light.turn_on: 
            id: laser
            brightness: 65%
        - delay : 1h
        - light.turn_off: 
            id: rgb_light
        - light.turn_off: 
            id: laser
    
interval:
  - interval: 1s
    then:
      if:
        condition:
          wifi.connected:
        then:
          - light.turn_on: 
              id: bled
              brightness: 50%
        else:
          - light.turn_off: 
              id: bled
  - interval: 1s
    then:
      if:
        condition:
          api.connected:
        then:
          - light.turn_on: 
              id: rled
              brightness: 50%
        else:
          - light.turn_off: 
              id: rled
```
🎁 Found this useful or want to say 'thanks' and support my efforts...

[![BMC](https://www.buymeacoffee.com/assets/img/custom_images/white_img.png)](https://www.buymeacoffee.com/kireque) **And leave a me a message to let me know.**  ❤

🍺 CHEERS! 👍


[img1]: images/tuya_wb3s.jpg
[img2]: images/wb3s_esp12.jpg
[img3]: images/opening_the_case.jpg
[img4]: images/pcb_front.jpg
[img5]: images/pcb_back.jpg
[img6]: images/pcb_esp12.jpg
[img7]: images/testing.jpg
[img8]: images/end_result.jpg
