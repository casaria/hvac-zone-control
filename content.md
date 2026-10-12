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

## Homeowner Advisory

**Document:** Advisory for the new homeowner (source: [HOMEOWNER-ADVISORY.md](https://github.com/casaria/hvac-zone-control/blob/main/HOMEOWNER-ADVISORY.md))

> **Status:** Release 1.0 2026-09-28 .
Draft, started 2026-09-27. Items marked **(To confirm)** still need the current owner's input.


### 1. About this document
This house has a custom controller that runs the heating and cooling on both floors. It works with the ordinary wall thermostats, so day to day nothing changes for you. This advisory explains:
- what is installed
- which behaviours are normal
- what to do when something seems wrong
- what needs to be handed over when the house changes hands

### 2. What's installed

#### Main components overview
- **Multi-zone controller (PLC1):** the controller box that runs heating and cooling on both floors. Location: **ON YELLOW WALL UNDERNEATH THE HONEYWELL ER VENTILATOR**
- **AC1 interface controller (Faikout) + fan speed detector/emulator:** a small Wi-Fi module on the Floor 1 Daikin heat pump (AC1). It passes PLC1's on/off, mode and temperature commands to the heat pump and reports the unit's status back. Its fan speed detector/emulator reports the heat pump's fan speed, which PLC1 uses to spot a slow-running fan. Location: **(ON SMALL WALL SECTION APP 2FT ABOVE THE FLLOR)**
- **AC2 interface controller (Faikout) + fan speed detector/emulator:** the same as the AC1 interface, but for the Floor 2 Daikin heat pump (AC2). Location: **(NEXT TO AC1)**
- **AC1.O outdoor unit:** the Daikin heat pump outdoor unit for Floor 1, 24,000 BTU/hr (2 ton). Model number: **(ON PROPERTY FENCE WALL)**
- **AC2.O outdoor unit:** the Daikin heat pump outdoor unit for Floor 2, 24,000 BTU/hr (2 ton). Model number: **(ON PROPERTY FENCE WALL)**
- **FAN0 in-duct fan:** a speed-controlled (PWM) fan in the ductwork that moves up to 850 CFM to Floor 2 Zones 1–3. PLC1 raises its speed as more of those zones call. Location and model: **(NEXT TO GAS FURNACE)**
- **FAN1 in-duct fan:** a speed-controlled (PWM) fan in the ductwork that moves up to 850 CFM to Floor 2 Zone 4 (master bedroom). It runs whenever Zone 4 is calling. Location and model: **(NEXT TO GAS FURNACE))**

##### Zone controllers
- **ZC0:** the wall thermostat for Floor 1 (Zone 0). It controls heating, cooling and fan for the whole floor. Model and location: **(corner of elcove)**
- **ZC1:** the wall thermostat for Floor 2 Zone 1. Rooms, model and location: **(2nd bedroom)**
- **ZC2:** the Aqara W100 wireless thermostat for Floor 2 Zone 2 (bathroom). It works like a regular thermostat, with auto, heat and cool modes, a setpoint and 4 fan speeds(ignored). It connects wirelessly (Zigbee) and runs on batteries **((2) CR2450 >> 2 YEARS RUNTIME)**. Location: **(PORTABLE UNIT TO REMAIN AT LOCATION IF ZC/ Thermosat enbabled)**
- **ZC3:** the wall thermostat for Floor 2 Zone 3. Rooms, model and location: **(1st bedroom upstairs)**
- **ZC4:** the wall thermostat for Floor 2 Zone 4 (master bedroom). Model and location: **(master bedroom, labeled incorrectly ZC1)**

#### Floor 1: one zone
- **Heat pump:** one Daikin unit ("AC1").
- **Gas furnace.**
- **One wall thermostat** controls the whole floor. There are no dampers on this floor.
- **Heat source is selectable:** **Gas** (the default), **Heat pump**, or **Combined**.
- **Cooling** always uses the heat pump.
- **Window sensor:** a wireless sensor on a Floor 1 window can pause heating and cooling while the window is open. This lockout is **switched off by default**.

#### Floor 2: four zones
One Daikin heat pump ("AC2") serves the whole floor. Each zone has a motorised damper in the ductwork that opens only when that zone asks for air. Two booster fans push air to the zones.

| Zone | Rooms | Thermostat | Notes |
|---|---|---|---|
| Zone 1 | **(2nd Bedroom)** | Wall thermostat | Counts double when rooms disagree (see section 3) |
| Zone 2 | Bathroom | Aqara W100 wireless thermostat | Auto, heat or cool |
| Zone 3 | **(1st Bedroom)** | Wall thermostat | |
| Zone 4 | Master bedroom | Wall thermostat | Its damper is slower to reach target opening, taking about 90 seconds to open fully |

#### Controls
- **The controller (PLC1):** a small electronics box, shown in Home Assistant as **MULTI_Z_MK_II**. It reads every thermostat, drives the dampers, fans and furnace, and sends commands over Wi-Fi to each Daikin unit. Location: **(under Rockwell ventilator)**
- **Home Assistant Yellow:** the home-automation box. You can use its app to view the system and change settings. The box itself must stay on, because it carries the messages between PLC1 and the heat pumps (see section 6).
- **Air-quality sensor:** part of the controller. It reports an indoor air-quality index.
- **Status light:** a coloured light on the controller that shows what Floor 1 is doing (see section 4).

#### What makes this system different
- **Self-contained logic on one platform.** Every microcontroller (PLC1, AC1 IF, AC2 IF) is an ESP32-S3 running software built with ESPHome. All heating and cooling decisions are made inside the controllers' own code, not in Home Assistant, so no Home Assistant automations are needed. The Home Assistant Yellow only carries messages between the controllers (MQTT and Zigbee2MQTT) and gives you a place to view and change settings (see section 6).
- **Shared heat pump, fair voting.** The four upstairs zones share one heat pump. When rooms disagree between heating and cooling, the larger demand wins, weighted by each room's priority. the weights are adjustable from the dashboard.
- **Compressor protection.** Floor 2 waits at least 4 minutes before switching between heating and cooling.
- **Nothing wasted at the end of a cycle.** When heating or cooling stops, fans and dampers keep running for about 80 seconds to push the remaining heat or cold into the rooms.
- **Self-calibrating dampers.** Every start-up drives each damper fully open and then fully closed, so the controller always knows where they are.
- **Airflow that follows demand.** Upstairs fan speed rises with the number of rooms calling, and the fan power is switched off completely when no fan is needed (saving standby electric consupmtion).
- **Hybrid heating downstairs.** Floor 1 can heat with gas, the heat pump, or both. Gas keeps working even if the network is down.
- **Self-monitoring heat pumps.** If a Daikin unit's fan runs too slowly, the controller restarts that unit and raises an alert.

### 3. Day-to-day use
- **Thermostats:** set temperatures on the wall thermostats as you would in any house. The controller takes care of the rest.
- **Floor 2 shares one heat pump,** so the floor can heat *or* cool, not both at once. If some rooms want heat and others want cooling, the larger demand wins, weighted by each room's priority setting. Zone 1 counts double by default. Rooms asking for the other mode wait their turn. You can change the priorities in Home Assistant.
- **Flr 2 Bathroom:** use the W100 like any thermostat: pick auto, heat or cool, and set the temperature. In auto it heats or cools as needed to hold the setpoint. Like the other upstairs rooms, it has to wait if the floor is busy in the other mode.
- **Floor 1 heat source:** change it in Home Assistant under **Floor 1 Heating Mode**. Gas mode keeps working even if Home Assistant is down (see section 6).

### 4. Normal behaviour that can look like a fault

| When | What you'll see | Why |
|---|---|---|
| After a power cut or restart | Floor 2 ignores the thermostats for about **3 minutes**. You may hear every damper open and then close. | Each damper is driven fully open and then fully closed so the controller knows exactly where it is. |
| After a power cut or restart | Both heat pumps switch off. | The controller turns both Daikin units off when it starts. They come back on by themselves as soon as a thermostat asks for heat or cooling. |
| When a room's heating or cooling stops | The upstairs fans keep running for about **80 seconds**, and that room's damper stays open. | This clears leftover heat or cold out of the unit. |
| When Floor 2 switches between heating and cooling | A wait of up to **4 minutes**. | This protects the heat pump's compressor. |
| Occasionally | A heat pump turns itself off and back on, and an alert appears in Home Assistant. | Its fan was reading too slow, so the controller restarted it. An occasional restart is fine. If it keeps happening, call for service. |

#### Status light (on the controller, shows Floor 1)
| Light | Meaning |
|---|---|
| Red, slow pulse | Floor 1 heating |
| Blue, slow pulse | Floor 1 cooling |
| Bright red pulse | Window open, so Floor 1 is paused (only when the window lockout is on) |
| Off | Idle |

### 5. If something seems wrong

| What you notice | Likely cause | What to do |
|---|---|---|
| No heat or cooling upstairs right after a power cut | The dampers are recalibrating | Wait 3–4 minutes. |
| One upstairs room gets no air while another does | The floor is serving the other mode, or is in its 4-minute wait | Check what the other upstairs thermostats are asking for, then give it a few minutes. |
| The heat pumps do nothing, but gas heat downstairs works | The Wi-Fi or the Home Assistant Yellow is down | Check that the Yellow is powered and running, then check the Wi-Fi router. Restart whichever is down. |
| Bathroom won't heat | W100 batteries or its wireless link | Check the W100 display and replace its batteries if needed. |
| Floor 1 stops when a window opens | The window lockout is switched on | Close the window. The setting is **Window Sensors Enabled** in Home Assistant. |
| No heating or cooling anywhere, and the internet is down too | Breaker circuit 10 has tripped | Reset circuit 10. Allow 3 minutes for the dampers to recalibrate. |
| Everything seems stuck | The controller needs a restart | Press **Restart-RESET** in Home Assistant, or turn the controller's power off and on. Allow 3 minutes for recalibration. |
| A Wi-Fi network called **Hvac-Control Fallback Hotspot** appears | The controller can't join your Wi-Fi | Usually the router's name or password changed. See section 6. |

### 6. What the system depends on
The system needs three things to run: **Wi-Fi**, **power**, and the **Home Assistant Yellow** box switched on. All the control logic is built into the controllers themselves. You don't need to use the Home Assistant app day to day, but the Yellow box must stay on, because it carries the messages between the controllers.

- **Wi-Fi (WLAN):** PLC1 and the two heat-pump interfaces (AC1 IF, AC2 IF) talk to each other over the home Wi-Fi.
- **Home Assistant Yellow:** it hosts the message service (the Mosquitto MQTT broker) that PLC1 uses to reach both heat pumps. It also runs Zigbee2MQTT, which connects the bathroom thermostat (ZC2) and the Floor 1 window sensor over Zigbee.
- **If Wi-Fi or the Yellow is down:**
  - The heat pumps won't respond to the thermostats, the bathroom thermostat goes silent, and the Floor 1 window lockout stops working.
  - These keep working, because PLC1 switches them directly: **Gas mode keeps heating Floor 1**, and the upstairs dampers and fans still follow the hardwired thermostats (ZC1, ZC3, ZC4).
- **Changing your router or Wi-Fi password:** the controllers have the Wi-Fi details built in. Have a technician update them and reinstall the controllers' software *before* you switch routers.
- **Power:**

| Supply | Feeds | Breaker / disconnect |
|---|---|---|
| 120 V | PLC1 enclosure: all 120 V and 24 VAC supplies inside it, which power the fans (FAN0, FAN1), the dampers and the zone controllers (ZC0–ZC4) | Breaker circuit **10** |
| 120 V | Network: modem, gateways, PoE switch and Home Assistant Yellow | Breaker circuit **10** (same circuit) |
| 240 V | AC1.O (Floor 1 outdoor unit) and AC1 IF | Fused disconnect, plus breaker circuits **38 and 39** |
| 240 V | AC2.O (Floor 2 outdoor unit) and AC2 IF | Fused disconnect, plus breaker circuits **37 and 40** |
| 120 V | Gas furnace (furnace receptacle) | Breaker circuit **10** (same circuit) |

> **Circuit 10 is critical.** It feeds the gas furnace receptacle, PLC1 and the whole home network. If it trips, the furnace itself loses power and heating and cooling control stops on both floors. Reset circuit 10 first whenever the whole system goes quiet.

### 7. Maintenance
- **Furnace filter — Honeywell F300 Electronic Air Cleaner (model F300E1019):** a whole-house electrostatic precipitator, 16×25 duct opening, 1400 CFM, mounted in the return duct upstream of the evaporator coil. Removes up to 98% of particles down to 3.0–10.0 microns. Carries a 10-year Clean Coil Guarantee.
  - **Electronic cells (FC37A1130, qty 2) and prefilters (203371, qty 2):** washable and reusable. Wash twice a year under normal use, more often with heavy use — a quick cleaning can be done anytime, and a full cleaning can wait for the yearly service since efficiency stays high as the cells load up. Wash the cells first, then the prefilters the same way, and make sure both are fully dry before reinstalling.
  - **Media post-filters (50000293-002, 16×12.5, qty 2):** consumable — replace every 6 months. Do not wash them; washing destroys the factory-applied electrostatic charge.
  - **Operating tip:** leave the air cleaner's own switch on at all times, and set the thermostat fan to "On" for continuous circulation.
- **Bathroom W100 and the window sensor:** replace batteries when their displays or Home Assistant show them low.
- **Seasonal:** check that each zone heats in autumn and cools in spring. Give the dampers the 3 minutes they need after any power cut.

### 8. Contacts
| Role | Name | Phone / email |
|---|---|---|
| HVAC contractor who knows this installation | **(Casaria technology, Inc.)** | Provided separately |
| Whoever maintains the controller and Home Assistant | **(home ownner)** | |
| Daikin dealer / service | **(all local HVAC contractors)** | |

### 9. Handover checklist
- [ ] Location of the controller box and its power supply
- [ ] Which rooms belong to Zone 1 and Zone 3
- [ ] Home Assistant administrator account transferred
- [ ] Controller passwords and Wi-Fi details, handed over securely
- [ ] Controller configuration files and change log handed over (kept separately from this document)
- [ ] Daikin model numbers for AC1 and AC2, and the service history
- [ ] Furnace model, filter size and replacement schedule
- [ ] Contacts in section 8 filled in
- [ ] Breakers and disconnects in section 6 labelled, including circuit 10 (furnace receptacle, PLC1 and network), 38/39 for AC1 and 37/40 for AC2


_Passwords and access details must be handed over separately. Never write them in this document._

## Status

Core spec is final for this floor. Open items: fan PWM hardware selection, damper actuator selection, and the Aqara W100 integration path.
