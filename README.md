# servo-hand-esp

MicroPython firmware for an ESP32-S2 robotic hand controller. Receives finger-position data over ESP-NOW or UDP from sensor gloves, PC ML inference, or other devices and drives 5 servo motors via a PCA9685 PWM board.

Part of the **OpenMuscle / OpenHand** project.

## Hardware

- ESP32-S2 microcontroller
- PCA9685 16-channel PWM servo driver (I2C, address 0x40)
- SSD1306 OLED display, 128x32 pixels (I2C)
- 5 servos on PCA9685 odd channels (1, 3, 5, 7, 9)
- 4 navigation buttons (Start, Select, Up, Down)
- Status LED

| Function | GPIO |
|----------|------|
| LED | 15 |
| I2C SCL | 33 |
| I2C SDA | 34 |
| Start button | 7 |
| Select button | 8 |
| Up button | 9 |
| Down button | 10 |

## Files

| File | Description |
|------|-------------|
| `micropython/boot.py` | Main firmware (V2). Runs on boot. |
| `micropython/pca9685.py` | PCA9685 I2C driver (Kevin McAleer) |
| `micropython/servo.py` | Servo abstraction on top of PCA9685 |
| `micropython/lask4.py` | Legacy sensor-glove firmware (reference) |
| `micropython/lask-boot.py` | Earlier sensor-glove firmware (reference) |

## Deployment

Upload files to the ESP32 root filesystem using mpremote:

```
mpremote connect <PORT> cp micropython/boot.py micropython/pca9685.py micropython/servo.py :
```

Or use Thonny / ampy. `boot.py` runs automatically on power-on.

## OLED Menu

On boot the display shows a 4-item menu navigated with UP/DOWN buttons. START selects an item, SELECT exits back to the menu from receive modes.

| Item | Action |
|------|--------|
| ESP-NOW Listen | Receive packets over ESP-NOW (broadcast) |
| UDP Listen | Connect to WiFi, receive packets on UDP port 3145 |
| Servo Test | Sweep all 5 fingers through 30, 90, 140, 90 degrees |
| Release All | Turn off PWM on all 16 channels |

## Packet Format

Compact comma-separated text. ~20 bytes for 5 fingers.

With a device ID prefix (first field non-numeric):
```
L5,400,300,500,200,100
PC,120,90,45,160,30
```

Bare values (all numeric, uses `default` device config):
```
400,300,500,200,100
```

Senders just need:
```python
','.join(['L5'] + [str(x) for x in data])
```

## Device Configuration

The `DEVICES` dict at the top of `boot.py` maps device IDs to their behavior:

```python
DEVICES = {
    'default': {'map': 'sigmoid', 'in_min': 0, 'in_max': 800, 'reverse': True},
    'L5':      {'map': 'sigmoid', 'in_min': 0, 'in_max': 800, 'reverse': True},
    'PC':      {'map': 'linear',  'in_min': 0, 'in_max': 179, 'reverse': False},
}
```

| Field | Description |
|-------|-------------|
| `map` | `'sigmoid'` (S-curve) or `'linear'` (direct scaling) |
| `in_min` / `in_max` | Expected input value range from the device |
| `reverse` | Flip finger order (`True` for gloves that send pinky-first) |

To add a new device, add one entry to this dict.

## Value Mapping

- **Sigmoid**: S-curve (k=10, midpoint 0.5) maps raw sensor values to 0-179 degrees. Good for analog sensors with non-linear response.
- **Linear**: Direct proportional mapping from input range to 0-179 degrees. Use for devices that send pre-computed angles.
