---
title: "Proximity-triggered LEDs"
description: "Use your phone as a beacon to turn on ESP32-controlled LEDs as you enter a room"
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [IoT, Microcomputing, Flutter]
---

# Proximity-triggered LEDs

Use your phone as a beacon to turn on ESP32-controlled LEDs as you enter a room

This was effectively my Computer Science/IoT/Flutter Capstone project

## Project Overview
We recently moved into a new house. I got the idea of making some ambient lights in the spare room from [Andreas Speiss' Raspberry Pi Server video](https://youtu.be/a6mjt8tWUws) - thanks Andreas and Google for the Youtube video recommendation. Its now 2.5 months later...

The tools used would be:
* My Android Pixel 4 phone (as a Bluetooth beacon)
	* A custom Flutter app acts as the beacon
* A Raspberry Pi 4 8GB home server
	* IOTStack running the necessary services
* WS2812B LEDs (Neopixels)
* ESP32 controller

<figure>
	<img src="https://storage.googleapis.com/craigstanton-public-assets/images/smarthome/smarthome_landscape_23Oct2021.png"/>
	<figcaption>The key compoenents of this smart home system</figcaption>
</figure>
The idea: as I walked into the spare room where my workstation is, the ESP32 would recognize I was in the room and turn on the lights. When I left, they turn off.

<figure>
	<img src="https://storage.googleapis.com/craigstanton-public-assets/images/smarthome/smarthome_objective.svg"/>
	<figcaption>A basic description of the system logic - when the beacon (phone) gets close enough to the sendor (ESP32), the lights should turn on</figcaption>
</figure>

I thought the above would be straightforward. I was wrong. Made me think about this tweet:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Almost every problem in my life can be traced to an unfounded optimism about how quickly I can get things done and how many things I can do at once.</p>&mdash; Sean J. Taylor (@seanjtaylor) <a href="https://twitter.com/seanjtaylor/status/1445945021010624512?ref_src=twsrc%5Etfw">October 7, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Upon reflection, I had no real business trying this as my ack of computer science was exposed early on in the project - but a little stubornness and perserverence got me through it.

### Current State of Project
This system works OK. Whilst I discuss potential improvements at the end, the two major fixes needed before this project can be considered a success are:

* [ ] Prevent Android from randomly terminating apps running in the foreground
* [ ] Add an RSSI processing filter on the RPi to improve proximity accuracy

## Architecture

<figure>
	<img src="https://storage.googleapis.com/craigstanton-public-assets/images/smarthome/actual_smarthome_workflow.svg"/>
	<figcaption>The smart home architecture</figcaption>
</figure>

The system works as follows:
1. The Flutter app makes the Pixel phone a Bluetooth beacon (specifically, it uses BLE to periodically advertise itself as an iBeacon). In Bluetooth speak, this is called the server. A custom UUID is broadcast in the iBeacon message, which is what the Bluetooth client (the device that is scanning for iBeacons) uses as a filter to isolate only one specific device.
2. The ESP32 (Bluetooth client) runs Arduino sketches on both of its cores. One core uses the built-in antenna to continuously scan for a device with a specific UUID. As the diagram implies, the same antenna is used for both Wifi and Bluetooth communications, so the code flips between using the antenna for Bluetooth and Wifi. It is in this step that the ESP32 evaluates the proximity of the beacon using RSSI.
3.  If the RSSI is above a certain threshold, the ESP32 sends a message via MQTT (over Wifi) to the Raspberry Pi server to indicate that a beacon is close. Conversely, when a beacon moves out of range (ie. the RSSI is below the threshold), the ESP32 sends a message indicating the beacon is not present.
4. IOTStack containers running on the Raspberry Pi make this the main server for the house. One of the containers is running the Mosquitto MQTT protocol and is subscribed to the specific topics which indicate if a beacon is close by or not. Another container is running NodeRed (a drag-and-drop UI for chaining together different actions depending on an input), which listens for the MQTT messages and triggers over MQTT back to the ESP32.
5. The ESP32 is simultaneously running an MQTT library that is subscribed to topics from the Raspberry Pi. The ESP32 interprets the incoming topic messages to either turn On or Off the Neopixel LED strip.
6. If the incoming topic contains an "On" message, the ESP32 sends the LED light program data through one of its GPIO pins which is connected to the LED strip. Conversely, an "Off" message stops the GPIO pin activity, and the LEDs shut off.

### Code 

The code to make the above system work has contains 3 distinct elements:
1. Arduino sketches (C++) for the ESP32
2. Flutter app that contains the persistent beacon 
3. IoTStack containers running on the Raspberry Pi

* [Code for ESP32 Beacon Sensor and Neopixels](https://github.com/stantonius/ESP32_smart_home_LEDs)
* [Code for Smart Home App](https://github.com/stantonius/flutter_smart_home_app)
* [Code for NodeRed flows](https://github.com/stantonius/smart_home_nodered_flows)

## Prototype Design

The initial plan was to have the LED strips around the base of the guest bed frame, which would have required ~5m of LEDs. Following some research, I came across two issues I wasn't prepared to solve:

1. The high current needed for this type of setup was too much for a first project. The max current per LED (to produce a pure white light) is 60mA. With a light density of 60 LEDs/m, a total of 300 LEDs would need to be powered. At their brightest, this would equate to 18A (300 x 60mA) to power all the LEDs - this felt too dangerous for a first timer.
2. The 5m LED strip would need to be powered every ~1m to avoid **voltage drop** across the LED strip to avoid light dimming near the end of the strip. I did not have the enough wiring to power the LEDs this way.
	1. Note the type of LEDs used were WS2812B (aka Neopixels) which require 5V - as I discuss in [insert link to lessons blog], if I wanted to do bedframe LEDs in the future I might use 12V LEDs (since higher voltage experiences less voltage drop over the same distance compared to 5V)

The next option was to mount the LEDs above the closet in the room, as there was a nice ledge to have the LEDs pointing down towards the floor. It also was ~1m long, so short enough to not require additional power input onto the strip.

<figure>
	<img src="https://storage.googleapis.com/craigstanton-public-assets/images/smarthome/lights_finished.jpg"/>
	<figcaption>The finished lights setup above the closet</figcaption>
</figure>

The design was such that the ESP32 would be over 2m away from the start of the LED strip - this was to avoid placing the homemade device inside the closet, which I was hesitant to do given this was a first time circuit build (I wanted the device visible to monitor any physical signs of system fault). I know this setup seems weird given we just discussed the impact of voltage drop on colour output - however I performed the following tests that confirmed this setup would work:

* I used 18AWG wiring to connect the ESP32 and power source to the LEDs (the smaller the gauge, the wider the wiring diamter, the less voltage drop occurs). I then measured the voltage at the start and end of the 2.2m of wiring using a multimeter, and there was negligible drop.
* I connected the 68 LED strip (just over 1m long) to the 2.2m wiring and didn't notice any discolouration. 

A final point to mention is how I managed to power the LEDs safely. Two features were added:

1. An LED strip any longer than a handful of LEDs need to be powered from the mains (ie. using power from an outlet). Even though I had fixed the components to a protoboard, which can handle far more current than a regular breadboard, I decided to be extra precautious and add a power bypass so that the LEDs could theoretically consume the power needed directly from the mains. This was to avoid any chance that the protoboard would overheat from high amounts of current.
<figure>
<img src="https://storage.googleapis.com/craigstanton-public-assets/images/smarthome/prototype_finished.jpg"/>
	<figcaption>The current prototype setup. Notice the bypass red and white wiring that connects the mains to the LEDs</figcaption>
</figure>
2. I added a fuse to the LED power input with a max current of 2A. This was done out of an abundance of precaution, to avoid any short in the LED strip that could theoretically cause a fire. Note these were basic fuses I found on Amazon that I think are meant for automobile electronics - however the nice part of this setup is the fuse is easily replaceable so if the LEDs require more power, I can simply add a higher capacity fuse.


<figure>
	<img width="50%" src="https://storage.googleapis.com/craigstanton-public-assets/images/smarthome/fuse.jpg"/>
	<figcaption>Image of generic auto fuses purchased from Amazon</figcaption>
</figure>



## Centralized Server

In the initial design, the ESP32 was the centralized controller for managing the LED status. However this posed a problem - how do you turn off the LEDs if the sole logic evaluates whether the beacon is close to the sensor or not, and you are sitting in the room with your phone (ie. lying in the bed about to go to sleep)? 

What we needed was a central server that would manage the LED status - this setup would allow different inputs to connect with the server and change the LEDs status (ie. the proximity beacon, as well as a light switch in the app, a Google Assistant integration, etc.). In this setup, the ESP32 just passes the status of the beacon to the central server (Raspberry Pi) via MQTT and listens for the LED status (also via MQTT).


### 1. ESP32 MQTT setup

Although initially trying the `PubSubClient` Arduino library, I decided to try another library called  `AsyncMqttClient` because a) it has been more recently maintained and b) it works in an *async* manner. Note I probably could have got the `async` working on the `PubSubClient` library, but I was struggling to wrap my head around it.

The [AsyncMqttClient](https://github.com/marvinroger/async-mqtt-client) library has very clear documentation on how to set this up. However what I want to discuss is what tripped me up (and was also happening with the `PubSubClient` library - one of the reasons for switching) - the MQTT connection was dropping and therefore subscribed messages were not being received by the ESP32.

#### Wifi, BT, and the ESP32 antenna
After a lot of time researching, it finally clicked that since searching for beacons and delivering MQTT messages over Wifi happened immediately after eachother in the same loop, **they were competing for the single antenna on the ESP32**.

[Here](https://esp32.com/viewtopic.php?t=6707) is an example of a forum that mentions exactly this - that Bluetooth and Wifi share a single antenna, and that when the device is sending/listening for a Bluetooth backet, it cannot send or receive a Wifi packet. 

Therefore the change the eventually resulted in a consistent MQTT broker connection and subscribed message delivery was:

1. Adding delays to the code *to allow the device time to send/receive and then free up the antenna*. Note below, there are two 100ms delays following the BLE and the MQTT sequences.
```cpp

// https://github.com/stantonius/ESP32_smart_home_LEDs/blob/main/src/main.cpp

void codeForTaskRunBLEChecks(void *parameter)
{
    for (;;)
    {
        if (pBLEScan->isScanning() == false)
        {
            // Start scan with: duration = 0 seconds(forever), no scan end callback, not a continuation of a previous scan.
            doBLEScans(pBLEScan);
            delay(100);
        }
		// logic for reducing false negatives
        if (deviceProximityHolder.sum() == 0)
        {
            if (isCloseVal)
            {
                isCloseVal = !isCloseVal;
            }
            mqttClient.publish("BeaconProximity", 0, false, "false");
        }
        else
        {
            mqttClient.publish("BeaconProximity", 0, false, "true");
        }
        delay(100);
    }
}
```
2. Closing the Bluetooth scan (which actively shuts down the Bluetooth use of the antenna)
```cpp

// https://github.com/stantonius/ESP32_smart_home_LEDs/blob/main/src/ble.h

void doBLEScans(NimBLEScan *pScan)
{
    /* 
    * Get results of scan;
    * Note that the scan results each trigger the callback when there are results regardless if its a beacon
    * Therefore we need to:
    * 1. Empty unordered_set to store the results for each scan
    * 2. Check the set to see if it includes a beacon
    * 3. If a beacon is found, add 1 to the holder vector; if not, add 0 to the holder vector
    * */
    scanResults.clear();

    NimBLEScanResults foundDevices = pScan->start(scanTime, false);

    if (scanResults.find(1) != scanResults.end())
    {
        deviceProximityHolder.add(1);
        Serial.println("Beacon presence recorded");
    }
    else
    {
        deviceProximityHolder.add(0);
        Serial.println("No beacon presence recorded");
    };

    Serial.print("Devices found: ");
    Serial.println(foundDevices.getCount());

    // stopping might be key as we want to free up the device antenna to use MQTT over Wifi
    pScan->stop();
    pScan->clearResults(); // delete results fromBLEScan buffer to release memory
}
```


### 2. NodeRed Setup

While the flow that processes beacon proximity can be found [here](https://github.com/stantonius/smart_home_nodered_flows), I will just touch on the challenge I faced building this.

<figure>
	<img align="center" src="https://storage.googleapis.com/craigstanton-public-assets/images/smarthome/beacon_proximity_flow.png"/>
	<figcaption>The NodeRed flow that listens for beacon proximity and switch commends from the Flutter app</figcaption>
</figure>

We needed logic that could be controlled by multiple inputs (beacon proximity and the app light switch) but that didn't override eachother. For example, in the initial setup of the flow, the light switch could turn off the lights, but the very next scan saw the presence of the beacon and immediately turned the lights back on. As you will see if this flow, what ended up working was creating a *state*, whereby only when the state of one of the inputs was *the same as the current state* would the next state count. In other words, if we manually switched off the lights via the switch, only when the beacon is no longer present can it then turn the lights back on when it gets closer.

## ESP32 Proximity Sketches

Without going into every element of the ESP32 Arduino sketches, I want to highlight two key elements of the proximity logic:

* In the code above that defines the `codeForTaskRunBLEChecks` function, there is logic where we sum the two most recent scan events in the line `if (deviceProximityHolder.sum() == 0)`. This evaluates whether the **unordered_set** `deviceProximityHolder` contains a found beacon from the prior 2 scans. If scan recognized the beacon, the lights turn off. This setup is to **optimize for Type I errors (false positives) over Type II errors (false negatives)**. Obviously in an ideal world, we have no errors, however seeing as RSSI is an imperfect proximity measure, we have to pick a preferred scenario. In our case, we want the lights to turn on immediately when we enter the room, even if it means the lights sometimes turn on incorrectly. Conversely, we want the lights only to turn off *if we are really sure* the beacon is no longer present, and so we wait for two consecutive negatives before setting the LEDs to "Off"
* As we mention above (and will mention in further detail later in this post), the RSSI threshold needs to be set experimentally and varies wildly depending on your local setup. Below is logic that the ESP32 uses to a) filter Bluetooth devices found on the scan using UUID, b) filter only if that device is close (below a certain threshold) and c) set the global variable `beaconPresent` if the beacon is close by.

```cpp

// check if the beacon is the one we set the service UUID for
// this will exclude iTags for example
if (oBeacon.getProximityUUID().toString() == SERVICE_UUID)
{
	if (advertisedDevice->getRSSI() > rssiThreshold)
	{

		beaconPresent = 1;
	}

}

```


## Conclusion

As I stated at the outset of this post, this project was incredible in terms of learning. However as a functioning system, it still need work on the issues mentioned above. 

### RSSI as a Proximity Measure

Others, and now myself, have shown that using RSSI as a proximity indicator is possible. However it is an imperfect science and should only be used when sub-metre precision is not needed.

The positioning of the beacon (server) and the receiver (client) also takes some tinkering. For example, I have the following observations regarding the position of the client:

1. Placing the client higher up in the room a) reduces false positives when my phone is on the floor below and b) reduces interference within the room (as most objects are on the floor in a room). The higher up the client, the more likely there is to be a direct *line of sight* between the beacon (phone) and the client (ESP32).
2. Related to the observation above, it turns out the guest room mattress must be incredibly dense as it causes a lot of interference. Therefore I could not place the client on the floor beside the bed, as my phone is most often on the other side of the bed where my desk is.
3. Wireless charging seems to weaken the BLE signal from my phone. 
4. Play with threshold of RSSI cutoff and phone BLE power. You need to find a balance between false positives and false negatives for the former, and battery life vs signal strength for the latter.
	1. Related is the number of negatives recorded before the lights turn off themselves (currently 2 straight)

