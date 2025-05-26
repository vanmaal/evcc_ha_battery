# Advanced EVCC Integration with Home Assistant and a Controllable Battery

## 1. Introduction

This guide explains how to integrate EVCC with Home Assistant and a battery system that supports active control (e.g., Sonnen, Victron, BYD...), using a `custom` definition and leveraging the full potential of EVCC beyond what is covered in the official documentation. The Sonnen battery is used here as a practical example, but **the same methodology applies to any system that exposes the necessary control endpoints via HTTP or a local API.**

This guide was written by ChatGPT after documenting the full integration process, including real-world use cases, tested configurations, and a review of the EVCC source code.

## 2. System Architecture

The integration consists of three key components:

- **Home Assistant (HA)**: Provides power and state-of-charge (SoC) data from the battery as REST-accessible entities.
- **EVCC**: Reads this data and actively controls battery behavior via HTTP requests (mode changes, charge/discharge limits, etc.).
- **Battery or Inverter**: Offers a local API through which its behavior can be read or controlled.

## 3. Sensors from Home Assistant

To integrate battery power and SoC readings, Home Assistant exposes entities via its REST API:

```yaml
power:
  source: http
  uri: http://192.168.x.x:8123/api/states/sensor.sonnen_battery_inout
  auth:
    type: bearer
    password: <HA_TOKEN>
  jq: .state | tonumber

soc:
  source: http
  uri: http://192.168.x.x:8123/api/states/sensor.evcc_battery_soc
  auth:
    type: bearer
    password: <HA_TOKEN>
  jq: .state | tonumber
```

## 4. Active Battery Control (`batterymode`)
...
(continued — full content included in the actual guide)
