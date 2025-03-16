+++
title = "Automating my in-floor heating - scripts"
summary = "How to run Javascript on the Shelly 1PM to control the temperature"
date = 2025-03-12
tags = ["Floor Heat", "IOT"]
+++

Be sure to [read the intro](/posts/iot-floor-heat-intro) and [wiring overview](/posts/iot-floor-heat-wiring) posts 
first.

## Shelly Scripts and MQTT Support

When I first started this project I assumed that the IOT device would simply measure the resistance of the sensor 
and update Home Assistant with the current value. Home Assistant could then have automations to turn ON/OFF the 
relay based on the current value. Then I learned about 
[Shelly Scripts](https://shelly-api-docs.shelly.cloud/gen2/Scripts/ShellyScriptLanguageFeatures). 

Recent Shelly devices provide a mechanism to upload and run Javascript scripts on the device itself. This provides 
the benefit of allowing the control mechanism for the floor heat to run on a single device. When building 
distributed services over the years, I've learned about always planning for failure. While my home automation system 
is reasonably stable, we do experience power outages, network/DNS issues, human error, etc. By running our 
automation logic on the Shelly device itself we protect ourselves from everything other than a power outage (which 
would prevent the floor heat from working anyway).

Shelly devices can also easily integrate with [MQTT](https://en.wikipedia.org/wiki/MQTT) for asynchronous pub/sub of 
home automation data. This allows the for the home automation system to manage the more complex logic around time of 
day rules, home presence detection, reporting and monitoring, etc. 

## Roles and Responsibilities

The Shelly script is responsible for:
* measuring the resistance from the floor sensor and calculating the current temp
* comparing the current temperature to a target temperature
* turning ON/OFF the relay to power the floor heat when actual temp < target temp 
* receiving updates from Home Assistant for the target temperature via MQTT
* publishing updates of the actual floor temp and relay state to Home Assistant via MQTT

Home assistant is responsible for:
* setting the target floor temperature based on the current time of day, time of year, are people home, etc
* tracking and reporting historical power usage

## Script

You can review the full Shelly script below. The script has a main control loop that runs once a minute where it 
checks the temperature and compares it with the target. You'll notice when reviewing the code that we try to be as 
defensive as possible. We never turn on the relay indefinitely - at most it should only ever stay on for 15 minutes 
(`CONFIG.maxOnSeconds`) unless the script continues to run and extends the auto-off value of the relay. If we 
haven't received an update from Home Assistant in 60 minutes we assume something is wrong and turn off the relay.
If anything goes wrong - even an error in the script itself - we should just turn off the floor heat and 
wait for someone to notice and resolve the issue. This is certainly better than the relay staying on perpetually.

Note that the constants for `A`, `B`, and `C` in the `calcTemp` function are for the NTC sensor in my floor heat. 
Other sensors might differ but they can be calculated using the 
[Steinhart-Hart equation](https://rusefi.com/Steinhart-Hart.html).

{{< gist tfasz 5c7ce04cd281107738278e40844ef5a8 >}}

## Next

Next read about [how to build Home Assistant automations](/posts/iot-floor-heat-automation) to control floor temps.
