Here's a draft formulation. Nothing is built yet; the full version is in the plan file at /Users/suryanarayanan/.claude/plans/this-repo-contains-data-modular-engelbart.md.

  The three sensors

  I'd use the three nw cameras. They all face westbound Bridge St traffic, which flows 13 → 12 → 9:

  ┌─────────────────┬──────────────────┬────────────────────────────┐
  │                 │      Sensor      │ Sees traffic arriving from │
  ├─────────────────┼──────────────────┼────────────────────────────┤
  │ s1 (upstream)   │ highschool_13 nw │ 23                         │
  ├─────────────────┼──────────────────┼────────────────────────────┤
  │ s2              │ corbins_12 nw    │ 13                         │
  ├─────────────────┼──────────────────┼────────────────────────────┤
  │ s3 (downstream) │ frantz_9 nw      │ 12                         │
  └─────────────────┴──────────────────┴────────────────────────────┘

  They watch the same flow about 30 s apart:
  - Their 5-minute counts correlate at 0.93–0.97.
  - 57–68% of vehicles can be matched one-to-one between neighbors.
  - Eastbound is messier: 9's camera mixes two incoming roads.

  Input

  A 3-channel count stream Y(t) = [y₁, y₂, y₃]: new vehicles per minute seen by each camera, in local time. More channels (exit predictions, re-identified vehicles, travel time) can be added later.

  Redundancy

  Downstream counts should follow upstream counts with a lag. That gives three pairwise checks: (1,2), (2,3) and (1,3). A fault in one sensor breaks exactly the two checks it's part of:
  - s1 faulty: (1,2) and (1,3) break, (2,3) holds.
  - s2 faulty: (1,2) and (2,3) break, (1,3) holds.
  - s3 faulty: (2,3) and (1,3) break, (1,2) holds.
  - All three silent at once: the cause is upstream of the sensors (central system or communication). We can detect it, but we can't blame one sensor.

  That pattern is what answers "which sensor."

  Anomaly types

  - No data: a sensor reports zeros while its neighbors say traffic is flowing.
  - Wrong signal: a sensor reports data that disagrees with its neighbors (too few, too many, or shifted in time).

  Algorithm and confidence

  In each 5–10 minute window, the algorithm scores how well the data fits each explanation: "all healthy", "s1 is faulty", "s2 is faulty", "s3 is faulty", or "all three are silent". Each "faulty"
  explanation is split into no-data and wrong-signal.

  The probability assigned to each explanation is the confidence score. Smoothing over time gives clean start and end times instead of flickering. To make "probability of a correct detection" honest, we
  check calibration: detections scored around 0.8 should be right about 80% of the time.

  Work week

  I'd use Mon 10-12 → Fri 10-16:
  - Monday to Wednesday are healthy, so they serve as "normal" reference days.
  - On Thu 10-15 at 20:02 local, s3 goes silent while s1 and s2 stay healthy. That's a real no-data event to test against.

  All real events we found are no-data. Testing wrong-signal will need faults injected into healthy days.

  Decisions for you

  1. Westbound nw triple, or another direction?
  2. Week 10-12 → 10-16, or another?
  3. Start with no-data only, or both classes from the start?
  4. This window-by-window probability approach, or a simpler voting baseline first?
  5. 1-minute bins with a 5–10 minute decision window, or coarser?

  I'm not asking for approval to build. React to whichever parts you want to change and I'll revise the formulation.