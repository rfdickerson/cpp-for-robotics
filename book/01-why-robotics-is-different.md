# Chapter 1  
## Why Robotics Is Different

If you have spent years writing C++ for games, graphics, engines, or high-performance systems, robotics will feel familiar for about five minutes.

After that, it will feel strange.

Not because the code is harder.  
Not because the math is scarier.  
But because the **assumptions you are used to relying on quietly stop being true**.

Robotics is where software leaves the safety of abstraction and starts negotiating with reality. Reality is noisy, late, incomplete, and occasionally hostile.

This chapter explains why robotics software demands a different mindset, even for very strong C++ developers.

---

## Code That Moves Mass

Most software fails politely.

A backend crashes.  
A frame drops.  
A packet is retried.  
A process restarts.

In robotics, failure has *momentum*.

When your code controls motors, actuators, or steering, bugs do not just return wrong values. They apply force. They create motion. They persist until something stops them.

This single fact changes everything.

A robot does not care that your code is “mostly correct.”  
It does not care that an edge case is rare.  
It does not care that undefined behavior “probably won’t happen.”

The physical world will happily amplify your smallest assumptions.

---

## Undefined Behavior Is Not an Academic Concern

In many software domains, undefined behavior is treated as a performance concern or a theoretical footnote. In robotics, it is a safety issue.

Uninitialized memory does not just corrupt data.  
It can command stale velocity.  
It can skip a safety check.  
It can prevent a stop condition from triggering.

In robotics, **the worst failure mode is not a crash**.

The worst failure mode is *silence*.

A robot that crashes is annoying.  
A robot that keeps moving because nothing explicitly told it to stop is dangerous.

This is why robotics code tends to:
- Favor explicit initialization
- Avoid sentinel values
- Reject “best effort” APIs
- Default to zero motion, not last known motion

Failing closed is not optional.

---

## Time Is Not a Convenience Variable

In game engines, time often looks like this:

```cpp
float delta_time;
