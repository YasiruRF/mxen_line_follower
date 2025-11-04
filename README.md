# Mechatronics Line Follower Robot

This repository contains the complete Arduino source code and desktop GUI for an advanced line-following robot, developed as part of the Mechatronics Engineering program at Curtin University.

The project's core technical feature is its custom-designed PCB that integrates an Arduino Nano with a unique **hardware-based PWM generation system**. Instead of relying on software PWM, this design uses DACs, 555 Timers, and Comparators to generate a stable, high-frequency motor control signal. The control logic is a state-driven, non-blocking "Bang-Bang" algorithm with sensor smoothing for a responsive and stable drive.

## ⚙️ Hardware Architecture

A key design decision was to offload PWM generation from the microcontroller, ensuring a clean and stable signal independent of the main code loop.

The signal flow for each motor is as follows:
1.  [cite_start]**Arduino Nano (Controller):** The software's control logic determines a desired speed as an 8-bit digital value (0-255)[cite: 147].
2.  [cite_start]**DAC0800 (Digital-to-Analog Converter):** This 8-bit value is sent to a DAC, which translates it into a stable, proportional DC control voltage (e.g., 0-5V)[cite: 148, 149].
3.  [cite_start]**555 Timer (Wave Generator):** A separate 555 Timer IC generates a continuous, fixed-frequency triangle wave[cite: 151].
4.  [cite_start]**LM741 Comparator (PWM Creation):** The comparator receives two inputs: the DC control voltage from the DAC and the triangle wave from the 555 Timer[cite: 153, 154, 155]. [cite_start]By comparing them, it outputs a square wave (PWM signal) whose duty cycle is directly proportional to the DAC's voltage[cite: 153, 156].
5.  [cite_start]**Push-Pull Amplifier (Motor Driver):** The low-current PWM signal from the comparator is fed into a push-pull amplifier (using NPN/PNP transistors) to provide the high current needed to drive the DC motors[cite: 158, 159, 160].

**Key Components:**
* [cite_start]**MCU:** Arduino Nano [cite: 10]
* [cite_start]**Sensors:** 2x IR Sensors (Emitter/Receiver) [cite: 20, 21]
* [cite_start]**Power:** 4x 3.7V Lithium-ion Battery Pack with a Buck-Boost Converter to regulate a stable 15V [cite: 16, 17]
* [cite_start]**Drivetrain:** 2x DC Motors with Track Wheels [cite: 12]
* **Signal Chain (per motor):**
    * [cite_start]1x DAC0800 [cite: 41, 42]
    * [cite_start]1x 555 Timer [cite: 45, 55]
    * [cite_start]1x LM741 OpAmp [cite: 43, 44]
    * [cite_start]1x Comparator [cite: 46, 52]
    * [cite_start]1x Push-Pull Amplifier [cite: 47, 48]

## 💻 Software & Features

The project is split into two main parts: the Arduino firmware and the desktop GUI.

### Arduino Firmware (`.ino`)

[cite_start]The Arduino code is built as a non-blocking, modular state machine[cite: 207, 208].

* [cite_start]**`loop()`:** The main loop acts as a central orchestrator[cite: 211]. [cite_start]It continuously calls `handleSerialCommands()` and `sendDataToGUI()`, and then executes motor logic based on the current `carState` (`STATE_STOPPED`, `STATE_RUNNING`, `STATE_REVERSING`) [cite: 866-870, 882].
* [cite_start]**`bangBangLineFollow()`:** The primary control algorithm for forward motion[cite: 237]. [cite_start]It evaluates four sensor states (Both White, Left White, Right White, Both Black) to determine motor speeds [cite: 239, 240, 241, 242, 663-690].
* [cite_start]**`readSensorsSmooth()`:** A smoothing function that averages the last `SMOOTH_SAMPLES` (e.g., 2) sensor readings[cite: 171, 433, 574]. [cite_start]This filters out electrical noise and "jitter" from minor track imperfections, leading to a more stable drive[cite: 172, 185].
* [cite_start]**`outputToDAC1()` / `outputToDAC2()`:** Hardware abstraction functions that take a single byte (0-255) and set the 8 corresponding digital pins for the external DAC0800 hardware [cite: 434, 435, 577-591].
* [cite_start]**`handleSerialCommands()` / `sendDataToGUI()`:** Manages the serial communication protocol with the GUI for receiving commands and sending status updates[cite: 326, 329].

### Desktop GUI

[cite_start]The GUI provides full control, tuning, and monitoring of the robot in real-time[cite: 927].

* [cite_start]**Primary Robot Control:** Start, Stop, and Forward/Reverse mode selection[cite: 946, 947].
* [cite_start]**Live Parameter Tuning:** "Fast PWM" and "Slow PWM" sliders allow for real-time adjustment of the robot's full speed and turning speed[cite: 176, 950].
* [cite_start]**Real-time Monitoring:** Displays live numerical readouts for both IR sensors and both motors[cite: 953].
* [cite_start]**Data Visualization:** A live-plotting line chart shows the historical PWM values for the left and right motors[cite: 958].
* [cite_start]**Experimental AI Control:** Includes a non-functional prototype feature for testing Gemini AI-based control[cite: 960, 969].

