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

```javascript 
/**
 * This script runs on the Shelly relay and turns the relay ON/OFF based on input temperature from the sensor.
 * In order to keep the logic here simple, the target temperature is set in Home Assistant based on the time of
 * day, away mode, etc. and published via MQTT. If anything goes wrong we should just turn off the relay to
 * ensure we don't leave the floor-heat on perpetually.
 *
 * Examples of failure cases are:
 *  - can't read sensor or sensor reading out of expected range
 *  - haven't heard update on target temp from HASS in more than 60m
 *  - we have an error in our main loop and can't update the switch - it will auto turn off in 15m
 *
 * We also publish a message on every poll with the current target temp, actual temp, voltage, resistance, etc.
 * This is consumed by Home Assistant to display information in the dashboard.
 *
 * Combined examples from:
 *  - https://shelly.guide/underfloor-heating-with-devireg-and-ntc-sensors/
 *  - https://github.com/ALLTERCO/shelly-script-examples/blob/main/ntc-conversion.js
 */

let CONFIG = {
  configTopic: "iot-config/floor-heat", // MQTT topic to listen to for config changes
  hysteresis: 2,         // turn switch ON/OFF when we are outside buffer (targetTemp +/- hysteresis)
  lastTempUpdate: 0,     // track the last time we received a target temp - assuming something is wrong if > 60m
  minVoltage: 4,         // min voltage expected (~92 F) - we assume something is wrong with sensor if below this threshold
  maxOnSeconds: 900,     // max duration we turn switch on for to ensure it won't stay on even if this program crashes
  maxVoltage: 8,         // max voltage expected (~30 F) - we assume something is wrong with sensor if above this threshold
  name: Shelly.getDeviceInfo().name,  // Name of device set via web UI
  scanInterval: 60,      // secs, this will run a timer for every 60 seconds to fetch the voltage and set relay
  startTime: new Date().getTime(),    // Time we started running
  targetTemp: 0,         // target floor temp (in F) - updated via MQTT from HASS
  targetMin: 0,          // min temp we allow
  targetMax: 75,         // max temp we allow
  voltmeterID: 100,      // the ID of the voltmeter - this is set during setup of add-on sensor

  /**
   * Applies some math on the voltage and returns the result. This function is called every time the voltage is measured
   * @param {Number} voltage The current measured voltage
   * @returns The temperature based on the voltage
   */
  calcTemp: function (voltage) {
    const constVoltage = 10;
    const R1 = 10000;
    // These need to be set by calibrating your sensor at known temps - see: https://rusefi.com/Steinhart-Hart.html
    const A = 0.0018214378493520188;
    const B = 0.00013311546059977414;
    const C = 3.980571797046328e-7;

    const R2 = R1 * (voltage / (constVoltage - voltage));
    const logR2 = Math.log(R2);
    let K = 1.0 / (A + (B + C * logR2 * logR2) * logR2);
    temp_c = K - 273.15; // Celcius
    temp_f = (K - 273.15) * 9/5 + 32; // Fahrenheit

    return { temp_c: temp_c, temp_f: temp_f, voltage: voltage, resistance: R2 };
  },

  /**
   * If anything goes wrong turn off the switch to ensure we are never accidentally on.
   */
  turnOff: function () {
    Shelly.call("Switch.Set", {
      id: 0,
      on: false
    });
  },

  /**
   * This function is called every time when a temperature is read
   * @param {Object} data Object containing temp_f, temp_c, voltage, and resistance
   */
  onTempReading: function (data) {
    // Check temperature against target
    let temp_diff = CONFIG.targetTemp - data.temp_f;
    let status;
    if (Math.abs(temp_diff) <= CONFIG.hysteresis) {
      status = "UNCHANGED";
    } else if (temp_diff > 0) {
      Shelly.call("Switch.Set", {
           id: 0,
           on: true,
           toggle_after: CONFIG.maxOnSeconds
      });
      status = "ON";
    } else {
      CONFIG.turnOff();
      status = "OFF";
    }
    console.log("Temperature: " + data.temp_f.toFixed(2) + "°F | Target: " + CONFIG.targetTemp + "°F | Voltage: " + data.voltage.toFixed(3) + "V | Resistance: " + data.resistance.toFixed(2) + "Ω | Relay: " + status);
    CONFIG.publishStatus(data, status);
  },

  /**
   * Publish current state to MQTT 
   */
  publishStatus: function (data, status) {
    if (MQTT.isConnected()) {
      let msg = {
        temp_c: data.temp_c,
        temp_f: data.temp_f,
        voltage: data.voltage,
        relay_status: status,
        target_f: CONFIG.targetTemp
      };
      MQTT.publish(CONFIG.name + "/status", JSON.stringify(msg), 0, false);
    }
  },
};

// Our main "loop" - check the voltage and enable/disable switch based on target temp vs. actual temp.
// If anything goes wrong we want to disable the heating.
function checkTemp() {
  // Don't do anything if we haven't gotten a target temperature via MQTT
  if ((new Date().getTime() - CONFIG.lastTempUpdate) > (60000 * 60)) {
    console.log("No recent target temp update - turning OFF.");
    CONFIG.turnOff();
    return;
  }

  // Fetch the voltmeter component and exit if we can't find our component.
  const voltmeter = Shelly.getComponentStatus("voltmeter:" + CONFIG.voltmeterID);
  if (typeof voltmeter === "undefined" || voltmeter === null) {
    console.log("Can't find the voltmeter component - turning OFF.");
    CONFIG.turnOff();
    return;
  }

  // Get voltage value and exit if can't read the voltage
  const voltage = voltmeter["voltage"];
  if (typeof voltage !== "number") {
    console.log("Can't read the voltage or it is NaN - turning OFF.");
    CONFIG.turnOff();
    return;
  }

  // confirm voltage is in expected range
  if (voltage < CONFIG.minVoltage || voltage > CONFIG.maxVoltage) {
    console.log("Voltage outside expected valid range - turning OFF.", voltage);
    CONFIG.turnOff();
    return;
  }

  // Get the temperature based on the voltage and exit if invalid
  const data = CONFIG.calcTemp(voltage);
  if (typeof data !== "object") {
    console.log("Something went wrong when calculating the temperature - turning OFF.");
    CONFIG.turnOff();
    return;
  }

  // Trigger our logic that process temp
  if (typeof CONFIG.onTempReading === "function") {
    CONFIG.onTempReading(data);
  } else {
    CONFIG.turnOff();
  }
}

/**
 * Listen for config changes to our target temperature.
 */
function mqttListener(topic, msg) {
    if (topic === CONFIG.configTopic) {
        if (typeof msg === "undefined" || msg === null) {
            console.log("Invalid message MQTT format.", msg);
        } else {
            let new_config = JSON.parse(msg);
            if (typeof new_config[CONFIG.name].targetTemp === "number") {
                temp = new_config[CONFIG.name].targetTemp;
                if (temp >= CONFIG.targetMin && temp <= CONFIG.targetMax && CONFIG.targetTemp != temp) {
                    console.log("Updating target temp from " + CONFIG.targetTemp + "°F -> " + temp + "°F.");
                    CONFIG.targetTemp = temp;
                }
                CONFIG.lastTempUpdate = new Date().getTime();
            }
        }
    }
}

//init the script
function init() {
  //listen for our target temp
  MQTT.subscribe(CONFIG.configTopic, mqttListener);

  //start the timer
  Timer.set(CONFIG.scanInterval * 1000, true, checkTemp);
}

init();
```

## Next

Next read about [how to build Home Assistant automations](/posts/iot-floor-heat-automation) to control floor temps.
