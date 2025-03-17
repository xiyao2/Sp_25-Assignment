# Sp_25-Assignment
# IMU RGB Plant Interaction Documentation

## Overview
This project uses an IMU sensor and RGB LEDs to simulate plant care interactions. The system detects specific hand gestures and motions to determine if a plant needs water, sunlight, or petting. Each action is acknowledged by LED color changes.

## Hardware Components
- **M5Stack**: Main controller
- **IMUProUnit**: For motion detection (connected via I2C)
- **NeoPixel LED Strip**: 30 RGB LEDs controlled by Pin 7
- **I2C Interface**: For sensor communication

## Initialization
1. **Hardware Initialization**
    - `M5.begin()`: Starts the M5 system.
    - `I2C(0, scl=Pin(1), sda=Pin(2), freq=100000)`: Sets up the I2C communication.
    - `IMUProUnit(i2c)`: Initializes the IMU sensor.
    - `NeoPixel(Pin(7), 30)`: Initializes the LED strip with 30 LEDs.

2. **State Initialization**
    - The plant has three interaction states: `needs_water`, `needs_sun`, and `wants_petting`.
    - Initial state set to 0 with `state_completed` set to `False`.
    - Sensitivity thresholds are set for shake, tilt, and raise motions.

## Main Loop Functionality
### 1. LED Flash Reminder
- LEDs flash in specific colors to indicate the plant's need:
    - **Red** for `needs_water`.
    - **Yellow** for `needs_sun`.
    - **Blue** for `wants_petting`.
- LED flashes every 500ms using the `flash_interval` timer.

### 2. IMU Data Processing
- IMU values are read every 100ms.
- Actions are detected as follows:
    - **Watering (Tilt Movement)**:
        - Detect significant tilt in the x-axis.
        - LED turns green and prints "watered".
    - **Sun Exposure (Raise Hand for 5 Seconds)**:
        - Detect if the y-axis value exceeds the `raise_threshold`.
        - Requires holding for at least 5 seconds.
        - LED turns green and prints "sunned".
    - **Petting (Shake Movement)**:
        - Detect rapid shake in the z-axis.
        - LED turns green and prints "petted".

<img width="848" alt="截屏2025-03-16 下午10 05 20" src="https://github.com/user-attachments/assets/ddf5c415-6194-4e7d-bdf4-eec10e257c56" />


### 3. State Transition
- After successful action, the green LED stays on for 2 seconds.
- The system then transitions to the next state and prints the current state.

## LED Control Function
```python
def set_led_color(r, g, b):
    for i in range(30):
        np[i] = (r, g, b)
    np.write()
```
- Controls the color of the entire LED strip.

## Thresholds and Timers
- **Shake Threshold**: 0.7
- **Tilt Threshold**: 0.5
- **Raise Threshold**: 0.4
- **LED Flash Interval**: 500ms
- **Action Hold Duration**: 2000ms
- **Sun Hold Duration**: 3000ms
  
  <img width="865" alt="11491742187020_ pic" src="https://github.com/user-attachments/assets/3f2251f1-030d-42a3-84fb-e7a11da9eeac" />


## Code Example
```python
import os, sys, io
import M5
from M5 import *
from hardware import I2C
from hardware import Pin, ADC
from unit import IMUProUnit
from time import *
from neopixel import NeoPixel
import m5utils  # remap function

# -------------------- 初始化硬件 -------------------- #
M5.begin()

# I2C 初始化
i2c = I2C(0, scl=Pin(1), sda=Pin(2), freq=100000)

# 初始化 IMU 传感器
imu = IMUProUnit(i2c)

# 初始化 RGB LED
np = NeoPixel(Pin(7), 30)

# -------------------- 状态设置 -------------------- #
plant_states = ['needs_water', 'needs_sun', 'wants_petting']
current_state = 0
state_completed = False

# IMU 记录
imu_x_last, imu_y_last, imu_z_last = 0, 0, 0

# 定时器
imu_timer = 0
flash_timer = 0
action_hold_timer = 0
sun_hold_timer = 0
action_hold_duration = 2000  # 2秒绿灯保持
sun_hold_duration = 3000     # 抬手持续5秒
sun_detected = False

# 震动检测灵敏度
shake_threshold = 0.7  
tilt_threshold = 0.5   
raise_threshold = 0.4  

# LED 控制
led_flash_state = False
flash_interval = 500  # 每500ms闪烁一次

# 设置LED颜色
def set_led_color(r, g, b):
    for i in range(30):
        np[i] = (r, g, b)
    np.write()

# -------------------- 主循环 -------------------- #
while True:
    M5.update()

    # -------------------- LED 闪烁提醒 -------------------- #
    if ticks_ms() > flash_timer + flash_interval and not state_completed:
        flash_timer = ticks_ms()
        led_flash_state = not led_flash_state
        
        if led_flash_state:
            if plant_states[current_state] == 'needs_water':
                set_led_color(255, 0, 0)  # 红色闪烁
            elif plant_states[current_state] == 'needs_sun':
                set_led_color(255, 255, 0)  # 黄色闪烁
            elif plant_states[current_state] == 'wants_petting':
                set_led_color(0, 0, 255)  # 蓝色闪烁
        else:
            set_led_color(0, 0, 0)  # 熄灭

    # -------------------- 读取 IMU -------------------- #
    if (ticks_ms() > imu_timer + 100):
        imu_timer = ticks_ms()
        imu_val = imu.get_accelerometer()
        imu_x, imu_y, imu_z = imu_val[0], imu_val[1], imu_val[2]

        # -------------------- 动作识别 -------------------- #
        if not state_completed:
            if plant_states[current_state] == 'needs_water' and abs(imu_x - imu_x_last) > tilt_threshold:
                set_led_color(0, 255, 0)  # 变绿灯
                print("watered")
                state_completed = True
                action_hold_timer = ticks_ms()
            elif plant_states[current_state] == 'needs_sun':
                if imu_y > raise_threshold and not sun_detected:
                    sun_detected = True
                    sun_hold_timer = ticks_ms()
                elif sun_detected and (ticks_ms() - sun_hold_timer >= sun_hold_duration):
                    set_led_color(0, 255, 0)
                    print("sunned")
                    state_completed = True
                    sun_detected = False
                    action_hold_timer = ticks_ms()
                elif imu_y <= raise_threshold:
                    sun_detected = False
            elif plant_states[current_state] == 'wants_petting':
                if abs(imu_z - imu_z_last) > shake_threshold:
                    set_led_color(0, 255, 0)
                    print("petted")
                    state_completed = True
                    action_hold_timer = ticks_ms()
        if state_completed and ticks_ms() - action_hold_timer > action_hold_duration:
            current_state = (current_state + 1) % len(plant_states)
            print(plant_states[current_state])
            state_completed = False
        imu_x_last, imu_y_last, imu_z_last = imu_x, imu_y, imu_z
    sleep_ms(10)
```



https://github.com/user-attachments/assets/165d4313-ff97-46c9-bac1-92fdd5e6b7ef




