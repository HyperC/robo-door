# Automated Hidden Garage for a Roborock using ESP32 + ESPHome

A hidden garage door for the Roborock Qrevo Slim, controlled via ESPHome and Home Assistant.

## Installation Guide

### 1. Hardware

You'll need:

- **ESP32** — I used a Seeed XIAO ESP32-C6, but any ESP32 should work (C3, C6, SuperMini, etc.).
- **Metal gear servo** — I used a TD-8125MG (25 kg). It's just strong enough, although I ended up adding a counterweight to the door (explained later).
- **5V power supply** — I used a 5V / 2A adapter to power both the ESP32 and the servo.
- **(P)IR motion sensor** — Any small 5V-compatible (P)IR sensor should work.
- **3D printed parts** — Hinges, servo mount, and linkage arm (STLs attached). You could also build these from hardware store parts if your setup allows it.

### 2. Software

- Home Assistant
- ESPHome
- Roborock integration

## Door Preparation

Cut an opening large enough for the robot to pass through.

I recommend leaving roughly:

- **2–3 cm** clearance on each side
- **6–8 cm** above the robot

The extra height is needed because of the hinge design.

If possible, don't place the opening directly against a wall like I did. It makes the hinge design much more complicated.

If you're making the opening, consider cutting the top edge at roughly **45°**. When the door closes, the seam becomes much less visible and you can reduce the gap significantly.

For the door itself, use something lightweight. I used a thin piece of MDF, although lightweight plywood would probably be a better choice.

## Hinges

A normal piano hinge doesn't work very well here because:

- The hinges shouldn't be visible from the outside.
- There's no room for the door to swing inward — the robot is in the way.

Instead of rotating around a fixed pivot, the hinge creates an offset pivot point. This causes the top edge of the door to move inward first, after which the door rotates outward and upward.

Because my opening sits directly against a wall, I designed custom hinges that mount on top of the frame instead of inside it. I've attached the STL files in case someone has a similar setup.

## Servo Installation

Mount the servo on the inside of the cabinet where it can move freely.

I designed a printable servo bracket that screws to the cabinet and keeps everything aligned. The linkage arm snaps onto the servo horn and connects to the door.

The little wheel at the end of the arm rolls against the door, which greatly reduces friction and makes opening much smoother. Glue the small retaining cap onto the wheel so it can't slide off.

See the photos for the exact positioning.

## ESPHome

Create a new ESPHome device in Home Assistant and upload the following configuration:

```yaml
esphome:
  name: esphome-web-332f40
  friendly_name: Robo Door
  min_version: 2025.0.0
  name_add_mac_suffix: false

esp32:
  board: esp32-c6-devkitc-1
  variant: esp32c6
  framework:
    type: esp-idf

logger:
api:
ota:
  - platform: esphome

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

# --- Servo ---
servo:
  - id: door_servo
    output: pwm_output
    auto_detach_time: 3s        # PWM stops 3s after movement
    transition_length: 1200ms
    min_level: 3%
    max_level: 12%

output:
  - platform: ledc
    id: pwm_output
    pin: GPIO23
    frequency: 50 Hz

# --- Buttons in HA ---
button:
  - platform: template
    name: "Door open"
    on_press:
      - servo.write:
          id: door_servo
          level: -55%

  - platform: template
    name: "Door close"
    on_press:
      - servo.write:
          id: door_servo
          level: 63%

# --- Manual calibration ---
number:
  - platform: template
    name: "Door manual"
    min_value: -100
    max_value: 100
    step: 5
    optimistic: true
    set_action:
      - servo.write:
          id: door_servo
          level: !lambda 'return x / 100.0;'

binary_sensor:
  - platform: gpio
    name: "Robot motion"
    id: robot_motion
    pin:
      number: GPIO2          
      mode:
        input: true
    device_class: motion
    filters:
      - delayed_off: 2s
    on_press:
      - servo.write:
          id: door_servo
          level: -55%
```

The configuration creates:

- Open button
- Close button
- Manual position slider (useful for calibration)
- Motion sensor
- Servo control

The PIR sensor immediately opens the door when it detects the robot moving. This is necessary because the Roborock API can take up to 30 seconds to report that cleaning has started, which would otherwise cause the robot to drive into the closed door.

## Wiring

The wiring is very straightforward.

The 5V adapter powers both the ESP32 and the servo. The servo and PIR sensor share the same 5V and GND connections, while each uses its own GPIO pin for the signal.

I've attached a photo of my wiring. It's definitely functional, although I'll probably design a proper enclosure for the electronics later.

## Counterweight

One problem I ran into was that the servo wasn't strong enough to reliably hold the door open once power was removed. I could have kept the servo energized continuously, but I'd rather avoid that.

Instead, I added a simple counterweight to the inside of the door. Now the door is almost perfectly balanced, with the outside still being slightly heavier so it naturally closes once the servo lowers the arm.

This also means ESPHome can safely detach the servo after opening (`auto_detach_time`), so the servo isn't constantly drawing power.

## Home Assistant Automations

Finally, I added a few Home Assistant automations using the Roborock integration.

The motion sensor opens the door immediately. The Roborock state is then used to determine when it's safe to close it again. For example, I close the door after the robot has finished drying its mop:

Close door after mop drying automation

## That's it!

If anything is unclear, or if you'd like the Shapr3D files, or more details about the ESPHome configuration, just let me know. I'm happy to share everything.