# Stability Shield

## FPGA Safety Supervisor for a Self-Balancing Robot

Stability Shield is an open-source experiment in placing a small hardware safety layer between a robot's control stack and its motor driver.

The first implementation is built for [SEBA Bot](https://github.com/ShahinFi/seba-robot), a self-balancing robot platform being developed in parallel.

The robot's control stack may include firmware, a main controller, normal software, remote operation, or an AI planner. Any of these layers can fail, be misconfigured, be compromised, or request something physically unsafe.

In normal operation, the main controller balances and drives the robot.

Stability Shield does not replace the main controller. It supervises the motor command path below it because the controller itself, or any layer above it, may produce unsafe commands.

Stability Shield sits lower in the command path and checks, clamps, blocks, or triggers a predefined fallback behavior before motor commands reach the motor driver.

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

For low-level balance control, that means keeping the robot inside a recoverable stability envelope.

For higher-level commands, that means preventing physically unsafe actions such as unsafe speed, direction, movement, or operating mode.

---

## Target Platform: SEBA Bot

Stability Shield is initially built for [SEBA Bot](https://github.com/ShahinFi/seba-robot).

SEBA Bot provides the physical self-balancing robot platform, dynamics model, sensing stack, motor system, and main control loop.

Stability Shield is designed around SEBA's specific dynamics, actuator limits, and operating constraints. It does not try to be a universal balancing-robot safety layer from day one.

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
  → physical action constraint checking
  → command blocking / clamping / predefined fallback
```

The longer-term goal is to learn whether this pattern can generalize to other unstable or safety-critical robotic systems.

---

## What This Project Is Trying to Prove

A robot's control stack can be large, updateable, networked, and intelligent.

That stack may include high-level autonomy, remote operation, firmware, and the main controller itself.

The low-level physical safety boundary should be smaller and more independent than the software stack it supervises.

Stability Shield explores this pattern:

```text
complex control stack
        ↓
independent hardware safety supervisor
        ↓
physical actuator
```

For this project, the test platform is SEBA Bot, a self-balancing robot.

A balancing robot is useful because failure is visible and dynamic. A bad low-level control command can push the robot outside the region where balance is still recoverable.

A successful low-level control demo looks like this:

1. The robot balances normally.
2. The controller or firmware sends a destabilizing command.
3. Without the shield, the robot falls or enters an unrecoverable state.
4. With the shield, the FPGA detects loss of recovery margin and intervenes before the unsafe command reaches the motor driver.

A successful higher-level safety demo looks like this:

1. The robot balances normally.
2. A remote operator, autonomy layer, or AI planner requests an unsafe action.
3. The balance controller may still be functioning correctly.
4. Stability Shield detects that the requested motor action violates the allowed physical action envelope.
5. The FPGA blocks, clamps, or triggers a predefined fallback behavior before the unsafe command reaches the motor driver.

---

## Why This Matters

Robots are becoming more complex, connected, and autonomous.

At the same time, AI systems are becoming more capable at finding vulnerabilities, generating exploits, and operating complex software systems. That makes it harder to assume that large software stacks around physical machines will remain trustworthy over time.

That creates two related but different risks:

- Software failure or compromise: bugs, bad updates, unstable firmware, remote access, malicious control, AI-assisted exploit discovery, or compromised controller logic can affect the command path.
- Autonomy failure: remote operators, planners, learned policies, or high-level controllers can request actions that are physically unsafe for the current state.

Stability Shield treats both cases through the same architectural pattern:

**Do not fully trust the software stack above the motor driver. Check the physical consequence before the command reaches the actuator.**

For low-level control failures, the question is:

**Does this command keep the robot dynamically recoverable?**

For higher-level unsafe actions, the question is:

**Does this command stay inside the allowed physical action envelope?**

The FPGA does not try to decide whether a command came from a bug, an attacker, a bad update, a human operator, a controller fault, or an AI planner.

It checks whether the requested motor action is allowed for the robot's current state, operating mode, actuator limits, and safety rules.

---

## Why FPGA

The FPGA is used to keep the safety layer small and separate from the main controller.

A normal firmware or software safety check may still depend on the same complex stack it is supposed to supervise: drivers, update logic, memory safety, network code, RTOS behavior, communication interfaces, controller firmware, and other dependencies.

The FPGA safety logic can be fixed-purpose, deterministic, directly wired into the command path, and much smaller than the software stack it supervises.

This does not make it perfect or impossible to attack.

The point is to reduce what must be trusted.

The main controller can be complex, updateable, and experimental.

The safety supervisor should be small, conservative, and harder to bypass.

---

## Not Just a Limiter

Stability Shield is not meant to be a simple speed, current, or torque clamp.

For a balancing robot, blindly limiting a motor command can make the robot fall. A large command may be dangerous in one state but necessary for recovery in another.

The shield has to evaluate the robot's state, operating mode, and command context, not only the raw command value.

Core signals may include:

- tilt angle
- tilt rate
- wheel speed
- requested motor command
- measured motor current
- battery voltage or available torque margin
- command timing
- actuator saturation
- operating mode
- allowed speed or movement constraints

Using these signals, Stability Shield can enforce two kinds of hardware safety checks:

1. **Recoverability checks**

   These use a simplified model of the robot's dynamics to estimate whether the robot can still recover balance from the current state and proposed command.

2. **Physical action checks**

   These enforce allowed operating boundaries such as speed, direction, mode, actuator limits, and command timing.

The question is not only:

**Is this value below a maximum?**

The real questions are:

**Does this command keep the robot recoverable?**

and:

**Is this command physically allowed for the current state and operating mode?**

---

## Layers This Can Protect Against

Stability Shield sits low in the system, but the unsafe command may originate from any layer above it, including the main controller.

### Firmware or Controller Failure

Examples:

- wrong control gains
- reversed motor direction
- unstable control loop
- corrupted firmware
- compromised motor-control software
- bad software update
- command that would make the robot fall

Here the shield protects against low-level control failures by enforcing recoverability constraints.

### Higher-Level Unsafe Actions

Examples:

- remote operator requests unsafe movement
- AI planner chooses an unsafe goal
- autonomy layer requests movement outside the allowed area
- high-level software requests unsafe speed
- learned policy optimizes for a task while ignoring physical boundaries

Here the shield protects against unsafe decisions above the controller by enforcing physical action constraints.

The FPGA does not need to know why the command was generated.

It only checks whether the requested action is allowed for the current state, operating mode, actuator limits, and safety rules.

---

## Scope

This is a research and educational project.

It is not a certified safety device.

It should not be used as the only safety mechanism for real machines, people, or hazardous environments.

The shield's decisions are only as good as the sensor signals, actuator model, recoverability rules, and physical action constraints used in the implementation.

---

## Core Idea

Software can be complex.

The low-level physical safety boundary should be simple, independent, and hard to bypass.

**Software commands. Hardware enforces the physical boundary.**