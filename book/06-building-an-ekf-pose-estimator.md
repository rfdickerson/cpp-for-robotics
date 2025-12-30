# Chapter 6

## Building an EKF Pose Estimator (Library-First)

Robots never know exactly where they are.

They estimate.

This chapter builds a **production-quality Extended Kalman Filter (EKF) pose estimator** for a differential-drive robot, written as a **standalone modern C++ library**. No middleware. No callbacks. No magic.

By the end of this chapter, you will have:
* A reusable pose estimation core
* Explicit state, covariance, and time handling
* Deterministic prediction and correction steps
* Clear trust boundaries between sensors and belief

This is not a SLAM system.
This is the *foundation SLAM systems are built on*.

---

## What We Are Estimating

We work in **SE(2)**: planar motion with orientation.

Our state vector is:

$$
\mathbf{x} =
\begin{bmatrix}
x \\
y \\
	heta
\end{bmatrix}
$$

Where:
* $x, y$ are position in meters
* $\theta$ is heading in radians

We also track a **covariance matrix** $\mathbf{P} \in \mathbb{R}^{3\times 3}$, which represents our uncertainty about the state.

---

## Design Constraints (Non-Negotiable)

Before writing code, we lock in constraints:

* **No hidden state**
* **Explicit timestamps**
* **Prediction and correction are separate**
* **Units enforced at the API boundary**
* **Deterministic execution**
* **No dynamic allocation in the hot path**

If a design violates one of these, it does not ship.

---

## Library Layout

We structure the estimator as a small, focused module:

```text
estimation/
├── pose2.hpp
├── covariance.hpp
├── ekf.hpp
├── motion_model.hpp
├── measurement.hpp
```

This keeps concerns isolated and testable.

---

## Pose and Covariance Types

We start with strong types.

```cpp
#include <array>

struct Pose2 {
    units::Meters x{0.0};
    units::Meters y{0.0};
    units::Radians yaw{0.0};
};

struct Covariance3 {
    // Row-major 3x3
    std::array<double, 9> data{
        0.0, 0.0, 0.0,
        0.0, 0.0, 0.0,
        0.0, 0.0, 0.0
    };

    double& operator()(int r, int c) {
        return data[r * 3 + c];
    }

    double operator()(int r, int c) const {
        return data[r * 3 + c];
    }
};

```

No linear algebra library yet. This keeps the math visible and auditable.

## The EKF State object

The EKF owns *belief*, not sensors.

```cpp
struct EkfState {
    Pose2 mean;
    Covariance3 covariance;
    units::Seconds timestamp{0.0};
};
```

This is the **single source of truth**.

## Motion Model (Prediction Step)

For a differential drive robot, we predict motion using commanded velocities.

```cpp
struct ControlInput {
    units::MetersPerSecond v;
    units::RadiansPerSecond omega;
};
```

The prediction step advances state forward in time:

```cpp
Pose2 predict_pose(
    const Pose2& pose,
    ControlInput u,
    units::Seconds dt)
{
    const double theta = pose.yaw.value();

    Pose2 out;
    out.x = pose.x + units::Meters{u.v.value() * std::cos(theta)} * dt;
    out.y = pose.y + units::Meters{u.v.value() * std::sin(theta)} * dt;
    out.yaw = units::wrap_angle(
        pose.yaw + units::Radians{u.omega.value()} * dt);

    return out;
}
```

This function is:

Pure

Deterministic

Unit-safe

Exactly what we want.

## Covariance Prediction

Uncertainty grows when we move.

```cpp
Covariance3 predict_covariance(
    const Covariance3& P,
    const Covariance3& Q)
{
    Covariance3 out = P;
    for (int i = 0; i < 9; ++i) {
        out.data[i] += Q.data[i];
    }
    return out;
}
```

Here, $Q$ is the process noise covariance. We inject uncertainty explicitly rather than pretending motion is perfect.

## The EKF Class

Now we assemble the estimator.

```cpp
class PoseEkf {
public:
    explicit PoseEkf(Covariance3 process_noise)
        : Q_{process_noise} {}

    const EkfState& state() const noexcept {
        return state_;
    }

    void initialize(Pose2 initial_pose,
                    Covariance3 initial_cov,
                    units::Seconds t0)
    {
        state_.mean = initial_pose;
        state_.covariance = initial_cov;
        state_.timestamp = t0;
    }

    void predict(ControlInput u,
                 units::Seconds t)
    {
        const auto dt = t - state_.timestamp;

        state_.mean = predict_pose(state_.mean, u, dt);
        state_.covariance = predict_covariance(state_.covariance, Q_);
        state_.timestamp = t;
    }

private:
    EkfState state_;
    Covariance3 Q_;
};
```

Important details:

* Prediction requires a timestamp
* $dt$ is derived, not passed blindly
* No prediction without initialized state

## Measurement Updates (Correction Step)

Sensors do not overwrite state.
They nudge it.

We represent measurements explicitly:

```cpp
struct PositionMeasurement {
    units::Meters x;
    units::Meters y;
    Covariance3 R;
};
```

A simple correction step:

```cpp
void correct_position(const PositionMeasurement& z)
{
    // Innovation
    const double dx = z.x.value() - state_.mean.x.value();
    const double dy = z.y.value() - state_.mean.y.value();

    // Apply a simple gain (placeholder)
    constexpr double gain = 0.5;

    state_.mean.x += units::Meters{gain * dx};
    state_.mean.y += units::Meters{gain * dy};

    // Covariance reduction omitted for clarity
}
```

This is not a full EKF yet, intentionally.

The goal here is architectural clarity:

* Prediction and correction are distinct
* Measurements are explicit
* State is never overwritten blindly

In later chapters, we refine the math without changing the structure.

## Time Alignment is Mandatory

All measurements must be time-aligned before correction.

This is why Chapter 7 exists.

Never do this:

```cpp
ekf.correct(z);
```

Always do this:

```cpp
ekf.predict(u, z.timestamp);
ekf.correct(z);
```

If you correct without aligning time, your estimator will appear stable until it catastrophically diverges.

## What We Have Built

At this point, you have:

* A deterministic EKF pose estimator
* Explicit belief state
* Clear separation of motion and measurement
* A library that compiles without ROS

This is already more disciplined than many production systems.

## What We Deliberately Deferred

We have _not_ yet:

* Added Jacobians
* Added delayed measurements
* Added multiple sensor types
* Added angle residual handling

Those belong in subsequent chapters.

This chapter establishes **architecture first, math second**.

## Summary

State estimation is not about equations.
It is about discipline.

By:
* Owning state explicity
* Treating time as a contract
* Separating predictions and correction
* Encoding uncertainty deliberately

you create systems that degrade gracefully instead of failing mysteriously.

** What Comes Next

In the next chapter, we will deal with reality:

Sensors are late.
Measurements arrive out of order.
Time does not cooperate.

We will build time buffers and interpolation so our estimator can reason about the past safely.

Estimation without time alignment is fiction.