---
title: "Lessons and Notes - Proximity-triggered LEDs"
description: "A collection of notes and learnings from my Capstone project"
layout: post
toc: true
hide: false
comments: true
show_tags: true
categories: [IoT, Microcomputing, Flutter]
---

# Lessons and Notes - Proximity-triggered LEDs

A random collection of concepts that I took away from this project

## TLDR
This project was my crash course in computer science, low-level programming, app development, electrical circuit theory, and app development. In hindsight, it was quite complicated for someone like me, who has never worked with embedded systems (actual hardware) before.

The project was expensive, definitely time consuming, incredibly frustrating at times, and yet may have been one of the most rewarding things I can remember doing. 

* [Code for ESP32 Beacon Sensor and Neopixels](https://github.com/stantonius/ESP32_smart_home_LEDs)
* [Code for Smart Home App](https://github.com/stantonius/flutter_smart_home_app)

## Lessons

Beware - the following is long and fairly unstructured. But it may help someone - or at the very least it will be my online reference for later.

### Microcomputing (mostly ESP32)

An ESP32 can be referred to as a SoC (System on a Chip) and/or an MCU (Microcontroller unit). However my understanding is that an MCU is appropriate when its just the ESP32 (no peripherals) whereas the SoC is for an ESP32 with peripherals (ie. a dev board).

Some older models of ESP32s **require antennae**. The first ESP32 I got required one - **ESP-WROOM-32U**. I saw what looked like a clip for the antenna but ignored it.
* I spent many hours troubleshooting why this ESP32 got the RSSI (proximity) values were so off before I clued in that an antenna would be massively helpful. This discovery was a **huge time sink**.
* In my case, consistent with other testaments online, using this device without an antenna actually ruined the device - even with one attached, nothing was accurate and Wifi connections were not reliable. A new ESP32 confirmed it was an antenna issue when RSSI.

Reading the datasheets of any device or chip is helpful to understand the device as this provides the pinouts (mapping of pins to functions), the voltage inputs and outputs, etc. However, the following are rough guidelines for the ESP32 that I learnt:
* The ESP32 runs on 3.3V
	* All output GPIO pins are 3.3V
	* GPIO input pins will tolerate up to 3.7V
* You can power the ESP32 [3 (sometimes 4) ways](https://diyi0t.com/esp32-tutorial-what-do-you-have-to-know-about-the-esp32-microcontroller/)
	* **Voltage regulated** - an AMS1117 3.3V regulator transforms the inputs into the working 3.3V:
		* 5V USB - standard micro USB
		* VIN pin - this method is a bit contentious on several forums. Some say that you actually need 7-12V (as 1-2V may be lost in heat), others say that 5V is fine. I can't quite understand why 7-12V is suggested for this method but not for the USB connection, when both methods pass through the voltage regulator to output 3.3V. The one thing for sure is that the 5V power supply must be high quality that supplies consistent and stable 5V (unlike phone chargers). However [some](https://www.reddit.com/r/esp32/comments/ncx31t/doit_esp32_devkit_v1_powering_with_5v_on_vin/) have suggested a buck converter could be placed in between the power supply and the VIN, but I imagine this is more useful if you are using a higher voltage power supply.
			> I have learnt not all chargers are equal. USB phone chargers are rated for 5V but they actually fluctuate in their output. 
		* **Voltage unregulated**
			* 3.3V pin - you can actually input voltage to this pin so long as it is *very close* to 3.3V, but in my case I need this output 3.3V anyway so this method isn't an option.
			* Battery - I didn't do much research on this method as this was never in the cards.

The ESP32 *chip* consumes anywhere from **160-260mA** in active mode. Note that the ESP32 has [multiple power modes](https://lastminuteengineers.com/esp32-sleep-modes-power-consumption/), but I am powering this through the mains (wall socket) so I am not concerned with these modes at the moment

### The Makers World

Don't be fooled by some posts that suggest this may be a cheap way to make smart lights. This has not been cheap. If you don't have any *maker* equipment, then the amount of tools you need are substantial.

This isn't to discourage - this has been one of the most rewarding projects I have done. It is an honest assessment that for me this has cost several hundred CAD more than using just the tools above because I started from scratch. One point to acknowledge however - I've come to the realization that this hasn't exactly been environmentally friendly. A lot of travel carbon, packaging, and unknown device manufacturing standards


### Electrical Theory

I could write a lot more here - but there are far better resources out there to explain this. However highlighted below are the key concepts I took away.

> Voltage = **Electrical Potential** = **Potential Difference between 2 points**

Note that Potential is the same as Gravitaional Potential Energy - ie. when an object is at the edge of a cliff, there is the *potential* to do work, but only when invoked (pushed). 

#### Ohm's Law

If I took anything away from this project, it is the following:

Do not think of Ohm's Law as 

$$V=IR$$ 

but rather 

$$I=V/R$$


Current flows from high voltage potential (ie. positive end of battery) to lower voltage potential (ground, 0 volts). This water-and-pipe analogy is massively helpful.
<p align="center">
<img src="https://cdn.sparkfun.com/assets/6/f/b/5/3/5113d1c3ce395fcc7d000000.png"/>
</p>
Source: https://cdn.sparkfun.com/assets/6/f/b/5/3/5113d1c3ce395fcc7d000000.png

Note that online tutorials are far better at explaining it, but the key takeaway is that the water flow (current) is a function of the amount of water (pressure, voltage) and the width of the pipe (resistance). This way of thinking is exactly like framing Ohm's law to ficus on current and not voltage. 

#### Voltage Drop

Here is where theory became practice. Grasping electrical theory and Ohm's Law is great, but another key element in building a circuit is understanding the materials you use and their impact. 

For my project, I was planning to run a 2.2m length of cable between the device and the LEDs since I wanted the ESP32 out of the closet (for safety concerns).
* I was made aware that there would be a voltage drop across the connecting wires due to its length, despite the fact that I used 18 AWG wires (see the Wire Gauge section).

```python
brightness = 50
total_brightness = 255 # FastLED uses a scale of 0-255 to set brightness
brightness_ratio = brightness/total_brightness
max_single_amperage = 60 #mA, typical for white Neopixel on max brightness
num_leds = 68

estimated_amperage = brightness_ratio * max_single_amperage * num_leds
estimated_amperage #800mA
```

To calculate voltage drop, we use the formula $$V_{drop} = I*R$$, where `I` is the current through the wire and `R` is the resistance of the wire (in reality, the resistance of a wire is impacted by many things, including the distance, width/gauge, whether the current is single or three-phased, AC or DC). Therefore I outsourced the calculation to [this online calculator](https://www.calculator.net/voltage-drop-calculator.html?necmaterial=copper&necwiresize=6&necconduit=steel&necpf=0.85&material=copper&wiresize=20.95&resistance=1.2&resistanceunit=okm&voltage=5&phase=dc&noofconductor=1&distance=2.2&distanceunit=meters&amperes=0.8&x=84&y=20&ctype=size), the expected voltage drop at this low current would be **0.074V** (or 1.47%), leaving 4.926V at the end.

In reality, the voltage drop was negligible.

If we were to increase the brightness of the 68 LEDs to half their theoretical max (125), thus increasing the current load, the voltage drop increases to **0.18V** (or 3.69%), leaving only 4.82V. However according to my multimeter, the voltage dropped only 30mV

Note shortening the distance to 20cm reduces the voltage drop at 2A load to just **0.017V** (0.34%). 

> This begs the question though - if all we are doing is deferring the distance traveled by the electrical current to an extension cord instead of the custom build cables, don't extension cords also experience significant voltage drop and thus will also impact the LED colour? The answer is yes and no. We need to think about this in the context of the types of current each is transmitting (in addition to the significantly larger gauge that an extension cord has). Devices that plug directly into a wall outlet (ie. a lamp) are receiving 120V (in NA; 220-240V mostly elsewhere). Devices like a phone or computer charger - this is a misnomer, they are actually power *adapters* that just convert AC to DC and step down the voltage to say 5V - have transformers and other components that convert AC to DC and down to a usable voltage. So back to our extension cord vs jumper wire discussion - the extension cord transmits **15-20A AC at 120V**, therefore any voltage drop will be negligible (both due to the high voltage and the smaller gauge). Given that our 5V power source transforms any voltage over 5V down to 5V, *it doesn't matter if the extension cord experienced voltage drop* at least for our use case. So the answer therefore is if we had to choose, we would use extension cords to bring the 120V source as close to the LEDs, so that the subsequent DC output jumper cables can be as short as possible and thus  experience the least amount of voltage drop when powering the LEDs. 
> Most NA house circuits have 15 or 20A, which equates to (15A * 120V =) 1800 watts or (20A * 120V =) 2400 watts before the breaker trips ([source](https://michaelbluejay.com/electricity/maxload.html#:~:text=Most%20modern%20residential%20circuits%20are,watts%20before%20the%20breaker%20trips.))

Why do we care about voltage drop for these LEDs? 

If the LEDs don't get close to the 5V they expect, the colours start to look whitewashed and saturated. This effect becomes more pronounced the further down the LED strip you go (because WS2812B pixels strips are connected *in series*, and each LED has its own resistance, thus creating an additive loss of current as you move down the strip).

Note most forums I read suggest adding multiple power supply connections along a strip of LEDs (roughly one connection for every metre). However since my strip is only 1m long, I am not too concerned.

> It has also come to my attention that we step down (transform) voltages in every device. Where does the excess power go? The first law of thermodynamics states that since we can't create or destroy energy, the difference in power from what the device consumes and what it was provided is converted into some other form of energy. In most cases this new energy form is heat - in fact this is what purpose-built resistors do - they convert a certain amount of electrical energy into heat energy.
> Surely this system, despite how good it has worked for humanity thus far, is quite inefficient? 


#### Power & Energy
**Power** is the **amount of energy** transferred or converted per unit time. Its is a ***rate*** of *how much* energy *can do work* (ie. **how much** work can be done at a single moment)

$$P = VI$$

* The unit of Power is Watt (W)
* 1 W = 1 Joule/second
* Interestingly: $$1 HP = 746W$$

**Energy** on the other hand is the **total amount of work done** over a period of time - it is **power over time**. 

$$E = P*t$$

* The unit of Energy is the **Joule**, which based on the definition above is 1W per second
* It is usually measured in kiloWatt hours (kWh) on our hydro bill
* **1 V = 1 J of energy done by 1 coulomb of charge**
	* A coulomb is just a defined number of electrons, which for our purposes is a large arbitrary number (like a mole in chemistry)

[This](https://energyeducation.ca/encyclopedia/Energy_vs_power) is the best explanation of power vs energy. Lifting the box in the image below requires a specific amount of energy, no matter how quickly the box is picked up. Lifting faster will change the amount of power, but not the amount of energy.

<p align="center">
<img src="https://energyeducation.ca/wiki/images/2/2b/Lifting_a_box.jpg" alt=""/>
</p>
Source: https://energyeducation.ca/wiki/images/2/2b/Lifting_a_box.jpg

#### Wire Gauge 

Wire gauge is quite important, and this is one area that adds to the cost unexpectedly for a newbie. It also is the point at which the simplified electrical theory understood thus far starts to break down, and we realize that this field is in reality far more complex. Wire gauge (diameter) impacts the circuits ability to carry current over longer distances.
* The higher the gauge, the thinner the wire
* The lower the gauge, the higher its current carrying capacity and the less voltage drop that will occur over longer distances. This was important for my project where the distance between the power supply and the LED strip was 2.2m.
* According to [Wires, connectors and current - what you need to know as a drone builder - Guides - DroneTrest](https://www.dronetrest.com/t/wires-connectors-and-current-what-you-need-to-know-as-a-drone-builder/1342), an 18 AWG wire can carry up to 30A, whereas 22 AWG is limited to 10A.

The general formula for calculating the max amperage for a wire is:

$$Amps = \text{cross sectional area } (mm^2) \times 25$$
					
Resistance per metre:

$$ \text{Resistance per meter} = 0.0168 / \text{ cross sectional area } (0.0168 / mm^2)$$
<<<<<<< HEAD:_posts/2021-10-30-proximity_leds_lessons.md

Voltage drop per metre
=======
					
Voltage drop per metre:
>>>>>>> 9bd14e895c18cc3f9d71953ca3787165f2a68a24:_posts/2021-10-30-smarthome_base.md

$$ \text{Voltage drop per meter} = (current \times 0.0168)/ \text{ cross sectional area } ((current x 0.0168) / mm^2) $$
					
Power loss per metre:

$$ \text{Power loss per meter} = (current^2) \times 0.0168)/ \text{ cross sectional area } ((current^2) \times 0.0168) / mm^2) $$
					

#### Connecting Wires Together

Keep in mind that breadboarding is a great start - however in order to create an actual device that is safe and portable, you need to eventually connect things together. For wires, I found that it was hard to commit to soldering them directly to endpoints. Therefore I often went with connectors - much easier to correct mistakes and adjust the device as you iterate (seeing as I was not working off a blueprint - the chances of mistakes were high).

There are several different types of connectors. I just happened to blindly purchase JST connectors off Amazon. These JST connectors have on average **0.04V drop per amp of current** according to [JST connector volage drop measurement results. - RC Groups](https://www.rcgroups.com/forums/showthread.php?184830-JST-connector-volage-drop-measurement-results). Therefore we *should always solder when possible*. There is [very little voltage loss through a solder joint](http://blog.bogpeople.com/2016/05/crimp-vs-solderfight.html). To be more specific, solder still has 5x the resistance as copper, so the most ideal answer is a solid copper connection. However this is never really possible

All this being said, small circuits that don't require significant amperage (ie. <1A) can use the standard 22AWG for breadboards. And despite breadboards having high voltage loss do it its small/poor quality contacts, this is normally fine for prototyping low power circuits. However it is important to note that breadboards [generally have a current limit of 1A](https://www.circuitspecialists.com/blog/common-breadboard-specifications/#:~:text=Due%20to%20the%20temporary%20nature,(pFs)%20for%20every%20connection.)
* Solderable breadboards (often called protoboards or perma-protoboards) have a much higher amperage.
* Even though the embedded copper connectors in a protoboard are far superior to breadboards (and [can handle much higher currents](https://forum.allaboutcircuits.com/threads/what-boards-to-solder-on-for-high-current-circuits.73606/)), most still say to err on the side of caution and add an additional wire to the high current traces of the circuit.

Soldering 18AWG wire to the small pads of the LEDs was very tough. Eventually I realized that trimming/filing *one side* of the exposed 18AWG wire so that it was flat against the LED copper pads was the trick that worked. However using JST connectors for wire of this low gauge is not the best idea for the next project.

After all of this work, it is clear that there is inevitable **voltage drop** that will occur the circuit contains the following characteristics:
* Current travelling long distance
* Number of connections (meaning number of joints such as solders, crimps)
* Smaller wire gauge
* More resistors in the circuit (everything is a resistor, therefore the more complex the circuit the higher the voltage drop)

Some design changes I would consider if I were to restart this project:
* Use a higher voltage power source (6 ot 7V) and use buck converters (weird name I know - they are configurable devices that allow you to drop voltage to a certain level) to lower the voltage near the devices (in my case near the LEDs and the logic level signal converter; the ESP32 could handle a higher voltage than 5V if using the VIN)
	* A higher voltage does two things (although they may be dependent of eachother):
		* Reduces voltage loss (important over longer distances)
		* Creates voltage buffer so that any voltage drop still results in voltage at or above the device requirement


#### Circuit Design
It had been a while since I had thought about circuits in series and in parallel.

**Series Circuit**: resistances in series are **additive**
**Parallel Circuit**: the potential difference across any component is the same and equal to the supply voltage. 

So in this new design, if we have two wires in parallel - 1 feeding the protoboard containing the ESP32, and one powering the lights directly - what is the current distribution across the two wires?

Well we know that an object's resistance is (basically) constant. Since energy cannot be created or destroyed, and voltage and resistance are constant, with Ohm's Law we can deduce that the current will halve between the two wires (assuming their resistances are the same).

#### Prototyping

The initial testing and design was done using a standard breadboard. In reading some of the forums discussing Neopixels, I was made aware that sending high amounts of current through a breadboard is really discouraged (and even dangerous). The estimated max current (**ampacity**) through a regular breadboard seems to be around **1A**.

##### Protoboard

A solderable breadboard with embedded channels (wires) is the better approach to prototype once you have designed and tested the initial layout. 

It was difficult to find an answer on the max ampacity for a protoboard. According to this old forum [Adafruit customer service forums • View topic - Perma-proto board ampacity](https://forums.adafruit.com/viewtopic.php?f=19&t=61250), the ptotoboards power lines are 32 mil wide. When using [this trace width calculator](https://www.digikey.ca/en/resources/conversion-calculators/conversion-calculator-pcb-trace-width) to estimate the voltage drop and heat produced, it suggested that for a 5C temperature increase I would only see a 0.05V drop (thereby suggesting this setup was relatively safe?) 

![[Pasted image 20211016204955.png]]

### C++

Because I chose to use the Arduino (a C++ derivative) language instead of Python, I had to get up to speed with C++ (something that has proven very difficult, but quite rewarding). Below are some of my oberservations as I *tried* to pick up the language:
* Not much different to Dart from what I can tell (this is probably offensive to those that know the langueages inside and out)
* I have found the elements that are tied deep to computer science theory to be the most difficult (things that are managed for you in Python and even Dart):
	* Pointers and references. Not so much what they are, but *when* to use them.
	* Arrays in c++. I just don't understand them.
* Some may cringe at me calling development on the ESP32 c++ when in reality it is Arduino code. But for my purpose they are the same. And one of the advantages of using the Arduino framework is that the libraries are well documented and purpose built for SoC's like the ESP32.
* Some of the other c++ habits I have developed that help me in my other coding (especially Dart) are:
	* **Type declarations**: not always needed in Dart, but required in c++, this helps you to think about how the object will sit in memory; the whole point of type declaration is that we are letting the computer know *how many bytes this object will need allocated in memory*.
	* **const**: should be used when creating variables wherever possible as it is the most performative. The reason is its value is calculated at *compile time*, meaning unlike other variables, the program doesn't have to go searching for what its value is when running.
		* Note **final** means something different in Dart vs c++
	* **Stack vs Heap**:
		* **Stack:** **temporary** memory allocation that allocates and deallocates memory **automatically**; safe - data on stack only accessible by the owner thread - and faster to read/write.
			* Whenever a function is called, its variables are placed on the stack and are only available while the function is running. See below from [Stack vs Heap Memory Allocation - GeeksforGeeks](https://www.geeksforgeeks.org/stack-vs-heap-memory-allocation/):

		```cpp
		int main()
		{
		   // All these variables get memory
		   // allocated on stack
		   int a;
		   int b[10];
		   int n = 20;
		   int c[n];
		}
		```
		* **Heap:** simply opposite of the stack. 

### FastLED & WS2812B

I used the [FastLED](https://fastled.io/) Arduino library to configure the LEDs since they had pre-programmed light sequences, which I didn't feel like piecing together. Again, some of my notes are below:
* Colours were off and I still can't explain why. However I do see all of the colours represented when I run a rainbow script - therefore, there is a 
* These series of Youtube videos were excellent and they explained some of the basics of colour theory
	* The one thing I took away is that the first number in the Hue colour system is a continuous 0-255 rolling 

Some notes on setting up the WS2812B LEDs:

* You CANNOT attach the data wire in any place other than the start of a strip. This caused many hours of troubleshooting, as I had soldered the wires to the second LED in the string as the first one's copper pads were too small for the 18 AWG wire. However once I clipped off the first LED, the programme worked
* The data pulse output from the GPIO pin to the LEDs is extremely fast - so fast that my cheap multimeter could not process the output before it "disappeared". This also compounded my troubleshooting issue mentioned above as it looked like my program wasn't working correctly (since when testing the GPIO output, the multimeter wasn't reading any voltage) when in reality the issue was in the physical circuit.
	* I assume that since we are powering the LED strips with external power, the data wire simply set which LEDs are to turn on via a quick pulse, but the sustained power is provided by the external source and thus the data output is only needed for an extremely quick pulse.

### Flutter App

The setup of Flutter is actually straight forward. The two elements of the app development process that were the most tricky were:
* State management: used Riverpod for this. Its a fantastic library and I really respect its developer Remi, but it has a steep learnign curve (for me anyway)
* Deployment for testing: I used Firebase app distribution, alongside XX	 and Github actions to always push the latest version of the app to my phone for testing. This worked with varying degrees of success
* Finding library(ies) that were permitted to run in the smart phone's background. Understandably, iOS and Android restrict what a developer can run and access on a user's phone to avoid , but these restrictions were quite constraining for an app I only ever intended for myself and my wife.

### Bluetooth

I decided to use Bluetooth Low Energy as the beacon mechanism on my phone, and take the resulting Received Signal Strength Indicator (RSSI) to gauge how close the phone is to the ESP32. I understand that RSSI is an imperfect localization indicator, but until I have a phone and SoC with Bluetooth 5+ or ultra wideband (UWB) technology, this is the best option available to me.

Note that I got some help from [this reddit post](https://www.reddit.com/r/esp32/comments/lgqxi5/phone_presenceroom_detection_using_multiple_esp32/), which also references a cool [cat localizer project](https://github.com/filipsPL/cat-localizer#ble-beacon). 

## Next Steps
The next blog will run through the physical setup updates of the lights to make them extra safe (and allow for them to be brighter).

Upcoming blogs will also:
* Review the Flutter app (incl. describing the lessons learnt from using Riverpod state management library) and turn off advertising when leaving the geofence
* Update of the ESP32 code 

- [ ] Allow for colour changes via app
- [ ] Integrate Google Assistant to turn on/off lights
- [ ] Power ESP32 via 5V power adapter
	- [ ] Build switch to stop 5V input to VIN so that we can still connect USB if needed (remember they can't both be connected at the same time)
- [ ] Build redundancy and failsafe mechanisms
	- [ ] Backup RPi server
	- [ ] Capture errors:
		- [ ] Node Red flows
		- [ ] MQTT down (ie. maybe if MQTT is down, we revert back to just turning the lights on?)
- [ ] Phone beacon battery optimization

## Personal Development Outcomes
* I can write a bit of c++ code
* I can write a bit of Dart/Flutter code
* I can use Github Actions and fastlane
* I know a lot more about electricity (and thus how the modern world operates)
* I can solder
* I can use MQTT and Node-Red
* I know more about Docker containers



