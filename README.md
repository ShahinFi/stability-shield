# Stability Shield

## FPGA Safety Supervisor for a Self-Balancing Robot

Stability Shield is an open-source experiment in placing a small hardware safety layer between a robot's control stack and its motor driver.

The first implementation is built for [SEBA Bot](https://github.com/ShahinFi/seba-robot), a self-balancing robot platform being developed in parallel.

The robot's control stack may include firmware, a main controller, normal software, remote operation, or an AI planner. Any of these layers can fail, be misconfigured, be compromised, or request something physically unsafe.

In normal operation, the main controller balances and drives the robot. Stability Shield does not replace it. Instead, it supervises the motor command path below the controller and checks, clamps, blocks, or triggers a predefined fallback behavior before commands reach the motor driver.

```text
operator / AI / software
        ↓
main controller / firmware  ←  sensors / state feedback
        ↓
Stability Shield            ←  sensors / state feedback
        ↓
motor driver
        ↓
robot
```

The goal is to keep the final motor action inside hardware-enforced physical safety boundaries.

For low-level balance control, this means keeping the robot inside a recoverable stability envelope.

For higher-level commands, this means preventing physically unsafe actions such as unsafe speed, direction, movement, or operating mode.

---

## Target Platform

Stability Shield is initially built for [SEBA Bot](https://github.com/ShahinFi/seba-robot).

SEBA provides the physical self-balancing robot platform, dynamics model, sensing stack, motor system, and main control loop.

Stability Shield is designed around SEBA's specific dynamics, actuator limits, and operating constraints. SEBA is the first concrete target system, not just a loose example.

The longer-term goal is to learn whether this pattern can generalize to other unstable or safety-critical robotic systems.

---

## What This Project Tests

Stability Shield explores this pattern:

```text
complex control stack
        ↓
independent hardware safety supervisor
        ↓
physical actuator
```

A self-balancing robot is useful because failure is visible and dynamic. A bad low-level control command can push the robot outside the region where balance is still recoverable. A bad higher-level command may keep the balance loop working while still requesting unsafe behavior.

A successful demo shows:

1. The robot balances normally.
2. The controller, firmware, remote operator, autonomy layer, or AI planner sends an unsafe command.
3. Without the shield, the robot falls, enters an unrecoverable state, or performs an unsafe action.
4. With the shield, the FPGA detects that the command violates the recoverability envelope or physical action envelope.
5. The FPGA blocks, clamps, or triggers a predefined fallback before the unsafe command reaches the motor driver.

---

## Why This Matters

Robots are becoming more complex, connected, and autonomous. At the same time, AI systems are becoming more capable at finding vulnerabilities, generating exploits, and operating complex software systems.

That makes it harder to assume that large software stacks around physical machines will remain trustworthy over time.

Stability Shield does not try to decide whether a command came from a bug, an attacker, a bad update, a human operator, a controller fault, or an AI planner.

It only asks whether the requested motor action is allowed for the robot's current state, operating mode, actuator limits, and safety rules.

---

## Why FPGA

The FPGA is used to keep the safety layer small and separate from the main controller.

A firmware or software safety check may still depend on the same complex stack it is supposed to supervise: drivers, update logic, memory safety, network code, RTOS behavior, communication interfaces, controller firmware, and other dependencies.

The FPGA safety logic can be fixed-purpose, deterministic, directly wired into the command path, and much smaller than the software stack it supervises.

This does not make it perfect or impossible to attack. The point is to reduce what must be trusted.

---

## Safety Logic

Stability Shield evaluates motor commands using SEBA-specific state information and a simplified safety model.

For a self-balancing robot, safety depends on state. The same motor command can be necessary for recovery in one moment and dangerous in another.

Core signals may include:

- tilt angle
- tilt rate
- wheel speed
- requested motor command
- measured motor current
- battery voltage or torque margin
- command timing
- actuator saturation
- operating mode
- allowed speed, direction, or movement constraints

Stability Shield performs two kinds of checks:

1. **Recoverability checks**

   These use SEBA's dynamics and state information to estimate whether the current state and proposed command keep the robot inside a recoverable balance envelope.

2. **Physical action checks**

   These enforce operating constraints such as speed, direction, mode, actuator limits, timing, and movement boundaries. Some checks may be simple limits; others may use the robot model when the safety of the action depends on state.

The key questions are:

```text
Does this command keep the robot dynamically recoverable?

Is this command allowed for the current state, operating mode, actuator limits, and physical constraints?
```

---

## Scope

This is a research and educational project.

It is not a certified safety device and should not be used as the only safety mechanism for real machines, people, or hazardous environments.

The shield's decisions are only as good as the sensor signals, actuator model, recoverability rules, and physical action constraints used in the implementation.

---

## Core Idea

Software can be complex.

The low-level physical safety boundary should be simple, independent, and hard to bypass.

**Software commands. Hardware enforces the physical boundary.**