## 📡 Control Algorithm (Bang-Bang)

[cite_start]The vehicle uses a "Bang-Bang" control strategy, which is simple, fast, and computationally inexpensive[cite: 119, 167, 169].

The logic, as seen in `bangBangLineFollow()`, is as follows:
1.  **Case 1: Both sensors on white (On Line)**
    * [cite_start]Action: Go straight at `motorFast` speed [cite: 690-695].
2.  **Case 2: Left sensor on white, Right on black (Veering Right)**
    * Action: Execute a gentle left turn. [cite_start]Set left motor to `motorSlow` and right motor to `motorFast` [cite: 677-682].
3.  **Case 3: Right sensor on white, Left on black (Veering Left)**
    * Action: Execute a gentle right turn. [cite_start]Set left motor to `motorFast` and right motor to `motorSlow` [cite: 683-689].
4.  **Case 4: Both sensors on black (Off Line)**
    * [cite_start]Action: Check the `lastLineSide` memory variable (which stores the last known line position)[cite: 122, 242, 663].
    * [cite_start]Execute a sharp recovery turn in that direction (e.g., stop one motor and run the other at `motorFast`) [cite: 123, 663-675].

## 📦 Serial Communication Protocol

Communication between the GUI and Arduino is handled via a custom 4-byte (PC to Arduino) and 6-byte (Arduino to PC) packet structure.

### GUI to Arduino (4-Byte Packet)

`[START_BYTE] [COMMAND_BYTE] [DATA_BYTE] [CHECKSUM]`
* [cite_start]**`START_BYTE`**: `255` [cite: 529, 930]
* [cite_start]**`CHECKSUM`**: `(byte)(START_BYTE + COMMAND_BYTE + DATA_BYTE)` [cite: 327, 792, 931]
* **Commands:**
    * [cite_start]`START_CMD (4)`: Sets `carState = STATE_RUNNING` [cite: 555, 799]
    * [cite_start]`STOP_CMD (5)`: Sets `carState = STATE_STOPPED` [cite: 556, 804]
    * [cite_start]`REVERSE_CMD (6)`: Sets `carState = STATE_REVERSING` [cite: 557, 808]
    * [cite_start]`SET_FAST_PWM (7)`: Updates `fastPWM` with `DATA_BYTE` [cite: 558, 812]
    * [cite_start]`SET_SLOW_PWM (8)`: Updates `slowPWM` with `DATA_BYTE` [cite: 559, 820]
    * [cite_start]`AI_MODE_CMD (9)`: Sets `isAIMode = true` [cite: 560, 826]
    * [cite_start]`MANUAL_MODE_CMD (10)`: Sets `isAIMode = false` [cite: 561, 829]
    * [cite_start]`AI_MOTOR_LEFT_CMD (11)`: Sets `aiLeftMotor` speed [cite: 562, 832]
    * [cite_start]`AI_MOTOR_RIGHT_CMD (12)`: Sets `aiRightMotor` speed [cite: 563, 837]

### Arduino to GUI (6-Byte Packet)

`[START] [LeftSensor] [RightSensor] [LeftMotor] [RightMotor] [CHECKSUM]`
* [cite_start]**`START_BYTE`**: `255` [cite: 932]
* [cite_start]**Sensor/Motor Data**: 8-bit values (10-bit sensor values are right-shifted `>> 2`) [cite: 763, 764, 932]
* [cite_start]**`CHECKSUM`**: `(byte)(START + LeftSensor + RightSensor + LeftMotor + RightMotor)` [cite: 766, 933]

## ⚠️ Project Status & Known Issues

* **Core Functionality:** The forward-facing Bang-Bang line-following (`bangBangLineFollow()`) is **operational**. [cite_start]The hardware PWM generation and sensor smoothing work as intended[cite: 198, 199, 204].
* [cite_start]**Known Issue (Mechanical):** The vehicle's performance was significantly compromised by a mechanical flaw: mismatched tightness in the wheel tracks[cite: 78, 82, 202]. [cite_start]This created unequal frictional loads, causing motor asymmetry (unequal speeds for equal commands) and forcing the control algorithm to constantly oscillate, which limited the vehicle's maximum stable speed[cite: 83, 84, 203, 204].
* [cite_start]**Non-Functional Feature (Reverse):** The `bangBangLineFollowReverse()` function is **non-functional**[cite: 238]. [cite_start]Because the IR sensors are mounted at the front of the chassis, the control latency when moving in reverse was too high, causing the vehicle to leave the track[cite: 997].
* **Non-Functional Feature (AI Mode):** The experimental Gemini AI control mode is **non-functional**. [cite_start]The project was not completed in time for the final competition due to an issue with parsing the API keys[cite: 969, 997].
