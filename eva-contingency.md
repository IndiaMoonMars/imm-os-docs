# EVA contingency: loss of signal (LOS)

The health monitor watches every crew member on EVA. Contact means the newest live frame of
either suit vitals (`habitat/eva/biosensors/<crew>`, 5 Hz) or fused position
(`habitat/eva/position/<crew>`, 5 Hz).

## Arming

- **Automatically:** on the first live, real frame from a crew member.
- **By plan:** when an EVA plan naming them goes IN_PROGRESS.
- **Manually:** "Start EVA monitoring" on the Health tab (any crew or MCC role).
- **Disarming** stops LOS monitoring. It happens when the plan is COMPLETE or ABORTED, or by "EVA ended"
  (MCC operator or commander only).
- Armed crew and their state survive a health-monitor restart.

## States

Thresholds are configurable (`EVA_LOS_*` in the infra `.env`). The measured times are from the
mission readiness test, VV-EVA-02.

| State | After no contact for | Alarm | Measured | Action |
|---|---|---|---|---|
| NOMINAL | – | – | – | – |
| LOS_WARN | 10 s | caution | 10.6 s | check comms; try voice |
| LOS | 30 s | warning | 30.3 s | start the comm-loss procedure; buddy checks in |
| CONTINGENCY | 120 s | emergency (mode EMERGENCY, EVA NO-GO) | 120.7 s | dispatch buddy / IV crew to the last position |

**Partial loss.** Vitals missing for ≥ 15 s while position still arrives, or the reverse,
raises a caution. Suspect a suit sensor or radio.

## What the MCC sees during LOS

- the last known position (UWB x/y or GPS);
- a **search radius** that grows with time: the speed seen before the loss (at least walking pace,
  1.4 m/s), capped at 2 km;
- the last vitals and the time since contact;
- with a simulated Mars delay: the data age as well. With minutes of delay the habitat crew
  owns the response, and the MCC supports.

## Recovery

- When live frames return, the crew member is NOMINAL again and the LOS alarm clears. It
  closes when acknowledged.
- The outage is recorded with its duration and worst state.
- **Suit backfill.** The suit's store-and-forward readings for the outage arrive marked
  delayed. They are stored at their own time and counted against that outage ("N readings filled in"),
  even when they arrive before the first live frame. They never raise a live alarm.

## Vitals limits (while armed)

| Vital | Warning | Emergency |
|---|---|---|
| Heart rate | > 160 bpm | > 185 or < 40 bpm |
| SpO₂ | < 94 % | < 90 % |
| Skin temperature | > 38.5 °C (caution < 30 °C) | > 39.5 °C |
