---
title: Building a balance controller that doesn't panic
date: 2026-05-18
tags: robotics, c++, control
excerpt: Notes on shipping a real-time LQR balance loop in C++ — and why the hardest part was teaching it to fail gracefully.
---

There is a particular kind of silence in a lab right before a robot either stands up or throws itself across the room. I have learned to listen for it.

This post is about the balance controller I have been building for a small two-wheeled platform — the math, the C++, and the long tail of failure modes that no textbook prepares you for.

## The loop, in one breath

At its heart the controller is unremarkable. We estimate the body state — pitch, pitch rate, wheel velocity — and feed it through a gain matrix tuned with LQR. The output is a wheel torque. We do this five hundred times a second and try very hard not to allocate memory while doing it.

```cpp
Vector3f x { pitch, pitch_rate, wheel_vel };
float tau = -(K * x).value();
motor.set_torque(clamp(tau, -TAU_MAX, TAU_MAX));
```

The elegance of LQR is that the *interesting* engineering does not live in this line. It lives everywhere around it.

## Estimation is the whole game

A controller is only as good as the state you hand it. Our pitch estimate is a complementary filter blending the accelerometer's slow truth with the gyro's fast lie. The accelerometer knows which way is down but is fooled by every bump; the gyro is smooth but drifts. Blend them and you get something usable.

> The robot does not know it is balancing. It only knows a number, and whether that number is getting worse.

The trap here is latency. Every millisecond between *the world changes* and *the torque responds* is phase margin you will never get back. We moved the filter onto the motor controller's MCU specifically to shave 4ms off the loop. That 4ms was the difference between a robot that stood and one that oscillated itself to death.

## Failing gracefully

The thing nobody tells you: a balance controller spends most of its life **not** balancing. It is being picked up, set down, bumped, and asked to recover from angles LQR was never linearized around.

So the real architecture is a small state machine wrapped around the elegant loop:

- **idle** — wheels free, waiting for an upright-ish pose
- **balancing** — the loop above, full authority
- **caught** — pitch exceeded the recovery envelope; cut torque, don't fight gravity

That last state is the one that earns its keep. A robot that knows when to give up does not break itself.

## What I would tell past me

Tune in simulation, but never *trust* simulation. The model is a hypothesis; the hardware is the experiment. Spend the money on a good IMU. And write the safety cutoff first, before the controller — because the day you need it, you will not be in a position to add it.

Next time: the wheel encoder calibration saga, which cost me a weekend and a small amount of dignity.
