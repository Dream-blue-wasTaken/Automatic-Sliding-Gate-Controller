# Automatic Sliding Gate Controller

An industrial-grade PLC ladder logic implementation for an automatic sliding gate controller designed and simulated in OpenLadder, featuring safety-hardened auto-reverse and travel fault detection.

## 💻 Simulation in OpenLadder

The system has been fully tested and simulated within the OpenLadder IDE environment:

![OpenLadder Simulator - Gates and Coils Overview](docs/1.png)
*Figure 1: Main ladder logic rungs showing the primary control loops and latching circuits.*

![OpenLadder Simulator - Monitoring Interface](docs/2.png)
*Figure 2: Active simulation state with interlocked motor coils and active limit switch feedback.*

![OpenLadder Simulator - Fault Testing](docs/3.png)
*Figure 3: Testing the safety-hardened auto-reverse loop with simulated obstruction triggers.*

---

## 📊 State Machine Diagram

Below is the conceptual state machine diagram governing the PLC controller's operation:

```mermaid
stateDiagram-v2
    [*] --> Stopped : Power On
    
    state Stopped {
        [*] --> Disabled
        Disabled --> Enabled : Start_Button Pressed (NO)
        Enabled --> Disabled : Stop_Button Pressed (NC) / Motor_Fault Tripped
    }
    
    Stopped --> Gate_Closed : System_Run = True
    
    state Gate_Closed {
        [*] --> Idle
        Idle --> Gate_Opening : Vehicle_Sensor (Rising Edge)
    }

    state Gate_Opening {
        [*] --> Motor_Open_On
        Motor_Open_On --> Dwell_Open : Gate_Open Limit Switch Actuated
        Motor_Open_On --> Fault_Lockout : Travel_Fault_Timer Done (15s Timeout)
        Motor_Open_On --> Stopped : System_Run = False (Stop / Fault)
    }

    state Dwell_Open {
        [*] --> Dwell_Timer_Active
        Dwell_Timer_Active --> Gate_Closing : Open_Timer Done (5s Dwell)
        Dwell_Timer_Active --> Stopped : System_Run = False (Stop / Fault)
    }

    state Gate_Closing {
        [*] --> Motor_Close_On
        Motor_Close_On --> Gate_Closed : Gate_Closed Limit Switch Actuated
        Motor_Close_On --> Gate_Opening : Obstruction_Sensor Tripped (Auto-Reverse)
        Motor_Close_On --> Fault_Lockout : Travel_Fault_Timer Done (15s Timeout)
        Motor_Close_On --> Stopped : System_Run = False (Stop / Fault)
    }

    state Fault_Lockout {
        [*] --> Motor_Fault_Active
        Motor_Fault_Active --> Stopped : Stop_Button Pressed (Fault Reset)
    }
```

---

## 🛠️ Design Decisions

### 1. Internal Latch Bits vs. Direct Raw Output Control
Instead of directly driving the physical motor outputs (`Motor_Open` and `Motor_Close`) with latch/unlatch coils, the system uses internal memory state bits (`Gate_Opening` and `Gate_Closing`).
* **Double Coil Prevention:** In standard PLC programming, writing to the same physical coil in multiple rungs leads to conflict ("double coil" syndrome) where only the last rung's evaluation determines the physical state. Internal bits allow multiple conditions (obstruction sensors, limit switches, timers, and stop signals) to influence the movement state cleanly across different rungs.
* **Safety Actuation Layer:** The physical outputs are kept as single-coil, non-latching rungs that read the internal state bits and are directly interlocked with their respective limit switches. This ensures that the motor output drops out instantly when a limit switch is reached, regardless of software latch bugs.
* **Clear State Separation:** Decoupling status representation (what the controller wants to do) from physical actuation (what the outputs are currently doing) makes auditing and debugging significantly easier.

### 2. Rising-Edge Detection on the Vehicle Sensor
The vehicle loop detection is processed via edge-detection logic (using `Vehicle_Latch` as a rising-edge memory bit).
* **Prevention of Gate Sticking:** If the vehicle sensor were evaluated as a static level, a car parked on the loop or a failed/shorted loop sensor would keep the opening command active indefinitely. The gate would remain stuck open.
* **Discrete Event Initiation:** Edge-detecting ensures the gate initiates a single opening sequence on vehicle *arrival*. Once open and the dwell timer (5 seconds) completes, the gate is allowed to close even if the car is still parked on the loop (or if the car has backed away), transferring safety checking to the dedicated obstruction photo-eyes.

---

## ⚠️ Known Limitations & Next Steps

This project is a high-fidelity PLC training simulation. Transitioning it to a production-grade, certifiable industrial gate operator requires addressing the following limitations:

### 1. Real Hardware I/O Mapping
* **Current Status:** PLC inputs and outputs currently reference direct internal memory tags.
* **Next Steps:** Map the logical tags to physical I/O registers (e.g., `%IX0.0`, `%QX0.0`) matching the target controller's hardware catalog.
* **Debouncing:** Add hardware or software filtering (timer-off/on debouncing) on inputs like the start/stop buttons and limit switches to prevent contact bounce from triggering unintended state transitions.

### 2. Safety-Rated Emergency Stop (E-Stop)
* **Current Status:** The `Stop_Button` logic is processed as a standard digital input within the PLC software routine.
* **Next Steps:** A safety-rated E-stop cannot rely on standard software code. True emergency stop circuits must cut motor contractor power in hardware using dual-channel, monitored safety relays (or a safety-certified PLC running redundant Failsafe code with SIL-3/PL-e ratings) to guarantee actuator shutdown in the event of controller CPU failure.

### 3. UL 325 Safety Compliance
To comply with the UL 325 standard for gate operators, the system must implement:
* **Monitored Entrapment Protection:** Entrapment sensors (e.g., photo-eyes or sensing edges) must be actively monitored. The controller must verify the sensor's health (typically by pulsing its power and checking for a state transition) before initiating gate movement. If sensor monitoring fails, the gate must not run.
* **Warning Device Actuation:** A pre-start warning alarm/light output must actuate for at least 2 seconds prior to movement and remain active during travel.
* **Double Obstruction Lockout:** If the obstruction sensor (or motor current torque limit) trips twice consecutively within a single travel cycle, the controller must enter a hard lockout state that cannot be cleared automatically, requiring a dedicated physical manual reset.
