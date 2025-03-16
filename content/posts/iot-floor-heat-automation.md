+++
title = "Automating my in-floor heating - automations"
summary = "Using Home Assistant to Publish Target Temperatures and Display Dashboards"
date = 2025-03-15
tags = ["Floor Heat", "IOT"]
+++

Be sure to [read the intro](/posts/iot-floor-heat-intro),  [wiring overview](/posts/iot-floor-heat-wiring), 
and [how to write script](/posts/iot-floor-heat-script) posts first.

## Home Assistant Automations

I use Home Assistant broadly for all home automation. It knows who is home based on which phones are connected to 
the WiFi, it knows the time of day and time of year, it has a great dashboard of controls for our home devices, etc. 
This all makes it a great place to keep the automations for the floor heat. 

## PyScript

Home Assistant provides a number of mechanisms to build automations. I've recently switched to using 
[Pyscript](https://github.com/custom-components/pyscript) because 
I find it much easier to write code than create automations in the UI or via YAML. The examples below use Pyscript but 
you can easily accomplish the same thing via standard automations.

## Tracking Target Temps with Helpers

Home Assistant helpers let you configure variables which you can then use in automations, on dashboards, etc. I've 
set up three helpers for each floor-heat zone - `input_boolean.floor_heat_[ZONE]_enabled`, `input_number.floor_heat_
[ZONE]_target`, and `input_number.floor_heat_[ZONE]_temp`. 

An automation runs at specific times of day - 6AM, 10AM, etc. and sets our target temps based on temperature presets,
if we are home or not, if floor heat is enabled, etc.
```python
MORNING = "06:00"
MIDDAY = "10:00"
...

TEMP_AWAY     = 0
TEMPS_MORNING = {BASEMENT: 65, UPSTAIRS: 70}
TEMPS_DAY     = {BASEMENT: 60, UPSTAIRS: 65}
...

@time_trigger(f"once({MORNING})")
def set_temp_morning():
    set_temp(TEMPS_MORNING)

@time_trigger(f"once({MIDDAY})")
def set_temp_midday():
    set_temp(TEMPS_DAY)

def set_temp(temps):
    # If we are away - just leave at away temps
    if input_boolean.home_mode == "off":
        input_number.floor_heat_basement_target.set_value(TEMP_AWAY)
        input_number.floor_heat_upstairs_target.set_value(TEMP_AWAY)
    else:
        if input_boolean.floor_heat_upstairs_enabled == "off":
            input_number.floor_heat_upstairs_target.set_value(TEMP_AWAY)
        else:
            input_number.floor_heat_upstairs_target.set_value(temps[UPSTAIRS])

        if input_boolean.floor_heat_basement_enabled == "off":
            input_number.floor_heat_basement_target.set_value(TEMP_AWAY)
        else:
            input_number.floor_heat_basement_target.set_value(temps[BASEMENT])
```

Then an automation runs every 5 minutes (or if target values change) to publish the target 
temperatures via MQTT.
```python
import json

@time_trigger("cron(*/5 * * * *)")
@state_trigger("input_number.floor_heat_upstairs_target")
@state_trigger("input_number.floor_heat_basement_target")
def publish_floor_heat_config():
    msg = { "iot-basement-floor-heat": { "targetTemp": int(float(input_number.floor_heat_basement_target)) },
            "iot-upstairs-floor-heat": { "targetTemp": int(float(input_number.floor_heat_upstairs_target)) } }
    mqtt.publish(topic="iot-config/floor-heat", payload=json.dumps(msg))
```

## Receiving Updates from MQTT

Since the Shelly device script publishes it's status once a minute via MQTT we can stay up to date with the most recent
actual temp values. 

```python
@mqtt_trigger("iot-basement-floor-heat/status")
def basement_status(**data):
    update_status("input_number.floor_heat_basement_temp", data)

@mqtt_trigger("iot-upstairs-floor-heat/status")
def upstairs_status(**data):
    update_status("input_number.floor_heat_upstairs_temp", data)
    
def update_status(name, data):
    attrs = {"device_target": round(data["payload_obj"]["target_f"])}
    state.set(name, value=round(data["payload_obj"]["temp_f"]), new_attributes=attrs)
```

## Home Assistant Dashboard

The current values then can easily be displayed on the dashboard.

![Dashboard data](data.png)

You can also graph the temps, power draw, and total energy usage.

![Dashboard history](history.png)

