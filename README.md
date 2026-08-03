# Pet Robot

A robot that finds a dog with a camera and drives toward it.

A neighbor's dog lost its companion, so this is an attempt to give it something to chase. A Raspberry Pi 5 runs object detection on a camera feed and decides which way to go. An Arduino Mega drives the wheels and decides whether that direction is safe. Ultrasonic sensors stop it before it hits anything.

Built by [Yunus Shavkatov](https://Yu5h2k4.github.io), starting April 2026.

## How it works

The core design rule, and the thing worth understanding before reading any of the code:

**The Pi owns intent. The Mega owns safety.**

The Pi decides where the dog is and what direction to travel. It has no authority to move the robot. It sends a request, and the Mega decides whether to honor it.

Commands arriving over serial are handled by exactly the same code path as button presses from the IR remote. That means `obstacleGuard()` overrides them at 25 cm no matter what the camera thinks it sees, and the existing 200 ms auto-release doubles as a deadman switch: if the Pi crashes or the USB cable pops out, the robot stops by itself within a fifth of a second.

Pressing `0` on the remote toggles vision mode off, and any direction button also drops out of it. That is the kill switch, and it exists because this robot is meant to operate near a live animal.

```
camera -> YOLOv8n (NCNN) -> largest dog box -> direction + deadzone
                                                    |
                                          USB serial, one char, ~20/sec
                                                    |
                                    Mega: obstacleGuard() + 200ms deadman
                                                    |
                                              ESC -> motors
```

## Hardware

| Part | Role |
| --- | --- |
| Raspberry Pi 5, 4 GB | Vision and decision making |
| Elegoo Mega 2560 | Motor control, sensors, safety overrides |
| goBilda 2x40A ESC | Motor driver. RC-style, takes servo PWM, not serial or DIP |
| 2x REV Core Hex motor | Drive, one per side. Tank / skid steer |
| 1x HD Hex motor + omni wheel | Mounted perpendicular, assists pivoting |
| 3x HC-SR04 | Distance. Front plus one at 90 degrees on each side |
| Camera module | CSI, not USB |
| 12 V NiMH pack | Motors only |
| USB-C PD power bank, 30 W+ | Pi only, deliberately a separate power domain |

### Wiring

ESC signal on D9. Motor power runs XT30 to an XT30-to-XT90 adapter into the ESC's XT90 socket.

Ultrasonic sensors, all sharing the breadboard 5 V and GND rails:

| Sensor | Trig | Echo |
| --- | --- | --- |
| Front | D6 | D7 |
| Left | D4 | D5 |
| Right | D2 | D3 |

No level shifting needed on the sensors, since the Mega is a 5 V board. D22 to D25 are reserved for a possible second pair.

Pi to Mega is **USB serial at 115200**, into the Mega's USB-B port. This was chosen over the GPIO UART deliberately: the Mega's 5 V TX line would need a voltage divider before it could safely touch the Pi's 3.3 V GPIO, and USB avoids that entirely. It also powers the Mega as a side effect.

### Why the Pi has its own battery

The Pi does not run off the 12 V motor pack, and this is not an oversight. Motors are inductive loads whose current spikes sag the pack for milliseconds at a time. That is the exact undervoltage condition that froze the Pi during early testing on an underpowered charger, except on the robot it would fire every time the wheels started moving. Separate power domains remove the failure by construction.

A 5 V/5 A UBEC with an input capacitor tapped at the battery terminals is the more elegant version and is planned, but it needs bench time.

## Repository layout

```
firmware/
  motor_test/motor_test.ino    Single-motor bring-up sketch
  tank_drive/tank_drive.ino    Drive logic, sensors, obstacle guard, serial command handling
  vision_test/                 YOLO test scripts and dog_chase.py
```

`tank_drive.ino` has the on-blocks diagnosis procedure written into its file header. Read that before touching the wiring.

## Running it

On the Pi:

```bash
cd firmware/vision_test
python3 dog_chase.py
```

The script picks the largest detected dog box, converts its center into a left, right, forward or stop decision with a deadzone, and stops when the box fills half the frame. It echoes the Mega's own telemetry back to the terminal, prefixed `[mega]`, because the Arduino Serial Monitor cannot share `/dev/ttyACM0` with the script.

Upload `tank_drive.ino` to the Mega first, using the Arduino IDE. Note that only one process can own the serial port, so close the Serial Monitor before starting the Python script.

### Test on blocks first

Put the robot up on blocks before any run that follows a wiring change. This is not general caution, it is a lesson from a specific failure documented below.

## Tuning

| Constant | Where | Notes |
| --- | --- | --- |
| `DEADZONE` | `dog_chase.py` | Raise it if the robot oscillates around center |
| `PULSE_STOP_RIGHT` / `PULSE_STOP_LEFT` | `tank_drive.ino` | Per-channel neutral trim, for ESC drift, so "stopped" does not creep |
| `PIVOT_90_MS` | `tank_drive.ino` | Currently 600, still needs measuring on the real robot |
| `SWAP_CHANNELS`, `RIGHT_REVERSED`, `LEFT_REVERSED` | `tank_drive.ino` | All false. The physical wiring is correct and should stay that way |

Inference resolution is set to 416 px. The tradeoff is real and was measured: 320 px gives 40-50 fps but misses distant dogs, and 640 px collapses to 8-11 fps and detects *worse*, because at 9 fps a running dog moves too far between inferences. 416 lands around 20-25 fps.

## Things that broke, and what fixed them

Kept here because they are the useful part of the repo.

**The robot drove backward.** Forward commands sent it in reverse while turns still went the right way. That combination is diagnostic: two separate faults, channels inverted *and* channels swapped, cancelled each other during turns and only appeared in straight-line motion. Fixed in hardware by swapping the motor leads back, not in software, so the harness matches what the firmware header documents. Lesson: any rewiring invalidates motion you already verified.

**A debug message killed the control loop.** `dog_chase.py` died on `UnicodeEncodeError` mid-run. The Mega's status strings contained a `→`, Python decoded it correctly, and then `print()` failed because the Pi's terminal locale is latin-1. It looked intermittent but was not. The arrow only appeared in the obstacle message, so it fired the first time the robot came within 25 cm of anything. Fixed on both sides: the Mega now uses ASCII `->`, and the Pi decodes with `errors="backslashreplace"`. The Python fix is the load-bearing one, since electrical noise from the motors could have crashed the loop the same way. Principle: a diagnostic must never be able to kill the thing it is diagnosing.

**A no-echo reading sent the robot the wrong way.** An HC-SR04 that hears nothing back returns -1, which sorts as the *least* open direction. The avoid-and-turn logic would have steered into a wall whenever a sensor saw nothing at all. Caught in review before upload, fixed with a `clearance()` helper that maps -1 to maximum clearance.

**A $70 part that a software change made unnecessary.** Detection ran at 8-12 fps on the Pi's CPU, which is not fast enough to track a moving dog, and the plan was to buy a Hailo AI accelerator. Exporting the model to NCNN instead took the same hardware to 40-50 fps. The purchase was cancelled.

## Status

Working: drive under code, IR remote control, front and side distance sensors, obstacle stop at 25 cm, YOLO detection, detection driving the wheels. The full chain from camera to wheels has been verified on the floor, with the robot turning to face a person and centering on them under its own control.

Not yet bench tested: the `autoDrive()` obstacle-avoidance state machine on remote button `1`.

## Next

- A control page served from the Pi to any phone browser, showing the live camera feed with detection boxes while you drive. The Pi already has WiFi, so this needs no extra hardware. It has to run as a thread inside `dog_chase.py`, since only one process can own the serial port.
- A shell over the frame.
- Running the Pi off the main battery through a UBEC instead of a separate power bank.
