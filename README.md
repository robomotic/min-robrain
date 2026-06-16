# Ideal Dev Kit for Physical Robotics on Low-Cost Servos

This document compiles key lessons learned from experimenting with hardware architectures and deploying Reinforcement Learning (RL) policies on physical robots. Building a robust, low-cost legged robot requires balancing strict pin requirements, power delivery limits, and microsecond-level control loop determinism. 

The Waveshare ESP32-S3 variants provide the ultra-compact footprint and computational headroom (via ample PSRAM) necessary to execute TFLite models while directly driving low-cost actuators without relying on bottlenecked external drivers.

## 1. Waveshare ESP32-S3-Zero (N8R8 Variant)

The most efficient option for minimizing robot footprint while maximizing pin availability. It is a tiny, castellated board roughly the size of a Seeed XIAO, with comprehensive pin access.

* **Pin Accessibility**: Breaks out 24 GPIO pins, all accessible via standard row pin headers. No hidden bottom pads.
* **TFLite / RL Suitability**: The N8R8 variant includes **8 MB of Flash and 8 MB of PSRAM**. The additional PSRAM provides necessary memory headroom for neural network layer allocation and heavy RL code pipelines.
* **Pin Budget**: Covers the 18-pin requirement with 6 spare pins available for future expansion (e.g., buzzers, ultrasonic sensors).

> ⚠️ **CRITICAL WARNING**: Do not purchase the *standard* ESP32-S3-Zero (which only has 4MB Flash and 2MB PSRAM). For TFLite and RL workloads, 2MB of PSRAM is a severe bottleneck and will cause out-of-memory crashes. You must specifically seek out the **N8R8** variant for its 8MB of PSRAM.

## 2. Waveshare ESP32-S3-Pico

An alternative featuring an elongated form factor similar to a Raspberry Pi Pico, combined with the processing power of the ESP32-S3.

* **Pin Accessibility**: Breaks out 27 × multi-function GPIO pins, offering ample physical layout space to organize signal wires cleanly.
* **Power Supply**: Features an onboard high-efficiency DC-DC buck-boost chip (MP28164) capable of handling a 2A load current. While servos should remain powered via a dedicated UBEC, this robust power rail effortlessly and stably powers the MPU6050, OLED display, and FSR sensors without logic-side voltage drops.
* **Technical Specifications**:
  * **Main Chip**: ESP32-S3R2
  * **Processor**: Xtensa® 32-bit LX7 dual-core processor (up to 240 MHz)
  * **Memory**: 16 MB Flash, 2 MB PSRAM, 512 KB SRAM
  * **Interfaces**: Full-speed USB OTG, SPI, I2C, UART, ADC, PWM, DVP, and LCD interfaces.
  * **Form Factor**: Castellated module allows for direct soldering and integration into custom baseboards if needed.

## Pin-Budget Breakdown (ESP32-S3-Zero N8R8)

Using the ESP32-S3-Zero (N8R8) as the layout base ensures a clean and uncrowded physical wiring configuration:

| Component | Connected Pins | Pin Count | Notes |
| --- | --- | --- | --- |
| 12x MG90S Servos | GPIO 1 through 12 | 12 | Standard digital PWM pins. |
| I2C Bus (IMU + Screen) | GPIO 13 (SDA), GPIO 14 (SCL) | 2 | Shared parallel data bus. |
| 4x FSR Foot Sensors | GPIO 15, 16, 17, 18 | 4 | Configured on native ADC channels. |
| Spare / Expansion | GPIO 38, 39, 40, 41, 42, 45 | 6 | Fully customizable. |

## MPU6050 IMU Wiring

The GY-521 MPU6050 breakout board communicates via the I2C protocol. Since the ESP32-S3 operates at 3.3V logic, connect the module directly to the 3.3V rail to avoid logic level conflicts.

| MPU6050 Pin | ESP32-S3 Connection | Notes |
| --- | --- | --- |
| **VCC** | 3.3V | Do not connect to 5V. The ESP32-S3 is a 3.3V device. |
| **GND** | GND | Common ground. |
| **SCL** | GPIO 14 | I2C Clock line. |
| **SDA** | GPIO 13 | I2C Data line. |

> **Note:** The OLED display can share this exact same I2C bus (GPIO 13/14) by connecting its SCL/SDA pins in parallel, as I2C supports multiple devices on the same bus with unique addresses.