As far as improving how the system evaluates RSSI in general, there is a great series of posts [by Beaconzone](https://www.beaconzone.co.uk/blog/category/rssistability/) that discuss improving the RSSI metric through processing/filtering to remove the noise. To me, this is the next option to try as it is the most cost effective.

### Future Direction

Of course having done this project, there are many design changes and improvements that could be made to make this project even better. Some of them are outlined below:

* Split out the sensor (Bluetooth client) and the LED controller onto two separate devices. I have a suspicion (for which I have zero evidence) that since the antennae is shared between Wifi and Bluetooth, that the Bluetooth client isn't as strong a receiver as result.
* Implement RSSI processing as mentioned above
* Utilize (in part, or as a whole) new technologies:
	* Devices with Bluetooth 5+ (the [nrf52840 product sheet](https://www.nordicsemi.com/Products/nRF52840) refers to "high-precision RSSI")
	* Ultra wideband (UWB) which would replace using Bluetooth for proximity sensing with a much more accurate measurement
		* [Novelda](https://novelda.com/technology/) has an UWB chip that can detect the presence of humans
	* Computer vision that detects presence (and even who) is in the room

It is important to note that given the above potential solutions, they will come with risks. I have outlined my thoughts on the relative risks and advantages of these soltutions below: 
<figure>
	<img src="https://storage.googleapis.com/craigstanton-public-assets/images/smarthome/proximity_technology_options.svg"/>
	<figcaption>My analysis of the benefits and risks of using different technical approaches to create a smart home LED system</figcaption>
</figure>

If anyone wants to collaborate on this project going forward, give me a shout! 