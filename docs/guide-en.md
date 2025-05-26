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

The `batterymode` key allows EVCC to dynamically change the battery's operating behavior. The following configuration mirrors the official `sonnenbatterie` template:

```yaml
batterymode:
  source: switch
  switch:
    - case: 1  # normal mode (self-consumption or TOU)
      set:
        source: http
        uri: http://192.168.x.x/api/v2/configurations
        method: PUT
        headers:
          - content-type: application/json
          - Auth-Token: <SONNEN_TOKEN>
        body: '{"EM_OperatingMode":"2"}'  # or "10" for TOU

    - case: 2  # hold: manual mode, no charging or discharging
      set:
        source: sequence
        set:
          - source: http
            uri: http://192.168.x.x/api/v2/configurations
            method: PUT
            headers:
              - content-type: application/json
              - Auth-Token: <SONNEN_TOKEN>
            body: '{"EM_OperatingMode":"1"}'
          - source: http
            uri: http://192.168.x.x/api/v2/setpoint/discharge/0
            method: POST
            headers:
              - content-type: application/json
              - Auth-Token: <SONNEN_TOKEN>
          - source: http
            uri: http://192.168.x.x/api/v2/setpoint/charge/0
            method: POST
            headers:
              - content-type: application/json
              - Auth-Token: <SONNEN_TOKEN>

    - case: 3  # charge: manual mode + forced charging
      set:
        source: sequence
        set:
          - source: http
            uri: http://192.168.x.x/api/v2/configurations
            method: PUT
            headers:
              - content-type: application/json
              - Auth-Token: <SONNEN_TOKEN>
            body: '{"EM_OperatingMode":"1"}'
          - source: http
            uri: http://192.168.x.x/api/v2/setpoint/discharge/0
            method: POST
            headers:
              - content-type: application/json
              - Auth-Token: <SONNEN_TOKEN>
          - source: http
            uri: http://192.168.x.x/api/v2/setpoint/charge/{{ .maxchargepower }}
            method: POST
            headers:
              - content-type: application/json
              - Auth-Token: <SONNEN_TOKEN>
```

## 5. Hourly Dynamic Tariffs

EVCC can optimize charging based on hourly energy prices from the market (e.g., OMIE), using a fixed markup and estimated access tariffs. This applies to any electricity provider offering such dynamic pricing.

### 5.1 Sensors in Home Assistant

To retrieve and process hourly price data into a format EVCC can use, the following sensors can be defined:

```yaml
# This REST sensor fetches hourly market prices from the OMIE API via REE.
# Data includes both today and tomorrow (available from 20:15 CET).
- platform: rest
  name: omie_preus_horaris
  resource_template: >
    {% set avui = now().date() %}
    {% set demà = (now() + timedelta(days=1)).date() %}
    https://apidatos.ree.es/es/datos/mercados/precios-mercados-tiempo-real?start_date={{ avui }}T00:00&end_date={{ demà }}T23:59&time_trunc=hour
  method: GET
  value_template: "OK"
  json_attributes_path: "$.included[0].attributes"
  json_attributes:
    - values
  scan_interval: 300
```

(*Note: The full Jinja template for forecasting will be translated in a later section or included in a Markdown file to preserve formatting*)

### 5.2 Integration with EVCC

Add the following block to `evcc.yaml` to connect EVCC with the forecast sensor:

```yaml
tariffs:
  - name: custom
    type: custom
    forecast:
      source: http
      uri: http://192.168.x.x:8123/api/states/sensor.evcc_tarifa_forecast_real
      auth:
        type: bearer
        password: <HA_TOKEN>
      jq: .attributes.forecast
```

EVCC automatically handles optimal charging periods based on the forecast data. **You do not need additional sensors in Home Assistant to calculate the lowest price or ideal charging hours—EVCC manages this internally as long as the forecast is properly formatted.**

Note: This forecast is only an estimate based on market prices, a fixed margin, and expected tariffs. It is intended for planning, not for replicating exact billing amounts.

## 6. Conclusion

EVCC is capable of much more than its documentation suggests. With a custom definition, it’s possible to integrate Home Assistant readings and actively control the battery via its API. The Sonnen example shown here demonstrates an approach that can be adapted to any system exposing the required controls, such as charge/discharge limits or operating modes.

In addition, EVCC can plan charging based on dynamic tariffs using forecast data built from market prices and estimated costs. This functionality enables meaningful cost optimization even with non-static rates.