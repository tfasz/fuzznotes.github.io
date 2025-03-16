+++
title = "Automating my in-floor heating - intro"
summary = "Introduction to automating my floor heat"
date = 2025-03-01
tags = ["Floor Heat", "IOT"]
+++

I remember hearing [Tony Fadell](https://en.wikipedia.org/wiki/Tony_Fadell) talk about his vision of replacing
all of the dumb white boxes around your house when he started Nest. At least he managed to do a good job with 
the thermostat (and an ok job with smoke detectors) before riding off into the sunset.

## The problem

We have in-floor heating in a few of our bathrooms which were controlled by dumb white boxes. These boxes were:
* rather expensive
* didn't have any wifi connectivity or ability to control them remotely
* had clocks that would drift from the actual time and need to be adjusted regularly
* didn't account for daylight savings
* would forget their scheduled programming if the power was out for more than a few minutes
* didn't know if we were home or away so we manually turned them OFF before leaving
* didn't have any ability to report energy usage

Yes, there probably are some more modern implementations available that solve some of these problems but they
usually come with their own cloud service and app. I'd prefer to be able to centrally control and manage these
via my own [home automation system](https://www.home-assistant.io/) along with everything else in my home. And hey - 
it's a fun problem to solve.

## In-Floor Heat Sensors

The first thing I wanted to understand was how these thermostats and sensors work.
In floor heating systems typically have a sensor embedded in the floor during installation. These sensors are
usually 10k NTC sensors (aka [Thermistors](https://en.wikipedia.org/wiki/Thermistor)) that have a varying 
resistance at different temperatures. If you know the resistance at 3 different temperatures you can use the 
[Steinhard-Hart equation](https://en.wikipedia.org/wiki/Steinhart%E2%80%93Hart_equation) 
to calculate the sensor's temperature at any resistance. 

I used my existing thermostat to get the floor temperature to 3 different points and then used a multi-meter to 
test the resistance of the sensors so I could later calibrate my replacement controller. Measuring the resistance 
is simple - just turn off the breaker to the thermostat, find the sensor wires where they attach to the thermostat, 
detach them and measure the resistance across the two wires of the sensor.

## IOT Device Choices

To replace my thermostat I needed to find a device that could measure the floor sensor's resistance to determine the 
current temperature and compare it to a desired target temperature. It would then need to turn ON/OFF power to the 
resistive heating element until the target temperature is reached. I wanted a device that was suitable for 
putting in a electrical box in the wall where the current thermostat, power, and sensor wire were all located. 
I also wanted to avoid any USB powered device that required an additional line power to USB transformer. 
Luckily Shelly makes great devices like the [Shelly 1PM](https://us.shelly.com/products/shelly-1pm-gen3) that are powered 
by line voltage, suitable for putting in a wall box, WiFi controllable, and have an in-line relay. They also can be
connected to a [Shelly Plus Add-On](https://us.shelly.com/products/shelly-plus-add-on) which can be used to integrate
with the NTC sensor.

## Next

In the next post we will talk about 
[how to wire these devices to the LINE, LOAD, and SENSOR](/posts/iot-floor-heat-wiring).