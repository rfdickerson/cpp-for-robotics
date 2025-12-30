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

