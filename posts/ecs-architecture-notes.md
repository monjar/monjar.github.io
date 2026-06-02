---
title: Why I keep reaching for an ECS
date: 2026-04-02
tags: gamedev, c++, architecture
excerpt: Entity-component-system architecture isn't just a performance trick. It's a way of thinking that changed how I structure everything — including robot code.
---

Every game engine I have written eventually converges on the same shape: an entity-component-system. It took me three projects to understand *why*, and the understanding turned out to be useful far outside of games.

## The inheritance trap

You start a game with a clean `GameObject` base class. Then you need a moving object, so you subclass. Then a moving object that takes damage. Then a moving object that takes damage but only sometimes renders. Six months later your class hierarchy looks like a family tree drawn by someone with a grudge.

The problem is that behaviour does not compose down an inheritance tree. It composes *across* one.

## Components as plain data

ECS answers this by separating the three things we were cramming together:

- **Entities** are just IDs. An integer. Nothing else.
- **Components** are plain old data — position, velocity, health. No methods.
- **Systems** are functions that run over every entity with a given set of components.

```cpp
struct Position { float x, y, z; };
struct Velocity { float x, y, z; };

void integrate(World& w, float dt) {
    for (auto [pos, vel] : w.view<Position, Velocity>()) {
        pos.x += vel.x * dt;
        pos.y += vel.y * dt;
        pos.z += vel.z * dt;
    }
}
```

That `integrate` function does not care whether the entity is a player, a bullet, or a falling leaf. It cares only that the entity *has a position and a velocity*. Behaviour is defined by what data you possess, not by what you inherit.

## The cache is the point

There is a performance story here too. When components live in tightly packed arrays, the `integrate` loop walks contiguous memory. The cache prefetcher loves you. The same loop written over a vector of polymorphic `GameObject*` pointers chases its own tail through the heap and runs an order of magnitude slower.

> Data-oriented design is just respect for the machine you are actually running on.

## It followed me into robotics

Here is the part I did not expect. When I started writing robot control software, I reached for the same shape. A robot is a bag of components — joints, sensors, estimators — and the control logic is a set of systems running over them at a fixed tick. The line between "game loop" and "control loop" turned out to be thinner than my degree suggested.

Architecture, it turns out, is mostly about deciding what gets to change independently. ECS is one very good answer.
