---
title: What robots taught me about game feel
date: 2026-02-14
tags: gamedev, robotics, craft
excerpt: Latency, prediction, and the uncanny valley of control — the things that make a robot feel alive are the same things that make a game feel good.
---

I split my time between two worlds that look unrelated: I make robots, and I make games. The longer I do both, the more I think they are the same craft wearing different clothes.

## Both are about latency

A game feels *good* or *bad* in the first thirty seconds, and almost always the difference is latency. Press the button, see the thing happen. Sixteen milliseconds feels alive. A hundred feels like wading through syrup.

Robots are identical, except the stakes are physical. A control loop that responds late does not feel sluggish — it falls over. In both cases the enemy is the same: the gap between intent and response.

## Both lie to you, beautifully

Good games predict. When you swing a sword, the animation often starts *before* the server confirms the hit, because waiting would feel terrible. The game makes a confident guess and quietly corrects if it was wrong.

Robots do this too. A well-tuned controller is constantly predicting the near future — where will this body be in 20ms? — and acting on the prediction rather than the present. The present is already too old to be useful.

> The art in both fields is hiding the seam between what is real and what is predicted.

## The uncanny valley of control

There is a window where a robot moves *almost* like a creature, and it is deeply unsettling. Too smooth and it reads as a puppet; too jerky and it reads as broken; somewhere in between it reads as *alive*, and that is the unsettling part.

Game characters live in the same valley. The most memorable ones have a slight imperfection in their motion — a wind-up, an overshoot, a settle — that signals weight and intent. Perfect motion is dead motion.

## What I carry between them

From games I learned to obsess over feel — to treat the subjective sensation as a real, debuggable property. From robots I learned that feel is not magic; it is latency, prediction, and damping, and you can measure every one of them.

The two disciplines keep handing each other tools. I have stopped trying to keep them separate.
