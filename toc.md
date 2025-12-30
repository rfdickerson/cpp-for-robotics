# 📘 C++ for Robotics — Project-Driven Edition

## Core project
Build a safe, deterministic, diff-drive robot control stack in modern C++.

- No magic frameworks.
- No hidden state.
- No “just trust ROS”.

By the end, the reader has:

- A reusable C++ estimation + control library
- A thin ROS 2 integration
- A simulation harness
- Explicit assumptions and safety boundaries

## Repo map (for writing)
- Chapters live in `book/`
- Site assets live in `assets/`

## Status legend
- ✅ Done (exists in `book/`)
- 🟡 Drafting
- ⬜ Planned

## Writing plan / Table of contents (mapped to weeks)

### Part I — Foundations (Weeks 0–1)

#### Chapter 1 — Why Robotics Is Different (✅ done)
- File: `book/01-why-robotics-is-different.md`
- Purpose:
	- Reset expectations
	- Explain stakes
	- Establish philosophy
	- No code. Pure mindset.

#### Chapter 2 — The Mental Shift (From Engines to Embodied Systems) (⬜ planned)
- Proposed file: `book/02-the-mental-shift.md`
- Purpose:
	- Contrast games vs robotics vs distributed systems
	- Introduce state, time, uncertainty as first-class ideas
	- Prepare reader for estimation/control thinking
	- Light code snippets, no full modules yet.

#### Chapter 3 — Time Is a First-Class Type (✅ done)
- File: `book/03-time-is-a-first-class-type.md`
- Project tie-in:
	- Everything in the estimator and controller depends on time
	- Introduce Seconds, fixed-step contracts, timestamped data
	- Sets up Week 3 directly.

#### Chapter 4 — Units or It Didn’t Happen (⬜ planned)
- Proposed file: `book/04-units-or-it-didnt-happen.md`
- Project tie-in:
	- Pose estimation and control will use meters, radians, velocities
	- Prevent dimensional bugs before they happen
- Deliverables:
	- Units library (Seconds, Meters, Radians, velocities)
	- Angle wrapping utilities

### Part II — State Estimation (Weeks 1–3)

#### Chapter 5 — What It Means to Know Where You Are (⬜ planned)
- Proposed file: `book/05-what-it-means-to-know-where-you-are.md`
- Project: EKF Pose Estimator
- Purpose:
	- Explain why robots never know their pose
	- Prediction vs correction
	- Sensors as noisy processes, not truth
	- No full EKF yet — concept + architecture.

#### Chapter 6 — Building an EKF Pose Estimator (Library-First) (✅ done)
- File: `book/06-building-an-ekf-pose-estimator.md`
- Week 1–2 project
- Deliverables:
	- ROS-agnostic C++ EKF library
	- SE(2) pose: (x, y, θ)
	- Wheel encoder + IMU fusion
	- Explicit covariance handling
- Key lessons:
	- State lives in one place
	- Updates are explicit
	- Time-stamped measurements
	- Deterministic math
- Note: This chapter is a huge differentiator for the book.

#### Chapter 7 — Time Buffers and Delayed Reality (⬜ planned)
- Proposed file: `book/07-time-buffers-and-delayed-reality.md`
- Week 3 project
- Deliverables:
	- Time-buffered state history
	- Interpolation utilities
	- Handling delayed / out-of-order measurements
- Why this matters:
	- Real robots never deliver data “now”
	- Controllers and estimators operate at different rates
	- This chapter connects estimation ↔ control cleanly.

### Part III — Control (Weeks 4–5)

#### Chapter 8 — Control Systems for Software Engineers (⬜ planned)
- Proposed file: `book/08-control-systems-for-software-engineers.md`
- Purpose:
	- PID as a state machine
	- Noise, derivative kick, windup
	- Why control is not just math
	- Light code, heavy intuition.

#### Chapter 9 — Differential Drive, Done Right (⬜ planned)
- Proposed file: `book/09-differential-drive-done-right.md`
- Week 4 project
- Deliverables:
	- Diff-drive kinematics
	- Go-to-goal controller
	- Heading PID + velocity gating
	- Saturation + slew rate limiting
- Key payoff:
	- Units + time + PID all come together
	- Safe commands only
	- Bounded behavior everywhere
- Note: This is the first chapter where the robot moves.

#### Chapter 10 — Wrapping Control in a Thin ROS 2 Node (⬜ planned)
- Proposed file: `book/10-wrapping-control-in-a-thin-ros2-node.md`
- Week 5 project
- Purpose: Show ROS as plumbing, not architecture.
- Deliverables:
	- Minimal ROS 2 node
	- Message ↔ C++ boundary
	- No control logic in callbacks
	- Deterministic update loop
- Note: This chapter teaches how not to abuse ROS.

### Part IV — Safety, Simulation, and Reality (Weeks 6–8)

#### Chapter 11 — Simulation as a Safety Tool (⬜ planned)
- Proposed file: `book/11-simulation-as-a-safety-tool.md`
- Week 6 project
- Deliverables:
	- Simple kinematic simulator
	- Noise injection
	- Latency simulation
	- Dropped messages
- Key idea: Simulation is not for realism. It’s for failure.
- Note: Perfect for readers with game/graphics backgrounds.

#### Chapter 12 — Failure Modes and Defensive Design (⬜ planned)
- Proposed file: `book/12-failure-modes-and-defensive-design.md`
- Purpose:
	- What happens when:
		- Sensors freeze
		- Time jumps
		- Commands stop
		- Estimates diverge
- Deliverables:
	- Watchdogs
	- Robot modes
	- Fail-safe defaults
- Note: This is where the book earns serious credibility.

#### Chapter 13 — Tuning, Profiling, and Assumptions (⬜ planned)
- Proposed file: `book/13-tuning-profiling-and-assumptions.md`
- Week 7–8 project
- Deliverables:
	- Gain tuning methodology
	- Profiling control loops
	- Documented assumptions:
		- Rates
		- Noise models
		- Limits
		- Failure responses
- Key idea: Good robotics systems are explicit about what they assume.

### Part V — Closing the Loop

#### Chapter 14 — What You’ve Built (and Where to Go Next) (⬜ planned)
- Proposed file: `book/14-what-youve-built-and-where-to-go-next.md`
- Purpose:
	- Reflect on the full system
	- Map this foundation to:
		- SLAM
		- MPC
		- Manipulation
		- Autonomy stacks
	- Encourage readers to extend, not restart.