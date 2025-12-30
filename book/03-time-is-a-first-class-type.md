# Chapter 3  
## Time Is a First-Class Type

Robotics does not fail because algorithms are wrong.  
It fails because **time assumptions are wrong**.

If you come from game engines or high-performance systems, you are used to treating time as a convenience. In robotics, time is a contract. Violating it produces behavior that looks correct in code and disastrous in motion.

This chapter shows how to make time explicit, enforceable, and difficult to misuse.

---

## The Lie of `deltaTime`

Most real-time software begins with something like this:

```cpp
float dt;
```

This variable usually means:

* “Time since last frame”
* “Approximately correct”
* “Probably fine”

In robotics, this is not enough.

* Sensors arrive late.
* Messages are dropped.
* Control loops jitter.
* Clocks drift.

If your API accepts a raw floating-point number, you have already lost information:

* What unit is this?
* Is it wall time or monotonic time?
* Is it bounded?
* Is it safe to integrate over?

Robotics software must _force_ time to be explicit.

## Time as a Strong Type

We start with a strong unit type for time. No aliases. No typedefs. No macros.

```cpp
namespace units {

struct SecondsTag {};

template <typename Tag>
class Quantity {
public:
    explicit constexpr Quantity(double value) noexcept
        : value_{value} {}

    [[nodiscard]] constexpr double value() const noexcept {
        return value_;
    }

    explicit constexpr operator double() const noexcept {
        return value_;
    }

    constexpr Quantity operator+(Quantity rhs) const noexcept {
        return Quantity{value_ + rhs.value_};
    }

    constexpr Quantity operator-(Quantity rhs) const noexcept {
        return Quantity{value_ - rhs.value_};
    }

private:
    double value_;
};

using Seconds = Quantity<SecondsTag>;

} // namespace units
```

This immediately prevents:

* Passing milliseconds where seconds are expected
* Mixing time with dimensionless values
* Silent unit drift during refactors

Time now has semantic weight.

## Functions That Require Time Must Say So

A core rule in this book:

> If a function depends on time, it must accept time explicitly.

Bad:

```cpp
double update(double error);
```

Better:

```cpp
double update(double error, units::Seconds dt);
```

This makes misuse visible at the call site and forces the caller to confront time.

## Fixed-Step Control Is a Requirement, Not an Optimization

Robots behave best when control runs at a fixed cadence. Variable-step control introduces instability, even when the average rate looks correct.

A simple fixed-step guard:

```cpp
class FixedStepGuard {
public:
    FixedStepGuard(units::Seconds expected,
                   units::Seconds tolerance)
        : expected_{expected},
          tolerance_{tolerance} {}

    [[nodiscard]]
    bool acceptable(units::Seconds dt) const noexcept {
        const double error =
            std::abs((dt - expected_).value());
        return error <= tolerance_.value();
    }

private:
    units::Seconds expected_;
    units::Seconds tolerance_;
};
````

Usage:

```
FixedStepGuard guard(
    units::Seconds{0.01},   // 100 Hz
    units::Seconds{0.002}   // ±2 ms jitter
);

if (!guard.acceptable(dt)) {
    // degrade, log, or halt
}
```

This is not paranoia.
This is how you prevent oscillations caused by scheduler jitter.

## Timestamped Data Is Not Optional

Robotics code should never accept raw sensor values without context.

Instead:

```cpp
template <typename T>
struct Timestamped {
    T value;
    units::Seconds timestamp;
};
```

This allows you to:

* Reject stale measurements
* Detect time going backwards
* Align data streams correctly
* Perform interpolation or buffering safely

If your system cannot answer “when was this true?”, it cannot reason correctly.

## Interpolation Beats Extrapolation

When control needs state at a specific time, interpolation is safe. Extrapolation is a gamble.

A minimal time buffer:

```cpp
#include <deque>
#include <optional>

template <typename T>
class TimeBuffer {
public:
    void push(Timestamped<T> sample) {
        buffer_.push_back(sample);
        while (buffer_.size() > max_size_) {
            buffer_.pop_front();
        }
    }

    std::optional<T> interpolate(units::Seconds t) const {
        if (buffer_.size() < 2) {
            return std::nullopt;
        }

        for (size_t i = 1; i < buffer_.size(); ++i) {
            const auto& a = buffer_[i - 1];
            const auto& b = buffer_[i];

            if (a.timestamp.value() <= t.value() &&
                t.value() <= b.timestamp.value()) {

                const double alpha =
                    (t - a.timestamp).value() /
                    (b.timestamp - a.timestamp).value();

                return a.value * (1.0 - alpha) +
                       b.value * alpha;
            }
        }
        return std::nullopt;
    }

private:
    std::deque<Timestamped<T>> buffer_;
    size_t max_size_{100};
};

```

Interpolation is predictable.
Extrapolation is how robots drive off tables.

## Wall Time vs Monotonic Time

Robotics software should never use wall-clock time for control.

Wall clocks:

* Jump
* Drift
* Synchronize
* Lie

Control loops must use monotonic time:

* Always increasing
* Immune to NTP corrections
* Stable under load

This distinction should exist in your API design, even if both map to the same underlying clock initially.

## Time as a Safety Boundary

Every time boundary is also a trust boundary.

Questions your code should answer explicitly:

* What happens if data is late?
* What happens if data is missing?
* What happens if time jumps?
* What happens if time stalls?

Silence is not acceptable.

A robot should slow, stop, or enter a fault state when time assumptions are violated.

## Summary

Time is not a parameter.
Time is a constraint.

Robotics software that treats time as an afterthought will behave beautifully in simulation and dangerously in reality.

By making time:

* Explicit
* Typed
* Checked
* Bounded

you force correctness into the structure of your code.