## Direct Servo Control Implementation: Hardware Parallel vs. I2C Sequential

While the **PCA9685 16-Channel PWM Driver** ([AliExpress](https://www.aliexpress.com/item/1005012137990430.html)) is a popular and inexpensive alternative, it is **not recommended for this Reinforcement Learning application**. 

* **The PCA9685 Limitation**: It communicates over I2C. Updating 12 servos requires the microcontroller to send 12 individual I2C write commands sequentially. This introduces microsecond-level latency, cumulative jitter, and unnecessary CPU overhead. In a high-frequency RL control loop, this sequential bottleneck degrades synchronization and environment determinism.
* **The ESP32-S3 Advantage**: By connecting servos directly to the ESP32-S3's GPIO pins, you leverage its dedicated hardware peripheral: the **LEDC (LED Control)**. This generates true, hardware-level parallel PWM signals. Once configured, the LEDC updates all 12 channels simultaneously in the background with zero CPU intervention or I2C bus contention, ensuring deterministic, glitch-free, and perfectly synchronized actuator control.

**Recommended Direct Implementation**:
* **Arduino**: Use the `ESP32Servo` library.
* **MicroPython**: Use the native `machine.PWM` module.

## The Importance of a Dedicated Servo Breakout Board

When directly driving 12 servos from an ESP32-S3, managing the physical wiring is just as critical as the software. Soldering 12 signal wires and their corresponding grounds directly to a microcontroller is prone to cold joints, short circuits, and mechanical failure under vibration. 

**Recommended Component: Serial Wombat Servo Breakout Board** ([Amazon UK](https://www.amazon.co.uk/Servo-Breakout-Board-Serial-Wombat/dp/B0F835FT7X))

This board is explicitly designed to solve the "rats nest" wiring problem in multi-servo robotics:
* **Organized Signal Distribution**: It provides clean, numbered, screw-terminal or header-based breakout points for all 12 servo signal lines, keeping the ESP32-S3's GPIO pins organized and easily traceable.
* **Robust Power Routing**: It allows the main high-current 5V rail from the UBEC to be securely distributed to all servos without overloading the microcontroller's pins or relying on fragile jumper wires.
* **Maintenance & Debugging**: If a single servo fails or needs replacement, it can be disconnected and swapped in seconds without desoldering or disrupting the rest of the robot's wiring harness.

## Power Requirements & Current Calculation (12x MG90S Servos at 5.0V)

To ensure stable operation without voltage sag or triggering the BEC's over-current protection, the following current profile must be supported:

* **Stable / Continuous Current**: Under normal load (e.g., moving robot limbs), each MG90S draws approximately 400mA–500mA.
  * *Calculation*: 12 servos × 0.5A = **6.0A continuous**.
* **Peak / Stall Current**: If all servos hit a mechanical stop or maximum torque simultaneously, each can draw up to ~800mA–1.0A at 5.0V.
  * *Calculation*: 12 servos × 1.0A = **12.0A peak** (transient).

**BEC Selection Rationale**: 
A UBEC rated for **10A continuous and 15A–20A peak** (like the recommended Hobbywing or fixed 10A alternatives) provides the necessary headroom. It comfortably handles the 6.0A baseline load and absorbs the 12.0A transient stall spikes without dropping the 5.0V logic rail.

### Logic & Sensor Power Consumption (at 3.3V)

While the servos draw the most current, the logic side must also be accounted for, especially during TFLite inference:

* **ESP32-S3 (TFLite Inference)**: ~250mA – 350mA (dual-core @ 240MHz with vector instructions and PSRAM active).
* **0.96" OLED Display (SSD1306)**: ~25mA (all pixels white at max brightness; typical text usage is ~10mA).
* **MPU6050 IMU**: ~5mA (gyroscope and accelerometer active).
* **4x FSR Sensors**: ~1.5mA (assuming 10kΩ pull-down voltage dividers).
* **Total Logic Current**: ~350mA (rounded to **400mA** for safe margin).
* **Total Logic Power**: ~1.3W.

**Total System Draw (from 2S LiPo)**: 
The system will draw a baseline of ~6.4A from the battery during normal simultaneous servo movement and active AI inference, with transient spikes up to **12.5A** if all servos stall simultaneously. A high-quality 2S LiPo (e.g., 1000mAh, 25C discharge rate) can easily supply up to 25A continuously, providing a massive safety margin.

### Why Not a 1S LiPo + Boost Converter Setup?
While a single-cell (1S) LiPo is smaller and lighter, it is explicitly rejected for a 12-servo RL robot due to fundamental power delivery physics:

1. **The Input Current Multiplier**: Power is conserved (minus conversion losses). To deliver 5V at a 12.5A peak (62.5W) with 90% efficiency, a 1S battery (~3.7V nominal) must supply **~18.8A peak** from the pack. A 2S battery (~7.4V nominal) only needs to supply **~9.4A peak**. Half the current drastically reduces stress on the battery, wires, and connectors.
2. **Boost Converter Thermal & Efficiency Limits**: Cheap boost (step-up) modules typically max out at 3A–5A continuous and run extremely hot. High-current (>10A) boost converters are expensive, inefficient, and prone to thermal shutdown under the rapid, high-current spikes characteristic of legged robotics. Buck (step-down) UBECs are inherently more efficient, cooler, and highly reliable.
3. **Voltage Sag & Servo Performance**: Under heavy load, a 1S LiPo's voltage sags rapidly (e.g., dropping to 3.0V). A boost converter must work even harder to step up a sagging input, often triggering undervoltage lockouts. Conversely, a 2S setup natively supports 6.0V–7.4V servo operation (via an adjustable UBEC), yielding significantly faster speeds and higher torque for the MG90S without conversion penalties.

### The 6.0V Servo Trade-off: Performance vs. Risk
Running the MG90S servos at 6.0V (instead of the recommended 5.0V) yields ~20% more torque and ~15% faster speed, which is highly desirable for dynamic RL locomotion. However, it introduces severe risks that require specific hardware modifications:

1. **Exponential Current Spike**: At 6.0V, the stall current of a single MG90S jumps from ~1.0A to **~1.5A**. If all 12 servos stall simultaneously, the system peak demand hits **18.0A**. This pushes the absolute limits of a 10A continuous / 15A peak UBEC and risks triggering over-current shutdowns.
2. **Thermal Runaway**: The MG90S uses nylon plastic gears. Under continuous 6.0V load, they generate significant internal heat, rapidly degrading the plastic and leading to stripped gears or melted internal potentiometers.
3. **Required Modifications for 6.0V Operation**:
   * **Mandatory Adjustable UBEC**: You *cannot* use the fixed 5.0V UBEC. You must use the **Hobbywing UBEC 10A** and explicitly configure its dip switches to **6.0V**.
   * **Heavy-Gauge Wiring**: Standard JST-SM connectors and 22-24 AWG wires will overheat at 18A peaks. The main power rail from the UBEC to the Servo Breakout Board must be upgraded to **16 AWG or 18 AWG silicone-stranded wire**, securely soldered or clamped in heavy-duty terminals.
   * **Aggressive Software Limiting**: The RL policy must implement strict action rate limiting and monitor the INA219 current sensor to immediately reduce commanded torque if the system approaches 12A continuous draw.
   * **Strongly Recommended**: If operating at 6.0V, consider replacing the MG90S with metal-gear alternatives (e.g., MG996R or the aforementioned Feetech STS3215) to survive the increased mechanical stress.

## Battery Voltage & Current Monitoring

Since the ESP32-S3's native ADC pins (GPIO 1-20) are fully allocated to servos and sensors, and the spare pins (GPIO 38-45) are digital-only, the most robust solution for monitoring the 2S LiPo voltage (max 8.4V) is an **I2C Voltage/Current Sensor**.

* **Recommended Component**: INA219 High-Side DC Current/Voltage Sensor Module.
* **Why**: It shares the existing I2C bus (GPIO 13/14) with the MPU6050 and OLED, consuming **zero** additional GPIO pins. It also provides real-time current draw data, allowing you to monitor actual servo load and detect mechanical binding.
* **Wiring**:
  * **VCC / GND**: 3.3V and GND (for the sensor's logic).
  * **SCL / SDA**: GPIO 14 and GPIO 13 (parallel with IMU/OLED).
  * **VIN+**: Connect directly to the 2S LiPo positive terminal (after the main power switch).
  * **VIN-**: Connect to the positive input of the UBEC and Servo Breakout Board.
* **Software**: Use the `Adafruit_INA219` library (Arduino) or `ina219` module (MicroPython) to read bus voltage and calculate remaining battery percentage.

## Bench Testing: AC/DC Power Supply Setup
For long-duration training, debugging, or bench testing without degrading LiPo cycles, powering the robot directly from a regulated AC/DC power brick is highly recommended.

* **Power Supply Requirements**: You need a 5V DC supply capable of delivering at least **10A continuous (50W)**. Standard USB wall chargers (5V/2A or 3A) will instantly trip their over-current protection. A laboratory bench power supply or a dedicated 5V/10A (or 5V/15A) switching power supply (e.g., Mean Well LRS-50-5) is required.
* **Current Limiting (OCP)**: Configure the power supply's Over-Current Protection (OCP) limit to **~7.0A - 8.0A**. This allows the 6.0A baseline load and typical dynamic peaks to pass, but will safely shut down the output in the event of a wiring short, protecting your servos and breakout board.
* **Wiring**: Use **16 AWG or 18 AWG flexible silicone-stranded wire** for the connection between the power supply and the Servo Breakout Board. Thin or stiff wires will cause significant voltage drop over distance, leading to servo jitter and MCU brownouts.

## Estimated Battery Life (2S LiPo) Across Operational States

Here is the estimated runtime for various 2S LiPo configurations across different operational states:

> *Note: The "C-rating" dictates the **maximum safe discharge current**, not the capacity. For a 1000mAh battery, a 25C rating means it can safely deliver up to 25A (1.0Ah × 25). Since our system peaks at ~12.5A, any battery rated **15C or higher** is technically safe, but 25C+ is strongly recommended to minimize voltage sag and internal heat generation during dynamic movements.*

| Battery Capacity | C-Rating (Min) | Max Safe Discharge | Heavy Load (6.5A) | Moderate Load (2.0A) | Light / Idle Load (1.0A) |
| --- | --- | --- | --- | --- | --- |
| **500 mAh** | 20C | 10A | **~4.5 min** | **~15 min** | **~30 min** |
| **800 mAh** | 25C | 20A | **~7.5 min** | **~24 min** | **~48 min** |
| **1000 mAh** | 25C | 25A | **~9.0 min** | **~30 min** | **~60 min (1 hr)** |
| **1300 mAh** | 25C | 32A | **~12.0 min** | **~39 min** | **~78 min (1hr 18m)** |
| **1500 mAh** | 20C | 30A | **~13.5 min** | **~45 min** | **~90 min (1hr 30m)** |

**Load Scenarios Explained:**
* **Heavy Load (~6.5A)**: Aggressive, continuous servo movement (e.g., rapid trotting, jumping, or RL recovery from falls).
* **Moderate Load (~2.0A)**: Slow walking, balancing in place, or moderate locomotion.
* **Light / Idle Load (~1.0A)**: Standing still, sensor polling, Wi-Fi telemetry, and TFLite inference with minimal servo actuation.

> **Recommendation**: The **1000mAh 25C** battery offers the best balance, providing ~30 minutes of moderate locomotion or ~9 minutes of aggressive testing without adding excessive weight to the robot's center of mass.

## Control Loop & Inference Feasibility (50 Hz Target)

A 50 Hz control loop provides a strict **20 ms budget** per cycle. The ESP32-S3 (240 MHz dual-core with hardware Vector Extensions) comfortably handles this for typical Reinforcement Learning policies.

### 1. Control Frequency Limits & Time Budget
* **Minimum Viable Frequency**: **~30 Hz**. Below this, dynamic legged balancing and RL policies become unstable due to gravity and actuator latency compounding between steps.
* **Maximum Feasible Frequency**: **~250 - 400 Hz**. The total execution time for sensor read, state estimation, and NN inference is ~2.5 ms. While the CPU can theoretically loop at 400 Hz, the practical ceiling is ~250 Hz due to MG90S mechanical response limits and I2C bus saturation. The 50–100 Hz range is the proven sweet spot for this class of robot.

**Time Budget Breakdown (per 50 Hz / 20 ms cycle):**
* **Sensor Acquisition (~1.5 ms)**: Reading MPU6050 (I2C at 400kHz) and 4x FSRs (native ADC).
* **NN Inference (~1.0 ms)**: Running a quantized (int8) TFLite Micro model via ESP-NN.
* **Control Logic & Safety (~0.5 ms)**: Action clipping, PD blending, or state machine updates.
* **Actuation Update (<0.1 ms)**: Writing new duty cycles to the LEDC hardware registers (zero CPU blocking).
* **Total Execution Time**: **~3.1 ms** (leaving ~16.9 ms of slack per cycle).

### 2. Core 1 Control Loop Sequence (ASCII Diagram)

```text
+-----------------------------------------------------------------------+
|                     CORE 1: REAL-TIME CONTROL LOOP                    |
+-----------------------------------------------------------------------+
|                                                                       |
|  [1. I2C READ]  MPU6050 (Accel/Gyro)                      (~1.0 ms)   |
|        |                                                              |
|  [2. STATE EST] Calc velocity, acceleration, filters      (~0.2 ms)   |
|        |                                                              |
|  [3. ADC READ]  4x FSR Foot Sensors (Analog)              (~0.3 ms)   |
|        |                                                              |
|  [4. TFLITE NN] Inference (int8 quantized, 2x256 MLP)     (~1.0 ms)   |
|        |       Input: Proprioception + Latent state                  |
|        |       Output: 12x Target Joint Angles                       |
|        v                                                              |
|  [5. PWM WRITE] LEDC Hardware Duty Cycle Update           (<0.1 ms)   |
|                                                                       |
+---------------------------------+-------------------------------------+
                                  |
                        [ vTaskDelayUntil (e.g., 20ms) ]
                        (Maintains strict 50 Hz cycle timing)
                                  |
                                  v
                            (REPEAT LOOP)
```

### 3. The Open-Loop Servo Reality (Why Not 400 Hz?)
Because MG90S servos are **open-loop** (no internal encoders), the microcontroller only knows the *commanded* angle, not the *actual* angle. 

If you run the control loop at 250+ Hz, the software will issue new commands long before the physical servo has had time to move. This leads to:
1. **Wasted Compute**: Running inference repeatedly for no mechanical gain.
2. **Oscillation/Hunting**: The RL policy may aggressively overcorrect, trying to fix a perceived error that is simply mechanical latency, not a state error.
3. **Overheating**: Rapid, micro-adjustments in PWM duty cycle cause the servo's internal controller to constantly brake and accelerate, drawing high peak currents and burning out the MG90S gearbox.

**The Sweet Spot (50–100 Hz)**: The MG90S no-load speed is ~0.1 sec/60° (slower under load). A 50–100 Hz loop provides the 10–20 ms window the servo physically needs to react to the previous command, perfectly aligning the software loop with mechanical reality.

**RL Mitigation Strategies**:
* **Action Rate Limiting**: Clamp the maximum change in target angle per cycle (e.g., `max_delta = 5°`) in your control code to prevent physically impossible jumps.
* **Sim-to-Real Latency**: Intentionally inject a 10–20 ms delay or low-pass filter on action outputs during simulation training to force the policy to learn robustness to physical lag.
* **Proprioception Trick**: Feed the *commanded* joint angles from the previous timestep into the observation space, giving the network implicit awareness of its own recent commands.

### 4. Neural Network Capacity
The ESP32-S3's Xtensa LX7 core features hardware-accelerated vector instructions, making it highly efficient for int8 matrix multiplications.
* **Typical RL Policy Size**: A standard PPO policy MLP with 45 inputs (proprioception), two hidden layers of 256 neurons, and 12 outputs requires ~80,000 Multiply-Accumulate (MAC) operations.
* **Inference Speed**: 
  * **int8 Quantized**: ~0.5 - 1.0 ms
  * **float32**: ~2.0 - 4.0 ms
* **Maximum Feasible Size**: You can comfortably run networks up to **~500,000 MACs** (e.g., 3×512 neuron layers) within the 20 ms window, though a 2×256 architecture is the sweet spot for balancing inference speed and PSRAM usage.

### 5. Optimization Strategies for Real-Time Performance
* **FreeRTOS Core Pinning**: Pin the strict 50 Hz control loop to **Core 1** to prevent interruptions from WiFi/Bluetooth stack tasks running on Core 0.
* **I2C Fast Mode**: Configure the I2C bus to 400kHz (or 1MHz if wiring is short and clean) to minimize MPU6050 and INA219 read latency.
* **Tensor Arena Placement**: Allocate the TFLite tensor arena in the faster internal SRAM (if the model fits) or ensure PSRAM is configured for quad-SPI for faster memory access.
* **int8 Quantization**: Always export and compile models as int8. The accuracy drop is negligible for RL policies, but the speedup and memory reduction on the ESP32-S3 are massive.

### 6. Servo Upgrade Path: Closed-Loop Serial Bus
If the budget allows, upgrading from open-loop MG90S servos to **closed-loop serial bus servos** is the single most impactful hardware change for RL locomotion. It eliminates the "open-loop reality" problem entirely.

* **Target Servo**: **Feetech / Waveshare STS3215** (or ST3215). Features a 360° magnetic encoder (4096 resolution) providing real-time feedback on position, speed, load/torque, and voltage. Metal gears offer 19.5 kg·cm torque (at 7.4V) or 30 kg·cm (at 12V).
* **Recommended Control Board**: **DFRobot Serial Bus Servo Driver Board (DRI0057)**. 
* **Why This Architecture Wins**:
  1. **Massive GPIO Savings**: Instead of using 12 dedicated PWM pins, all 12 servos daisy-chain on a **single TTL serial line**. The ESP32-S3 only needs **2 GPIO pins** (UART TX/RX, e.g., GPIO 17/18) to talk to the DFRobot driver board.
  2. **True Closed-Loop Proprioception**: The RL policy can read the *actual* joint angles, not just the commanded ones. This enables advanced impedance control and allows the network to detect leg slips, collisions, or mechanical binding via the real-time load feedback.
  3. **Simplified Debugging**: The DFRobot board includes a USB interface, allowing you to easily configure servo IDs, set angle limits, and test movements from a PC before deploying the code to the ESP32-S3.
* **Integration**: The ESP32-S3 connects to the DFRobot board via UART. The DFRobot board handles the heavy lifting of packet routing and power distribution to the daisy-chained STS3215 servos.

## Bill of Materials (BOM)

| Component | Description | Link |
| --- | --- | --- |
| Microcontroller | Waveshare ESP32-S3-Zero (N8R8) or ESP32-S3-Pico | - |
| Servo Driver (Alt) | PCA9685 16-Channel PWM Driver *(Not recommended: causes sequential I2C latency, breaks parallel RL control)* | [AliExpress](https://www.aliexpress.com/item/1005012137990430.html) |
| **Servo Upgrade (Optional)** | **DFRobot Serial Bus Servo Driver Board (DRI0057) + 12x Feetech STS3215** *(Highly recommended for true closed-loop RL control)* | [DFRobot Driver](https://www.dfrobot.com/product-3002.html) |
| Servo Power Board | Servo Breakout Board Serial Wombat | [Amazon UK](https://www.amazon.co.uk/Servo-Breakout-Board-Serial-Wombat/dp/B0F835FT7X) |
| Servos | 12x MG90S Servos | - |
| IMU | GY-521 MPU6050 6-Axis Accelerometer/Gyroscope | [Amazon UK](https://www.amazon.co.uk/Aokin-MPU-6050-Accelerometer-Gyroscope-Converter/dp/B07PSCB75V) |
| Foot Sensors | 4x FSR (Force Sensitive Resistor) Sensors | - |
| Display | 0.96" I2C OLED Display Module | [The Pi Hut](https://thepihut.com/products/0-96inch-oled-display-module-128x94?srsltid=AfmBOoqp-t6EtKOAbMK_AmFE9vgYUqKxAWEiF5hYnIa922I6ifoR53_q) |
| Power Regulation | **Adjustable**: Hobbywing UBEC 10A (2-6S) set to 5.0V via dip switches.<br>**Fixed (Non-adjustable)**: 5.0V/10A dedicated output.<br>*(Input: XT30, Output: Direct Solder or JST-SM)* | [Adjustable](https://www.hobbywing.com/en/products/ubec-10a-2-6s152) / [Fixed](https://www.aliexpress.com/item/1005009420262174.html) |
| Battery Monitor | INA219 I2C High-Side Voltage/Current Sensor Module (shares I2C bus, 0 extra GPIO) | - |
| Battery Charger | Low-cost 2S LiPo Smart Charger (e.g., ISDT 608AC) with XT30 output cable | - |
| Battery | 2S LiPo Battery (500mAh–1000mAh) with XT30 connector | - |