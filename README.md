# Mechatronics Line Follower Robot

This repository contains the complete Arduino source code and desktop GUI for an advanced line-following robot, developed as part of the Mechatronics Engineering program at Curtin University.

The project's core technical feature is its custom-designed PCB that integrates an Arduino Nano with a unique **hardware-based PWM generation system**. Instead of relying on software PWM, this design uses DACs, 555 Timers, and Comparators to generate a stable, high-frequency motor control signal. The control logic is a state-driven, non-blocking "Bang-Bang" algorithm with sensor smoothing for a responsive and stable drive.

---

## Hardware Architecture

A key design decision was to offload PWM generation from the microcontroller, ensuring a clean and stable signal independent of the main code loop.

The signal flow for each motor is as follows:

1. **Arduino Nano (Controller):**
   The software’s control logic determines a desired speed as an 8-bit digital value (0–255).

2. **DAC0800 (Digital-to-Analog Converter):**
   This 8-bit value is sent to a DAC, which translates it into a stable, proportional DC control voltage (e.g., 0–5V).

3. **555 Timer (Wave Generator):**
   A separate 555 Timer IC generates a continuous, fixed-frequency triangle wave.

4. **LM741 Comparator (PWM Creation):**
   The comparator receives two inputs: the DC control voltage from the DAC and the triangle wave from the 555 Timer.
   By comparing them, it outputs a square wave (PWM signal) whose duty cycle is directly proportional to the DAC’s voltage.

5. **Push-Pull Amplifier (Motor Driver):**
   The low-current PWM signal from the comparator is fed into a push-pull amplifier (using NPN/PNP transistors) to provide the high current needed to drive the DC motors.

### Key Components

* **MCU:** Arduino Nano
* **Sensors:** 2x IR Sensors (Emitter/Receiver)
* **Power:** 4x 3.7V Lithium-ion Battery Pack with a Buck-Boost Converter for stable 15V
* **Drivetrain:** 2x DC Motors with Track Wheels

**Signal Chain (per motor):**

* 1x DAC0800
* 1x 555 Timer
* 1x LM741 OpAmp
* 1x Comparator
* 1x Push-Pull Amplifier

---

## Software & Features

The project is split into two main parts: the Arduino firmware and the desktop GUI.

### Arduino Firmware

The Arduino code is built as a non-blocking, modular state machine.

* **`loop()`:**
  Acts as the central orchestrator. Continuously calls `handleSerialCommands()` and `sendDataToGUI()`, then executes motor logic based on the current `carState` (`STATE_STOPPED`, `STATE_RUNNING`, `STATE_REVERSING`).

* **`bangBangLineFollow()`:**
  The primary control algorithm for forward motion. Evaluates four sensor states (Both White, Left White, Right White, Both Black) to determine motor speeds.

* **`readSensorsSmooth()`:**
  A smoothing function that averages the last few sensor readings to filter out electrical noise and minor track imperfections.

* **`outputToDAC1()` / `outputToDAC2()`:**
  Hardware abstraction functions that take an 8-bit value and set the 8 corresponding digital pins for the external DAC0800 hardware.

* **`handleSerialCommands()` / `sendDataToGUI()`:**
  Manage the serial communication protocol between the Arduino and GUI.

---

### Desktop GUI

The GUI provides full control, tuning, and monitoring of the robot in real time.

**Features:**

* **Primary Robot Control:**
  Start, Stop, and Forward/Reverse mode selection.

* **Live Parameter Tuning:**
  “Fast PWM” and “Slow PWM” sliders allow real-time adjustment of full and turning speeds.

* **Real-time Monitoring:**
  Displays live numerical readouts for both IR sensors and both motors.

* **Data Visualization:**
  A live-plotting line chart shows historical PWM values for both motors.

* **Experimental AI Control:**
  Includes a prototype Gemini AI-based control feature (non-functional).

---

## Control Algorithm (Bang-Bang)

The vehicle uses a "Bang-Bang" control strategy — simple, fast, and computationally efficient.

The logic in `bangBangLineFollow()` works as follows:

1. **Both sensors on white (On Line)**

   * Move straight at `motorFast` speed.

2. **Left sensor on white, Right on black (Veering Right)**

   * Gentle left turn: left motor `motorSlow`, right motor `motorFast`.

3. **Right sensor on white, Left on black (Veering Left)**

   * Gentle right turn: left motor `motorFast`, right motor `motorSlow`.

4. **Both sensors on black (Off Line)**

   * Use `lastLineSide` to recall the last known line position.
     Execute a sharp recovery turn by stopping one motor and running the other at `motorFast`.

---

## Serial Communication Protocol

Communication between the GUI and Arduino is handled via a custom 4-byte (PC to Arduino) and 6-byte (Arduino to PC) packet structure.

### GUI → Arduino (4-Byte Packet)

`[START_BYTE] [COMMAND_BYTE] [DATA_BYTE] [CHECKSUM]`

* **START_BYTE:** `255`
* **CHECKSUM:** `(byte)(START_BYTE + COMMAND_BYTE + DATA_BYTE)`

**Commands:**

* `START_CMD (4)`: Start robot (`STATE_RUNNING`)
* `STOP_CMD (5)`: Stop robot (`STATE_STOPPED`)
* `REVERSE_CMD (6)`: Reverse mode (`STATE_REVERSING`)
* `SET_FAST_PWM (7)`: Update `fastPWM`
* `SET_SLOW_PWM (8)`: Update `slowPWM`
* `AI_MODE_CMD (9)`: Enable AI mode
* `MANUAL_MODE_CMD (10)`: Disable AI mode
* `AI_MOTOR_LEFT_CMD (11)`: Set AI left motor speed
* `AI_MOTOR_RIGHT_CMD (12)`: Set AI right motor speed

---

### Arduino → GUI (6-Byte Packet)

`[START] [LeftSensor] [RightSensor] [LeftMotor] [RightMotor] [CHECKSUM]`

* **START_BYTE:** `255`
* **Sensor/Motor Data:** 8-bit values (10-bit sensor data shifted `>> 2`)
* **CHECKSUM:** `(byte)(START + LeftSensor + RightSensor + LeftMotor + RightMotor)`

---

## Project Status & Known Issues

* **Core Functionality:**
  The Bang-Bang line-following control is fully operational. Hardware PWM generation and sensor smoothing work as intended.

* **Mechanical Issue:**
  The robot’s performance was affected by mismatched wheel track tension. This caused unequal friction, resulting in speed asymmetry and oscillations at higher speeds.

* **Non-Functional Reverse Mode:**
  `bangBangLineFollowReverse()` is non-functional. The IR sensors, mounted at the front, caused excessive latency during reverse movement.

* **Non-Functional AI Mode:**
  The Gemini AI-based mode remains incomplete due to an API key parsing issue.

---

