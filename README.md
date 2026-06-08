# Stability Shield

## FPGA Safety Supervisor for a Self-Balancing Robot

Stability Shield is an open-source experiment in placing a small hardware safety layer between a robot's control stack and its motor driver.

The first implementation is built for [SEBA Bot](https://github.com/ShahinFi/seba-robot), a self-balancing robot platform being developed in parallel.

The robot may be controlled by firmware, normal software, a remote operator, or an AI planner. Any of those layers can fail, be compromised, or request something physically unsafe.

The main controller still balances and drives the robot.

Stability Shield does not replace the main controller. It sits lower in the command path and checks, clamps, blocks, or triggers a safe fallback before motor commands reach the motor driver.

```text
operator / AI / software / firmware
        ↓
main controller  ←  sensors / state feedback
        ↓
Stability Shield ←  sensors / state feedback
        ↓
motor driver
        ↓
robot
```

The goal is to keep the final motor action inside a hardware-enforced recoverability envelope.

---

## Target Platform: SEBA Bot

Stability Shield is initially built for [SEBA Bot](https://github.com/ShahinFi/seba-robot).

SEBA Bot provides the physical self-balancing robot platform, dynamics model, sensing stack, motor system, and main control loop.

Stability Shield is designed around SEBA's specific dynamics and actuator limits. It does not try to be a universal balancing-robot safety layer from day one.

For the first implementation, SEBA is the concrete target system, not just a loose example.

This keeps the project concrete:

```text
SEBA Bot
  → robot mechanics
  → sensors
  → motor driver
  → balancing controller
  → dynamics and control model

Stability Shield
  → FPGA safety supervisor for the SEBA implementation
  → recoverability-envelope checking
  → command blocking / clamping / safe fallback
```

The longer-term goal is to learn whether this pattern can generalize to other unstable or safety-critical robotic systems.

---

## What This Project Is Trying to Prove

A robot's software stack can be large, updateable, networked, and intelligent.

The final safety boundary should be smaller and more independent than that.

Stability Shield explores this pattern:

```text
complex control stack
        ↓
independent hardware safety supervisor
        ↓
physical actuator
```

For this project, the test platform is SEBA Bot, a self-balancing robot.

A balancing robot is useful because failure is visible and dynamic. A bad command does not only exceed a simple speed or current limit. It can push the robot outside the region where balance is still recoverable.

A successful demo looks like this:

1. The robot balances normally.
2. The controller or a higher-level layer sends a bad command.
3. Without the shield, the robot falls or enters an unrecoverable state.
4. With the shield, the FPGA detects loss of recovery margin and intervenes before the unsafe command reaches the motor driver.

---

## Why This Matters

Robots are becoming more complex, connected, and autonomous.

That creates two related risks:

- Software failure or compromise: bugs, bad updates, unstable firmware, remote access, or malicious control can affect the command path.
- Autonomy failure: planners, learned policies, or high-level controllers can request actions that are physically unsafe for the current state.

Stability Shield treats both cases the same way.

It does not try to decide whether a command came from a bug, an attacker, a bad update, a human operator, or an AI planner.

It only asks:

**Given the robot's current state, actuator limits, and proposed command, does the robot remain recoverable?**

---

## Why FPGA

The FPGA is used to keep the safety layer small and separate from the main controller.

A normal firmware safety check may still depend on a large software stack: drivers, update logic, memory safety, network code, RTOS behavior, communication interfaces, and other dependencies.

The FPGA safety logic can be fixed-purpose, deterministic, directly wired into the command path, and much smaller than the software stack it supervises.

This does not make it perfect or impossible to attack.

The point is to reduce what must be trusted.

The main controller can be complex and updateable.

The safety supervisor should be small, conservative, and harder to bypass.

---

## Not Just a Limiter

Stability Shield is not meant to be a simple speed, current, or torque clamp.

For a balancing robot, blindly limiting a motor command can make the robot fall. A large command may be dangerous in one state but necessary for recovery in another.

The shield has to evaluate the robot's state, not only the raw command value.

Core signals may include:

- tilt angle
- tilt rate
- wheel speed
- requested motor command
- measured motor current
- battery voltage or available torque margin
- command timing
- actuator saturation

Using these signals, Stability Shield implements a simplified recoverability model of the robot.

The question is:

**Does this command help keep the robot inside the recoverable stability envelope, or does it push the robot closer to an unrecoverable fall?**

---

## Layers This Can Protect Against

Stability Shield sits low in the system, but the unsafe command may originate from any layer above it.

### Firmware or Controller Failure

Examples:

- wrong control gains
- reversed motor direction
- unstable control loop
- corrupted firmware
- compromised motor-control software
- bad software update
- command that would make the robot fall

Here the shield protects against low-level control failures.

### Higher-Level Unsafe Actions

Examples:

- remote operator requests unsafe movement
- AI planner chooses an unsafe goal
- autonomous layer requests movement outside the allowed area
- high-level software requests unsafe speed
- learned policy optimizes for a task while ignoring physical stability

Here the shield protects against unsafe decisions above the controller.

The FPGA does not need to understand the intent of the command.

It only checks whether the requested action is physically allowed for the current state.

---

## Scope

This is a research and educational project.

It is not a certified safety device.

It should not be used as the only safety mechanism for real machines, people, or hazardous environments.

---

## Core Idea

Software can be complex.

The final physical boundary should be simple, independent, and hard to bypass.

**Software commands. Hardware enforces recoverability.**