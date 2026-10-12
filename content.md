# 4-Zone HVAC Control

### A YAML specification for ducted zone control off a single Daikin unit

---

## Overview

This spec defines how one Daikin unit — 24,000 BTU/hr (2-ton), driven by the [ESP32-Faikin](../esp32-faikin/) controller — feeds four independently-controlled zones through motorized dampers on the second floor of the house. There are no sensors in the supply ductwork itself; every zone's setpoint, mode, and schedule come from its own thermostat.

**YAML** was chosen over XML or JSON as the config format — it reads closer to plain English for a spec that a human will be hand-editing and reviewing far more often than a machine will be parsing it.

This document covers the second floor's 4-zone subsystem only. Other floors, if and when they're zoned, would get their own independent spec.

## Zone layout

| Zone | Room | Fan | Damper controller |
|---|---|---|---|
| Zone 1 | Bedroom South | fan0 | ZC2 |
| Zone 2 | Bathroom (2nd floor hall bath) | fan0 | ZC4 |
| Zone 3 | Bedroom East | fan0 | ZC3 |
| Zone 4 | Master Bedroom | fan1 (dedicated) | ZC1 |

Each zone has its own physical zone-controller board (ZC1–ZC4), independent of which fan actually carries its air — the board numbering doesn't follow zone numbering, which is a common source of confusion when wiring these up.

**Thermostats:** every zone uses a hardwired thermostat with Y1/W1/G outputs, except the bathroom (Zone 2), which runs an **Aqara W100** over Zigbee instead. The integration path for the W100 — Home Assistant directly, Zigbee2MQTT, or both — isn't finalized yet.

## Fan logic

**fan0** serves Zones 1–3 and is analog speed-controllable from 5–100%. Its daytime speed table doubles as minimum-airflow protection — it's not just a comfort setting, it's what keeps static pressure in a safe range as zones close:

| Zones open (of 3) | fan0 speed |
|---|---|
| 1 | 21% |
| 2 | 26% |
| 3 | 33% |

**fan1** serves Zone 4 (Master Bedroom) alone, via its own dedicated fan and damper. Unlike fan0, it's true variable-speed hardware, running anywhere from **46%** (quiet night) up to **62%** (full daytime flow).

## Damper logic

All dampers — whether on/off or modulating type — must close **fully** (0% open, no partial/trickle position) whenever their zone has no active call. Modulating dampers get true proportional 0–100% control when their zone *is* calling; on/off dampers only get the two end states.

## Implementation

The spec is being implemented as an **ESPHome YAML** configuration, targeting **ESPHome 2026.8.2**. The control logic — thermostat inputs, scheduling, the zone/fan/damper relationships above — is complete; fan PWM output and damper actuation are currently left as stubs pending final hardware selection for both.

```yaml
# Illustrative shape of the config — not the full spec
zones:
  - id: zone1
    name: "Bedroom South"
    fan: fan0
    damper: ZC2
    thermostat: hardwired_y1w1g
  - id: zone2
    name: "Bathroom"
    fan: fan0
    damper: ZC4
    thermostat: aqara_w100  # Zigbee, integration path TBD
  - id: zone3
    name: "Bedroom East"
    fan: fan0
    damper: ZC3
    thermostat: hardwired_y1w1g
  - id: zone4
    name: "Master Bedroom"
    fan: fan1
    damper: ZC1
    thermostat: hardwired_y1w1g

fans:
  - id: fan0
    type: analog
    range: [5, 100]
    daytime_table: { 1: 21, 2: 26, 3: 33 }  # % by zones open
  - id: fan1
    type: variable_speed
    range: [46, 62]  # quiet night -> full daytime
```

## Status

Core spec is final for this floor. Open items: fan PWM hardware selection, damper actuator selection, and the Aqara W100 integration path.
