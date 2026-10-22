# 04 - Smart Tank Level Alarm: Real Low Level vs Sensor Fault

A rule-based diagnostic that decides whether a low tank level reading is a **real process event** or a **sensor fault**, so operators are not alarmed (or misled) by unreliable data.

## The Problem

In oil and gas facilities, a level transmitter can fail in ways that look like process events. A standard low-level alarm cannot tell the difference:

- A **real low level** is a gradual, physically plausible decline.
- A **sensor fault** is a sudden impossible jump, or a reading that stops moving entirely.

Acting on a faulty reading (for example, tripping a pump on a false low level) costs production. Ignoring a real low level is a safety and equipment risk.

## Approach

Synthetic tank level data (1000 one-minute samples) was generated with four behaviors: normal operation, a gradual real drop, a spike fault, and a stuck-sensor fault. Each minute is then classified from three indicators:

| Indicator | How it is computed | What it reveals |
|---|---|---|
| Rate of change | first difference of level | Physically impossible jumps (spike fault) |
| Smoothed trend | slope of 20-sample rolling mean | Gradual decline consistent with a real drawdown |
| Flatness duration | rolling std below 0.05 for 30+ minutes | Frozen reading (stuck sensor) |

Classification rules, in priority order (later rules override earlier ones):

1. `normal` - default
2. `real_low` - level below the 50% alarm threshold
3. `sensor_stuck` - flat for 30+ minutes (also back-filled 8 minutes to cover rolling-window delay)
4. `sensor_spike` - rate of change above 5 %/min

**Key design choice:** a sensor fault overrides a low-level alarm. If the reading itself is untrustworthy, the operator should be told "check the instrument", not "tank is low".

## Results

| State | Samples |
|---|---|
| normal | 561 |
| real_low | 366 |
| sensor_stuck | 70 |
| sensor_spike | 3 |

All injected faults were detected, and the real gradual decline was correctly kept separate from them.

## Limitations and Practical Notes

- The rolling window means detection is delayed in live use. A shorter window detects faster but raises false-alarm risk. In a real system, window size should be tuned per instrument.
- The back-fill step is valid for after-the-fact analysis and reporting, not for real-time alarming.
- Thresholds (5 %/min, 0.05 std, 30 min) must be set from real tank dimensions, flow rates, and sensor noise levels.
- Data is simulated. Next step is validating against real historian data.

## Practical Applications

- Reducing nuisance alarms and false trips from failed level transmitters
- Instrument health reporting for maintenance planning
- A first diagnostic layer before moving to ML-based anomaly detection

## Tools

Python, NumPy, pandas, Matplotlib (Google Colab)

## Author

Mustafa Alsaeedy - Instrumentation and Automation Engineer (oil and gas)
