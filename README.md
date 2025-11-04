

```markdown
# 🤖 Mechatronics Line Follower Robot

This repository contains the complete **Arduino source code** and **desktop GUI** for an advanced line-following robot, developed as part of the **Mechatronics Engineering program at Curtin University**.

The project's core innovation lies in its **custom-designed PCB** integrating an **Arduino Nano** with a unique **hardware-based PWM generation system**.  
Instead of relying on software PWM, this system uses **DACs**, **555 Timers**, and **Comparators** to generate a stable, high-frequency motor control signal.  
A **non-blocking, state-driven Bang-Bang algorithm** with sensor smoothing ensures responsive and stable motion.

---

## ⚙️ Hardware Architecture

A key design choice was to **offload PWM generation** from the microcontroller, ensuring clean and stable signals independent of the main code loop.

### 🔄 Signal Flow (per motor)

1. **Arduino Nano (Controller):**  
   Determines the desired speed as an 8-bit digital value (0–255).  
   [cite: 147]

2. **DAC0800 (Digital-to-Analog Converter):**  
   Converts the digital value into a proportional DC control voltage (0–5V).  
   [cite: 148, 149]

3. **555 Timer (Wave Generator):**  
   Generates a fixed-frequency triangle wave.  
   [cite: 151]

4. **LM741 Comparator (PWM Creation):**  
   Compares the triangle wave and DC control voltage to output a PWM signal with a duty cycle proportional to the voltage.  
   [cite: 153–156]

5. **Push-Pull Amplifier (Motor Driver):**  
   Amplifies the low-current PWM signal to drive the motors.  
   [cite: 158–160]

---

### 🔧 Key Components

| Component | Description |
|------------|-------------|
| **MCU** | Arduino Nano [cite: 10] |
| **Sensors** | 2× IR Sensors (Emitter/Receiver) [cite: 20, 21] |
| **Power** | 4× 3.7 V Li-ion Batteries + Buck-Boost to 15 V [cite: 16, 17] |
| **Drivetrain** | 2× DC Motors with Track Wheels [cite: 12] |

#### Signal Chain (per motor)
- 1× DAC0800 [cite: 41, 42]  
- 1× 555 Timer [cite: 45, 55]  
- 1× LM741 Op-Amp [cite: 43, 44]  
- 1× Comparator [cite: 46, 52]  
- 1× Push-Pull Amplifier [cite: 47, 48]

---

## 💻 Software & Features

The project is divided into two main parts:
1. **Arduino Firmware (`.ino`)**
2. **Desktop GUI**

---

### 🧠 Arduino Firmware

Built as a **non-blocking, modular state machine**. [cite: 207, 208]

#### Main Functions

- **`loop()`** – Orchestrates all subsystems, calling:
  - `handleSerialCommands()`
  - `sendDataToGUI()`
  - Executes control logic based on `carState` (`STATE_STOPPED`, `STATE_RUNNING`, `STATE_REVERSING`).  
  [cite: 866–870, 882]

- **`bangBangLineFollow()`** – Core control algorithm for forward motion.  
  Evaluates sensor states (Both White, Left White, Right White, Both Black) to adjust motor speeds.  
  [cite: 239–242, 663–690]

- **`readSensorsSmooth()`** – Smooths sensor readings using an average of the last `SMOOTH_SAMPLES` (e.g., 2) values, filtering electrical noise.  
  [cite: 171, 433, 574]

- **`outputToDAC1()` / `outputToDAC2()`** – Writes an 8-bit value (0–255) to digital pins connected to the DAC0800.  
  [cite: 434–435, 577–591]

- **`handleSerialCommands()` / `sendDataToGUI()`** –  
  Manages bidirectional communication with the GUI.  
  [cite: 326, 329]

---

### 🖥️ Desktop GUI

Provides real-time control, tuning, and monitoring. [cite: 927]

#### Features

- **Primary Robot Control:** Start, Stop, Forward/Reverse modes. [cite: 946–947]  
- **Live Parameter Tuning:** Adjust Fast and Slow PWM speeds in real-time. [cite: 950]  
- **Real-time Monitoring:** Displays live sensor and motor values. [cite: 953]  
- **Data Visualization:** Line chart for historical PWM data. [cite: 958]  
- **Experimental AI Mode:** Prototype Gemini AI control (non-functional). [cite: 960, 969]

---

## 🧭 Control Algorithm — Bang-Bang Strategy

A **Bang-Bang** control strategy offers fast, simple, and low-cost control. [cite: 119, 167, 169]

### Logic Flow

| Sensor Condition | Action | Details |
|------------------|--------|----------|
| **Both White (On Line)** | Go straight | `motorFast` speed [cite: 690–695] |
| **Left White, Right Black (Veering Right)** | Gentle left turn | Left motor = `motorSlow`, Right = `motorFast` [cite: 677–682] |
| **Right White, Left Black (Veering Left)** | Gentle right turn | Left motor = `motorFast`, Right = `motorSlow` [cite: 683–689] |
| **Both Black (Off Line)** | Recovery turn | Uses `lastLineSide` to decide direction [cite: 663–675] |

---

## 📡 Serial Communication Protocol

Custom packet-based protocol for PC–Arduino communication.

### 🖥️ GUI → Arduino (4 Bytes)
```

[START_BYTE] [COMMAND_BYTE] [DATA_BYTE] [CHECKSUM]

```
- **START_BYTE:** `255` [cite: 529, 930]  
- **CHECKSUM:** `(START_BYTE + COMMAND_BYTE + DATA_BYTE)` [cite: 327, 931]

#### Commands
| Command | Action |
|----------|--------|
| `START_CMD (4)` | Run [cite: 799] |
| `STOP_CMD (5)` | Stop [cite: 804] |
| `REVERSE_CMD (6)` | Reverse [cite: 808] |
| `SET_FAST_PWM (7)` | Update Fast PWM [cite: 812] |
| `SET_SLOW_PWM (8)` | Update Slow PWM [cite: 820] |
| `AI_MODE_CMD (9)` | Enable AI mode [cite: 826] |
| `MANUAL_MODE_CMD (10)` | Disable AI mode [cite: 829] |
| `AI_MOTOR_LEFT_CMD (11)` | Set left motor speed [cite: 832] |
| `AI_MOTOR_RIGHT_CMD (12)` | Set right motor speed [cite: 837] |

---

### 🔄 Arduino → GUI (6 Bytes)
```

[START] [LeftSensor] [RightSensor] [LeftMotor] [RightMotor] [CHECKSUM]

```
- **START_BYTE:** `255` [cite: 932]  
- **CHECKSUM:** `(START + LeftSensor + RightSensor + LeftMotor + RightMotor)` [cite: 933]  
- **Sensor/Motor Data:** 8-bit values (10-bit sensors are right-shifted `>>2`) [cite: 763–764, 932]

---

## ⚠️ Project Status & Known Issues

| Feature | Status | Notes |
|----------|---------|-------|
| **Line-following (Forward)** | ✅ Functional | PWM & sensor smoothing stable [cite: 198–204] |
| **Mechanical Alignment** | ⚠️ Issue | Uneven track tension causes motor asymmetry and instability [cite: 78–84, 202–204] |
| **Reverse Mode** | ❌ Non-functional | Sensors only front-facing; high latency in reverse [cite: 997] |
| **AI Mode** | ❌ Non-functional | Gemini AI integration incomplete due to API key issue [cite: 969, 997] |

---

**Author:** *Yasiru Fernando*  
**Institution:** Curtin University – Mechatronics Engineering  
**License:** MIT  
**Status:** Academic Project (Prototype)
```

---
