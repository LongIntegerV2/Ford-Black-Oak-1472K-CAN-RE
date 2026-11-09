<p align="center">
  <img src="logo.png" alt="Trent's Tuning LLC" width="300">
</p>

<h1 align="center">Black Oak 1472K CAN</h1>
<p align="center"><b>A Complete Technical and Functional Reference for the Ford 5.4L 3V Powertrain Control Module</b></p>
<p align="center">Performance Tuning that Delivers Proven Results</p>
<p align="center"><a href="https://trentstuning.com/">TrentsTuning.com</a></p>

<p align="center"><b>Written by Trenton Hill</b><br>
Copyright 2026 Trent's Tuning LLC. All Rights Reserved.</p>

## Copyright and ownership

This document and all of the engineering work behind it are the property of Trent's Tuning LLC and were authored by Trenton Hill. It is copyrighted and protected, 2026 Trent's Tuning LLC, all rights reserved. You are welcome to read it, learn from it, and reference it for your own study and research. You may not copy it, repackage it, resell it, present it as your own work, or use it in a commercial product or service without written permission from Trent's Tuning LLC.

Several topics in this reference are marked **REDACTED**. For those topics the document still explains what the thing is and what it does in general engineering terms, but the Black Oak specific implementation details are proprietary to Trent's Tuning LLC and are deliberately withheld. This covers the bootloader and anything that would let someone write over it, the operating system kernel, module reading and writing and reflashing, security access and the seed and key exchange, the firmware validation and checksum, the custom operating system and every custom feature including the multiple switchable calibration map mechanism, and bidirectional controls. The goal of this reference is to teach how the module genuinely works so tuners and students understand these truck computers, not to hand anyone a recipe for the sensitive internals.

## How to read this reference

This is written to take someone from new to the subject all the way to an engineering level understanding of the Black Oak 1472K CAN powertrain control module, which from here on is called the PCM. It describes the platform in general rather than any one vehicle or calibration file, so the functional reference itself carries no part specific identifiers. The only addresses anywhere in this repository are the handful of load offsets in the Ghidra section below, which you need in order to open a raw read at all. Terms that may be unfamiliar are explained the first time they appear, and there is a full glossary at the end. If you are brand new, read it top to bottom. If you already know engine control, you can jump to any subsystem, since each section stands on its own.

## Table of contents

* [Repository contents](#repository-contents)
* [Loading this module in Ghidra](#loading-this-module-in-ghidra)
* [Part 1, Foundations](#part-1-foundations)
* [Part 2, The real time execution model](#part-2-the-real-time-execution-model)
* [Part 3, Data, units, and the calibration format](#part-3-data-units-and-the-calibration-format)
* [Part 4, Sensing and inputs](#part-4-sensing-and-inputs)
* [Part 5, Finding engine position and speed](#part-5-finding-engine-position-and-speed)
* [Part 6, Air charge and load](#part-6-air-charge-and-load)
* [Part 7, Fuel](#part-7-fuel)
* [Part 8, Spark and combustion timing](#part-8-spark-and-combustion-timing)
* [Part 9, Knock](#part-9-knock)
* [Part 10, Variable cam timing](#part-10-variable-cam-timing)
* [Part 11, Torque based control, throttle, idle, limiters](#part-11-torque-based-control-throttle-idle-limiters)
* [Part 12, Transmission, the 4R75E](#part-12-transmission-the-4r75e)
* [Part 13, Engine protection and failure handling](#part-13-engine-protection-and-failure-handling)
* [Part 14, Driver interface and cruise](#part-14-driver-interface-and-cruise)
* [Part 15, Communication](#part-15-communication)
* [Part 16, Diagnostics, monitors, and OBD](#part-16-diagnostics-monitors-and-obd)
* [Part 17, Accessories and charging](#part-17-accessories-and-charging)
* [Part 18, From key on to idle, the start sequence](#part-18-from-key-on-to-idle-the-start-sequence)
* [Part 19, The calibration surface](#part-19-the-calibration-surface)
* [Part 20, Memory and regions](#part-20-memory-and-regions)
* [Part 21, Proprietary internals, explained and REDACTED](#part-21-proprietary-internals-explained-and-redacted)
* [Part 22, Scope of the teardown](#part-22-scope-of-the-teardown)
* [Part 23, Full function map](#part-23-full-function-map)
* [Glossary](#glossary)
* [About and contact](#about-and-contact)

## Repository contents

Alongside this reference, the repository includes a small Ghidra processor spec so you can load a raw Black Oak read correctly and start reading the module in the same environment it was taken apart in.

* **`MPC565_BlackOak.pspec`** The processor spec. It presets the on chip memory regions and declares the small data bases, so Ghidra's own PowerPC analysis resolves the register relative accesses on its own, with no scripting.
* **`MPC565_BlackOak.ldefs`** The language definition. It registers a selectable PowerPC variant for this module that reuses Ghidra's bundled PowerPC core and compiler spec unmodified, so nothing here replaces or patches Ghidra itself.

Both files are original work that describe the module's factual memory layout. They contain no Ford code or data. You may use and redistribute them provided the copyright notice inside each file stays intact. They are provided as is, with no warranty.

## Loading this module in Ghidra

1. Copy both files into your Ghidra PowerPC language folder, `Ghidra/Processors/PowerPC/data/languages/`, then restart Ghidra. The variant **Ford MPC565 PCM, Black Oak 1472K CAN** now shows up in the language picker.
2. Import your raw read. Format **Raw Binary**, language the new **MPC565 Black Oak** variant, base address **0x0**, and import.
3. The on chip RAM and register regions and the small data bases are applied automatically from the spec, so register relative reads and writes land on real memory instead of falling into unmapped space.
4. These reads place the upper flash bank at file offset **0x70000**, but the code addresses it at **0x500000**. In the Memory Map window add one block for it, start `0x500000`, length `0x100000`, initialized from the file bytes at offset `0x70000`. The lower bank at `0x0` is already the main imported block. The spec file comments spell this out as well.
5. Run Auto Analysis with the defaults. Ghidra's PowerPC analyzer reads the small data bases out of the spec and recovers the register relative cross references on its own.

What this gets you is a clean, correct load and the cross references that Ghidra's standard analysis can recover by itself. Going from that starting point to a fully mapped module, every table and every live variable tied to the exact function that touches it, is the work this reference walks through, and doing that yourself is a good part of the point. The spec hands you a correct starting line. The rest is yours to earn, and it is very much worth earning.

# Part 1, Foundations

## What the Black Oak 1472K CAN PCM is

A powertrain control module is the computer that runs the engine and, on an automatic vehicle, the transmission. Black Oak is the internal Ford name for the control software family used on the 2004 to 2006 F150 and related trucks fitted with the 5.4 liter three valve modular V8, and 1472K CAN is the specific variant covered here, the one that communicates on a Controller Area Network bus. The PCM reads the sensors on the engine, decides how much fuel to inject, when to fire each spark, where to position the camshafts and the electronic throttle, how to manage the transmission, and it reports all of this to the rest of the vehicle and to a diagnostic tool.

Everything the PCM does happens dozens to thousands of times per second, in a fixed order, forever, while the key is on. It is a hard real time system, which means tasks are not just expected to produce the right answer, they are expected to produce it within a strict time budget tied to crankshaft rotation, because a spark that is correct but late is still a misfire.

## The processor and the way the firmware is organized

The PCM is built around a Freescale MPC5xx series microcontroller. This is a 32 bit PowerPC processor, it is big endian, which describes the byte order it stores numbers in, and it has a hardware floating point unit. That floating point unit matters more than it sounds, because it means the calibration values inside the PCM are stored as real IEEE 754 floating point numbers rather than the scaled integers that older engine computers were forced to use. In practice that means a table value is simply the number it represents, which makes the whole calibration far easier to reason about.

The firmware is split into two clearly separated layers, and understanding this split is the single most useful idea for anyone working on these modules.

The first layer is the **strategy**, which is the program itself, the compiled code that implements every control algorithm. The second layer is the **calibration**, which is the large body of tables and scalars that the strategy reads to make its decisions. Ford writes the strategy once and reuses it across many vehicles, and then ships a different calibration for each application. The engine behaves the way it does because of the numbers in the calibration, but what those numbers mean is defined entirely by the strategy code that consumes them. This is why genuine tuning knowledge is really knowledge of the strategy. A table by itself is just a grid of numbers until you know which function reads it, when, and what it does with the result.

## The peripherals that make it a real time engine controller

A general purpose processor cannot, on its own, catch a crankshaft tooth edge to the microsecond or fire eight coils at precise angles. The MPC5xx does that work in dedicated hardware blocks that run alongside the main processor.

The **Time Processor Unit**, or TPU, is effectively a small timing coprocessor with its own microcode. It captures the crankshaft and camshaft tooth signals and it generates the precisely timed output pulses that fire the injectors and charge the coils. Because the tooth decoding lives in TPU microcode rather than in ordinary program code, the main strategy does not decode teeth itself, it reads the already measured results the TPU hands back.

The **Queued Analog to Digital Converter**, or QADC, is a dual channel analog front end that scans the engine sensors and converts their voltages into numbers. The **Modular Input Output Subsystem**, or MIOS, is an on chip block of timers and pulse width modulation channels, used for outputs such as the camshaft oil control valves. The **TouCAN** block is the Controller Area Network controller that handles all bus communication.

Alongside the microcontroller is an external smart input output application specific integrated circuit, usually just called the ASIC. It provides additional pulse width modulation outputs such as the throttle motor and solenoids, it contains the analog to digital path for the oxygen sensors, and it houses a dedicated knock detection circuit. The practical consequence is that the PCM has two different pulse width modulation output systems, the on chip MIOS and the external ASIC, and two different analog front ends, the on chip QADC and the ASIC path used for the oxygen sensors. Knowing which one a given output or input belongs to is essential, because they are programmed in completely different ways.

A useful way to picture the whole module is as a fast general purpose brain surrounded by specialist helpers. The brain decides what should happen. The TPU, the QADC, the MIOS, and the ASIC turn those decisions into precisely timed electrical events and turn messy analog signals back into clean numbers. Most of the strategy is the brain talking to its helpers through a thin layer of hardware access functions.

# Part 2, The real time execution model

Before any single subsystem makes sense, it helps to understand the clock they all march to, because the timing of when code runs explains why it is written the way it is.

Work in the module happens on two different kinds of schedule. **Time based** tasks run on a fixed clock, a background loop that comes around every few milliseconds and handles things that do not need to be synchronized to the engine, such as temperature filtering, diagnostics, communication, and the slower accessory controls. **Angle based** tasks run once per cylinder event, triggered by crankshaft position rather than by the clock, and they handle everything that has to line up with combustion, fuel quantity, injection timing, spark advance, and dwell. The reason for the split is simple and physical. A spark decision is only meaningful relative to where the piston is, so it must be made per event, and the time available to make it shrinks as the engine speeds up. At idle there is a comfortable window between events. Near redline that same decision must be finished in a fraction of a millisecond.

On top of those two schedules sit **interrupts**, which are the highest priority work. When a crankshaft tooth arrives or a timed output must fire, the hardware interrupts whatever the processor was doing, services the event immediately, and returns. This is why the heavy, deadline critical timing is handled in the TPU and the interrupt handlers rather than in the ordinary loop, the loop simply cannot guarantee it will be looking at the right moment.

A practical consequence for anyone reading values on a scan tool is that different numbers update at different rates. A coolant temperature that changes slowly is filtered and refreshed in the background loop, while engine speed and load are recomputed every event. When two logged channels seem to disagree for an instant during a fast transient, it is often because they were sampled by tasks running on different schedules.

The details of the scheduler itself, how tasks are registered, prioritized, and dispatched inside this module, are part of the operating system and are **REDACTED** in Part 21. What is described here is the general model, which is enough to understand everything else in this reference.

# Part 3, Data, units, and the calibration format

## Numbers the way the module stores them

Because the processor has a hardware floating point unit, nearly every meaningful quantity inside the module is a 32 bit IEEE 754 floating point number. A temperature is stored as the actual temperature, a pressure as the actual pressure, a ratio as the actual ratio. This is a real break from older controllers, where every value was a scaled integer and you had to know the scaling and offset for each one before it meant anything. Here the strategy does its math in engineering units directly, which is a large part of why the calibration is as readable as it is once you know which function reads each value.

There are still integers in the module, and they are used deliberately where they are faster or more natural, counters, state numbers, bit flags, table indices, and a handful of time critical comparisons described in Part 5. A single byte is used for compact flags and small enumerations, and for a few values that get broadcast on the bus in a packed form.

Everything is big endian, meaning the most significant byte of a multi byte number is stored first. If you are reading raw bytes by hand this is the first thing to get right, because a four byte float read in the wrong byte order is nonsense.

## How a table is read

A scalar is a single number, a threshold, a gain, a limit. A table is a grid of numbers with one or two axes. A one dimensional table maps a single input, for example engine coolant temperature, to a single output, for example a fuel enrichment multiplier. A two dimensional table maps two inputs, most often engine speed on one axis and load on the other, to one output, for example spark advance.

The important mechanical idea is **interpolation**. The axes are defined by breakpoints, a list of input values at which the table has a defined output. When the live input falls between two breakpoints, the module does not jump to the nearer one, it computes a weighted blend so the output moves smoothly as the input moves. For a two dimensional table this is done in both directions at once, a blend of the four surrounding cells weighted by how close the operating point is to each, which is why it is called bilinear interpolation. The practical result is that the engine gets a continuous, smooth value everywhere between the defined points rather than a staircase, and it is why a well shaped table has smooth gradients rather than sharp cell to cell jumps that the interpolation would turn into abrupt changes in behavior.

The breakpoints are stored with each table rather than assumed, which lets the grid be uneven, packed tightly where the engine is sensitive and spread out where it is not. When you change the axis of a table you are changing where the engine looks, not just what it finds, so axis edits ripple through everything that shares that axis.

## Scaling for the outside world

A few values leave the module in a packed integer form for the communication bus, and those use a simple fixed scaling, a multiply by a constant and a pack into one or two bytes. The clearest example is the engine speed value the instrument cluster reads to drive the tachometer, which is the engine speed multiplied by a fixed constant and placed into two bytes. The cluster multiplies back by the inverse and moves the needle. This is worth knowing because it is the one place the real internal speed meets a human facing display, and the constant that ties them together is a fixed part of the broadcast format.

# Part 4, Sensing and inputs

## The analog pipeline

Almost every analog sensor on the engine reaches the strategy through the same pipeline, and learning that pipeline once explains every sensor at the same time. The QADC continuously scans its inputs into a set of result registers. A per channel accessor reads a result and averages a short burst of samples to reject electrical noise. A scaling step then converts the raw converter counts into real engineering units using a gain and an offset, so volts become degrees, or kilopascals, or grams per second. Finally a general purpose rationality filter applies range checking, debounce, slew rate limiting, and a lag filter, and only then does the finished value, together with any fault flags it raised, become the number the rest of the strategy uses.

A subtle but important point is that the channel a reader asks for is a slot in the conversion queue, not a physical pin. The mapping from queue slot to physical pin is set up once at initialization. What the strategy consumes, a given slot producing a given scaled sensor value, is fully defined, and the physical wiring simply sits underneath it.

The oxygen sensors are the one exception to this pipeline. They are not read through the QADC at all, they are read through the external ASIC's own analog to digital path, because their signal conditioning is handled in that chip.

## Thermistor sensors and how a temperature is recovered

The coolant and intake air temperature sensors are thermistors, resistors whose resistance changes with temperature. The module measures the voltage at a divider formed by that thermistor and a fixed internal resistor, which gives a voltage that moves with temperature but not in a straight line. The strategy converts that voltage back to a temperature through a transfer table that captures the nonlinear thermistor curve, so the stored output is a true temperature in engineering units. This is why these sensors read with good resolution in the middle of their range and coarser resolution at the extremes, the curve is steep in the middle and flat at the ends. The charge temperature reader also carries a failsafe, if the sensor reads implausibly hot or cold the strategy substitutes a sensible default so a single bad sensor does not drive the fuel and spark into a dangerous place.

## The sensor set

**Engine speed** begins at the crankshaft and is important enough that Part 5 is devoted to it. For sensing purposes the point is that engine speed is not read from an analog pin at all, it is derived from the timing of the crankshaft teeth captured by the TPU.

**Vehicle speed** is derived from the output shaft or driveshaft signal and is produced both as a precise floating point value and as a compact single byte value for broadcasting.

**Throttle position** on this platform uses two sensors on the electronic throttle body, and the **accelerator pedal** has its own separate redundant sensor pair. These are kept distinct because the pedal expresses what the driver is asking for, while the throttle sensors report where the throttle plate actually is, and the control system constantly compares the two. The redundancy is a safety feature. The two pedal signals track each other in a known relationship, and if they ever disagree beyond a tolerance the module knows a sensor has failed rather than trusting a possibly runaway request. The same cross check applies to the two throttle sensors.

**Mass airflow** is measured by a hot film sensor whose voltage is converted to a mass flow rate through a transfer table, which is one of the most important calibration tables in the whole module because airflow feeds fuel, load, and spark at once. **Manifold pressure** on this engine is normally not measured by a sensor at all, it is modeled from the airflow and the volumetric efficiency model, with a true sensor path available behind a configuration switch. **Barometric pressure**, the ambient air pressure that changes with altitude and weather, is established from the available air signals and is used to correct airflow, idle, and other calculations so the engine behaves consistently from sea level to the mountains.

**Coolant temperature** and **intake or charge temperature** are the thermistor inputs described above. **Fuel rail pressure and temperature** are read for the returnless fuel system. **Intake manifold runner control position** reports the state of the variable intake runners. The **returnless fuel pump duty feedback** closes the loop on fuel pressure. **Battery voltage** is filtered and then used in many places, including coil dwell, injector dead time compensation, throttle control, and alternator control, because almost every actuator behaves differently as supply voltage changes.

The **oxygen sensors**, front and rear on both banks, are read through the ASIC, filtered, and compared against a stoichiometric switch voltage with hysteresis to produce a clean rich or lean indication per bank. The front sensors drive fuel control, the rear sensors watch the catalytic converter. The **steering wheel cruise switches** are read as an analog resistor ladder on a single converter slot, described in Part 14.

# Part 5, Finding engine position and speed

This is the heart of a real time engine controller, and it contains some of the most elegant engineering in the whole module, so it is worth slowing down for.

## Why position is hard and why it needs two signals

To fire fuel and spark correctly the module must know two things, exactly where the crankshaft is in its rotation, and which stroke each cylinder is on. The crankshaft alone cannot answer the second question, because the crankshaft turns twice for every complete engine cycle, so any given crankshaft position occurs once on the compression stroke and once on the exhaust stroke, and those two need completely different treatment. The camshaft turns once per engine cycle, so a camshaft reference resolves the ambiguity. The module therefore reads a crankshaft position signal for fine resolution and a camshaft identification signal to establish which half of the cycle it is in, and it correlates the two to lock onto full engine phase.

## Teeth, the missing tooth, and how the TPU does the catching

The crankshaft signal comes from a toothed wheel passing a sensor, producing one electrical edge per tooth. The wheel has a deliberate gap, a missing tooth, that acts as a once per revolution landmark. When the TPU sees the longer than normal interval where the tooth should have been, it knows exactly where in the rotation it is. The TPU captures the precise time of each edge in hardware, so the measurement is not at the mercy of whatever the main processor happened to be doing at that instant, and it hands the strategy the finished tooth to tooth interval and the synchronization state. Because the actual edge decoding lives in TPU microcode, there is no ordinary software routine decoding teeth to inspect, the strategy consumes results.

## The clever part, speed as a period rather than a number

Here is the detail that surprises people the first time they see it, and the one that explains a lot about how the module stays fast. Engine speed in revolutions per minute is what humans think in, but the module often does not carry speed that way at all for its most time critical work. It carries the **crankshaft period**, the time between tooth edges. A faster engine produces a shorter period, a slower engine a longer one, and the two are simply inverses of each other, revolutions per minute is a constant divided by the period.

Why bother? Because measuring time between edges is exactly what the timing hardware already does, so the period is the natural, directly measured quantity, and converting it to revolutions per minute would mean a division every time, which is comparatively expensive. Working in the period avoids that division in the places that run most often. It gets better. Even the rev limit is stored and checked as a period. Being over the rev limit simply means a tooth arrived sooner than the limit period, so the overspeed check is a single comparison of two positive numbers, and positive numbers can be compared with plain integer comparisons, which are about the cheapest operation a processor can do. The module decides whether the engine is over the limit, the single most time critical safety check it makes, with one integer compare at the exact moment a tooth arrives. That is a beautiful piece of design, and it is a good example of how thinking about the physical quantity rather than the human facing number leads to faster, simpler code.

## Speed, the event index, and prediction

From the measured period the module computes engine speed where it needs the human facing value, and it produces the **cylinder event index**, the running count that tells the rest of the strategy which cylinder is next in the firing order so fuel and spark get scheduled to the right place. It deliberately keeps two speeds, a control speed used by the limiters and the fuel system and a separate map axis speed used to index the tables, so that table lookups do not jitter on raw tooth to tooth noise while control still sees the real value.

Because fuel and spark for an upcoming event must be computed slightly before that event actually arrives, the module also **predicts** forward, projecting where the crankshaft and the air charge will be a fraction of a cycle ahead and scheduling for that predicted point rather than for where the engine was when the math started. At high speed the difference between where the engine was and where it will be is large, so this prediction is a real part of getting timing right rather than a refinement.

## Synchronization as a state machine, and losing sync

Catching phase is not instant, so a synchronization state machine manages the process. It moves through reset, acquisition, confirmation, and running states. While it is unsynchronized, during cranking, or after a stall or a missed tooth event, it holds the global combustion enable flag clear so no spark or fuel is scheduled against a position it is not sure of. Once it has seen enough consistent teeth and correlated the camshaft reference, it confirms sync and enables combustion. Supporting functions handle each acquisition phase, take a snapshot of the tooth pattern, detect the missing tooth and any abnormal gap, gate on threshold and speed, select the right timing calibration, and pack a diagnostic status word. A hardware enable toggles the ignition driver power in step with that flag. This is also the mechanism that protects the engine if the crank signal is lost at speed, the moment sync is in doubt the module stops committing to positions it cannot trust.

## The injector and timing channel layer

Underneath the schedulers is a low level hardware abstraction layer, a collection of small functions that write an injector pulse width, clear a channel, perform a bank write, encode an injection angle, set a channel's mode bits, arm a channel, read a channel's captured value, and read its status, along with the dwell and spark angle writers. These are the direct interface to the timing hardware, and naming them is what makes the higher level schedulers legible rather than a wall of raw register operations.

# Part 6, Air charge and load

## The quantity everything depends on

If you only deeply understand one subsystem, make it this one, because nearly every other table in the module is indexed by the number it produces.

The PCM estimates how much air each cylinder is taking in, the cylinder air charge, using three methods and blending or selecting among them by operating condition. The first is the **mass airflow** method, which is primary whenever the airflow sensor is valid, it measures the air entering the engine directly. The second is the **speed density** method, which infers air charge from the modeled manifold pressure and a volumetric efficiency model. Volumetric efficiency is the ratio of the air a cylinder actually ingests to the air it theoretically could hold at the current conditions, and the model blends several volumetric efficiency tables according to the current camshaft advance, so camshaft timing directly reshapes the air estimate. The third is a **throttle based** method, sometimes called alpha N, which estimates air from throttle angle and speed and is used when the airflow sensor is unavailable or at very low openings.

Around these three methods sits a chain of supporting steps, a manifold pressure validity check, a source selection, an airflow rationality gate, a key on high flow check, a transient filter for throttle tip in and tip out, a current charge calculation, and the predictive estimators that project the charge forward as described in Part 5.

## Load, and why errors here spread everywhere

The output of all of this is engine load, defined as the cylinder air charge divided by a reference standard air charge. Load is the vertical axis of essentially every fuel, spark, cam, and knock table in the module. The subsystem also computes an air density correction, a normalization divisor, a small leak and offset compensation, and a measure of how close the engine is to wide open throttle. Because load is produced here and consumed everywhere, any error here propagates into fuel and spark across the entire map. That is the single most important practical fact about this subsystem, and it is why airflow and load are the first things to verify after any significant hardware change such as a camshaft, intake, or injector swap. If the module's idea of how much air is in the cylinder is wrong, every table downstream is being read at the wrong place, and no amount of editing those downstream tables will truly fix it.

## Transient fuel, the wall wetting model

During rapid throttle changes the air moves faster than the fuel, because some fuel momentarily condenses as a film on the port walls and intake surfaces, then evaporates again. On a throttle tip in extra fuel is needed to both fill the cylinders and build that film, or the mixture goes briefly lean and the engine stumbles. On a tip out the film evaporates and adds fuel the engine no longer wants, so fuel must be pulled. The transient filter models this film forming and evaporating and trims the delivered fuel accordingly, which is what makes the difference between a crisp, clean throttle response and a flat spot or a rich sag on transitions. It is one of the least glamorous and most felt parts of a good calibration.

# Part 7, Fuel

## From air to an injector pulse

Base fuel quantity comes directly from the air charge and a target air to fuel ratio, the module simply computes the fuel mass that pairs with the measured air to hit the target ratio. That fuel mass is then converted into an injector opening time, the pulse width, by a dual slope injector model. Real injectors do not flow linearly across their whole operating range, so the model uses a low slope and a high slope joined at a breakpoint, adds compensation for fuel pressure since higher differential pressure pushes more fuel through the same opening, and enforces a minimum pulse width below which an injector cannot be trusted to open repeatably. The finished pulse width is handed to the timing hardware to be delivered at the correct crank angle. This injector model is the group that must be corrected when injector size changes, because the whole fuel calculation assumes it knows exactly how much fuel a given pulse delivers.

The module contains two injection schedulers, a sequential scheduler that fires each injector individually in the firing order for normal running, and a bank or simultaneous scheduler used during starting when full phase may not be established yet. Both can divert into a shared injector disable routine that simply commands no fuel, which is the common path used by decel shut off and by protection logic.

## Closed loop fuel trim

Open loop fueling relies on the models being perfect, which they never are, so the PCM measures the result at the front oxygen sensor and corrects. A closed loop fuel manager runs this correction for each bank. The short term controller is the classic Ford exhaust gas oxygen integrator. Each time the oxygen sensor switches between rich and lean it jumps the correction by a step, then ramps it gradually, incorporating a transport delay that accounts for the time for exhaust to travel from the valve to the sensor, a hold time, and a bias drawn from speed and load tables along with contributions from the rear sensor and from air conditioning load. The result is a short term fuel multiplier per bank, bounded by a limit manager that also governs open loop defaults. When the system is in open loop, for example at wide open throttle or before the sensors are warm, the trim is forced back to the neutral stoichiometric value, because at those times the sensor reading is not a trustworthy basis for correction.

## Adaptive fuel learning

Short term trim corrects instantly but forgets instantly. The long term adaptive system, usually called keep alive memory or KAM, watches the short term trim and, once it has been stable long enough as judged by a maturity counter, learns the steady state error into cell tables held in battery backed memory, indexed by airflow or load. These learned corrections are clamped to minimum and maximum limits, they adapt faster while the engine is warming up, and they are applied to the injector pulse so that the short term trim can stay centered near zero. A configuration switch selects whether the learning is referenced to measured airflow or to the speed density model. The single most important practical fact about this system is that if adaptive learning is left enabled it will slowly compensate for deliberate fueling changes, effectively undoing them over time, so it is normally disabled while fuel is being developed and only re enabled once the base fuel is correct.

## Cranking, cold start, and warm up

Starting an engine needs its own fuel logic because almost none of the running assumptions hold yet. A mode selector chooses sequential or simultaneous injection based on the temperature at the moment of the start. A family of cranking tables sets the target ratio during cranking, a fuel multiplier that decays with time since the start, a wall wetting or lost fuel term that accounts for fuel that condenses on cold surfaces, an injection delay, and a dedicated cranking spark value. Because the airflow transfer table sits upstream of fuel, load, and spark all at once, it influences every one of these, which is another reason it matters so much.

## Decel fuel shut off

When the driver lifts off at speed the engine does not need fuel to keep spinning, so the PCM can shut the injectors off entirely to save fuel and reduce emissions. An entry function arms a decel cut when the throttle is closed and engine speed, vehicle speed, the speed to gear ratio, and load are all above their thresholds, with separate lower thresholds for exiting so the function does not rapidly toggle. An apply stage turns that into the actual injector cut with controlled ramp in and ramp out, and a transition stage smooths the return of fuel so it is not felt as a bump when the engine comes back onto fuel.

## The fuel supply, returnless pressure control

This engine uses a returnless fuel system, meaning there is no return line carrying excess fuel back to the tank, instead the in tank pump is controlled to deliver just the pressure demanded. The module commands the pump duty and uses the fuel rail pressure and the pump duty feedback to hold the target pressure. Keeping rail pressure where the injector model expects it is part of keeping fuel accurate, because the injector flow compensation assumes a known pressure, if the real pressure drifts the delivered fuel drifts with it.

## Flex fuel and purge compensation

Where ethanol content varies, the strategy carries handling that adjusts fueling and timing for the different stoichiometric ratio and knock behavior of ethanol blends. Separately, the evaporative purge system periodically draws fuel vapor from the charcoal canister into the intake, which adds uncommanded fuel, so the fuel control compensates for the estimated vapor contribution while purge is flowing so the mixture stays on target. Both are examples of the fuel system accounting for fuel that arrives by a path other than the injectors.

## Airflow plausibility monitoring

Separate from the airflow transfer that computes fuel, a monitoring dispatcher runs the plausibility diagnostics for the airflow sensor. Several rate limited filters feed accumulators that set circuit fault codes when the signal is out of range. This path is diagnostic only and does not itself compute fuel, which is a common point of confusion, the fuel path and the monitoring path both read the airflow sensor but for different purposes.

# Part 8, Spark and combustion timing

## Computing spark advance

Spark timing is expressed as advance, the number of crankshaft degrees before top dead center at which the mixture is ignited. A per cycle spark advance manager coordinates the whole calculation. It begins with a best torque lookup, the timing that would produce maximum torque if nothing limited it, usually abbreviated MBT. In parallel it computes a borderline value, the most advance that is safe before the onset of knock, built from a knock limited table with added corrections for intake geometry such as the runner control state and for camshaft phase, then interpolated against load and speed and modified by a global octane term. A ramp and limiter slews the working value toward its target using knock recovery rate tables, so timing is restored gradually rather than snapped back after a correction.

## The arbiter, the most important idea in spark

The decisive step is the arbiter. It takes the minimum across every candidate, the best torque ceiling, the borderline limit, a torque model ceiling, the accumulated knock retard, and an overspeed floor, and it records a code identifying which term won. The result is written into the per cylinder advance array. Because the arbiter always takes the minimum, on any knock prone combination the borderline table is what actually governs timing under load, since it is usually the lowest of the candidates there. That is the single most important thing to understand about tuning spark on this platform, the table that sets your timing under load is whichever one is lowest at that operating point, and most of the time under load that is borderline.

The torque model ceiling deserves its own mention. The engine's torque is modeled as a function of spark, and two functions invert that relationship, using coefficient tables indexed by speed and load, to find the exact amount of spark retard from best torque that will deliver a requested torque target. This is how the torque based control system reaches into spark, and it is completely separate from the knock sensor retard. When the transmission or the traction system asks for less torque, this is one of the levers that answers, quickly, by pulling timing.

## Scheduling the spark angle and charging the coils

Once a final advance is chosen it must become physical coil events. An angle scheduler converts the commanded advance into a per cylinder end of dwell angle, applies the firing order remap and the coil pairing arrangement, and sets the per cylinder event masks. A dwell update then runs for each spark event. Dwell is the length of time a coil is charged before it fires, and getting it right matters because too little dwell gives a weak spark while too much wastes energy and overheats the coil. The dwell computation uses a measured charge feedback multiplied by a harness compensation term with battery voltage correction, or a fixed dwell with a per cylinder trim, clamps the result between voltage derived minimum and maximum limits, and writes each coil channel and its firing angle to the timing hardware. The ignition is coil on plug, one coil per cylinder. A coil setup function builds the dwell versus voltage relationship and the multi strike configuration, and a repetitive spark controller manages multi strike firing at low speeds, where firing the plug more than once per event improves combustion stability during idle and starting.

One architectural detail is worth stating because it is easy to misread. The per cylinder fire mask that appears in this path is a software record used for misfire bookkeeping. It is not the signal the coil hardware fires on. The physical coil charging is governed by the dwell value written to the timing hardware.

## The spark and fuel enable gate

A single global flag enables or disables combustion. A spark and dwell event scheduler checks that flag, along with a spark blanking counter, and when disabled it writes zero dwell to every channel so no coil charges. That same flag is also read by the fuel injector scheduler, which diverts to the injector disable routine when it is set. The consequence is that the one factory global kill cuts spark and fuel together, it is not a spark only mechanism. That flag is owned by the crankshaft synchronization state machine described in Part 5, and during normal running it stays clear. Understanding that this one factory flag takes both spark and fuel is important context for appreciating why a true spark only cut is a different thing entirely, which is part of the custom material in Part 21.

# Part 9, Knock

Knock, also called detonation, is the uncontrolled secondary ignition of the end gas in the cylinder, and sustained knock destroys engines, so detecting and suppressing it is a safety critical function. On this platform the per cylinder knock detection and the retard integration are performed in the dedicated external knock circuit rather than in the main strategy. The firmware configures that circuit and reads its results, which is why the per cylinder knock indications are referenced in relatively few places in the main code, the heavy signal processing happens in the chip.

The firmware side consists of a sense setup that programs the per cylinder gains and the readback scaling into the chip, a detect setup that computes the per cylinder knock thresholds from speed and load tables along with the noise and knock crank angle windows and the enabling conditions of load, coolant, and speed, a rate scheduler that chooses between fast and normal retard and recovery ramp rates based on a transient counter, a rate update that pushes those rates into the chip and rearms the detection window each cylinder event, a sensor monitor that compares the measured knock magnitudes against noise tables to set sensor circuit faults, and a coordinator that prevents the knock self test and the oxygen sensor self test from running at the same time. The accumulated retard then limits spark through the arbiter's knock term, bounded by a retard authority limit.

The crank angle windows are worth understanding because they are what let the system tell real knock from engine noise. The circuit only listens during the window where knock would occur, for the frequency band knock produces, and it compares what it hears against a noise reference that scales with speed and load, so ordinary mechanical clatter does not trip it but a genuine knock event does. The practical takeaway is that because the detection and integration happen in hardware, the retard algorithm itself is fixed. Its gains, windows, rates, and authority are all adjustable through calibration, but the fundamental behavior cannot be rewritten by changing a table.

# Part 10, Variable cam timing

This engine uses twin independent variable camshaft timing, with each cylinder bank having its own camshaft position sensor, its own control loop, and its own oil control valve that hydraulically advances or retards that camshaft. Variable cam timing lets the engine optimize valve events across the operating range, improving low speed torque, high speed power, idle quality, and emissions, all from the same hardware. On this three valve engine the cam timing authority is large and the air model depends on it, which is why Part 6 stressed that cam advance reshapes the air estimate.

The control is a five stage chain. An enable and ramp stage builds a normalized authority for how much cam advance is currently allowed, gated by the absence of knock, the absence of faults, and a temperature window, and reduced when knock is detected. A scheduling stage reads speed and load target tables, trims them for oil temperature since oil viscosity affects how quickly the mechanism can move, scales by the authority, and filters to smooth per bank target angles. A control stage, a proportional integral derivative loop, drives each bank's valve duty from the error between the target angle and the measured angle. A duty calculation stage converts that into the actual oil control valve command, with an open loop override and limp defaults for fault conditions. A hardware output stage writes both banks to the on chip pulse width modulation and latches any camshaft sensor faults. Because detected knock pulls cam advance back, this subsystem is also a useful window into how the whole engine responds to knock, and because cold oil moves the mechanism slowly, cam response is deliberately softened until the oil warms.

# Part 11, Torque based control, throttle, idle, limiters

## Driver demand as a torque request

A defining characteristic of this generation of control is that the accelerator pedal does not command throttle opening directly, it commands torque. The pedal position is linearized into an angle, then driver demand tables indexed by pedal and engine speed produce a desired engine torque and a desired wheel torque, with separate low range variants. A torque demand control loop converts the desired torque into a throttle area command, applying traction related gains and clipping limits, and feed forward and effective area models convert that area into a throttle angle. A final angle scheduler resolves the commanded throttle angle and applies a maximum throttle versus speed authority cap, which is the lever behind reduced power modes, and publishes the overall control state. Thinking of the whole system as a torque request flowing down to a throttle position, rather than a cable pulling a plate, is the key mental model, and it is why so many different systems can ask the engine for more or less torque and have it cooperate.

## The throttle position servo

The electronic throttle must be held precisely at the commanded angle against a return spring and airflow forces, which requires a closed loop position servo. An acquire and filter stage produces two filtered throttle position feedbacks. A position error monitor compares commanded against actual and raises throttle faults on persistent error. A duty output stage converts the commanded angle into a motor pulse width, with fixed safe duties in fault modes, and drives one channel of the external ASIC, since the throttle motor is driven through an H bridge on that chip. Supporting this are mode arbitration, a drive disable that releases the plate to its limp home spring position, a relay enable state machine, a cold start enable gate, and fault reporting through a diagnostic maturation manager.

## Why electronic throttle is wrapped in so much monitoring

Electronic throttle is safety critical, because a throttle that opens when it should not is dangerous in a way that a fuel or spark error usually is not. This generation of control therefore uses a layered monitoring philosophy, the same idea the industry adopted across electronic throttle systems. The main control computes what the throttle should do. An independent monitoring layer continually checks that the torque the engine is actually producing is no more than the driver asked for, and if it ever sees more torque than is justified by the pedal it has the authority to intervene, cut power, and force the throttle toward its safe limp position. The limp position itself is mechanical, the plate is spring loaded to a small default opening, so that if the motor is released the engine can still idle and limp but cannot run away. This is why you see so many throttle and pedal rationality checks, cross checks between redundant sensors, and fault reactions in this area, the system is designed so that no single failure can produce unintended acceleration.

## Idle control

On this platform there is no separate idle air valve, idle air is metered by the same electronic throttle. A target function schedules the desired idle speed from base values for drive and neutral, with variants for air conditioning and cold start, a barometric correction, and a lag when shifting into gear. An idle control loop, with separate gains for opening and closing the throttle chosen by the direction of the error, outputs an idle airflow demand that is summed with the driver demand and bounded by idle limits and the closed throttle calibration. Feed forward idle airflow tables and a decel to idle dashpot support it, and a compensation raises the target under fan and air conditioning load so accessory engagement does not drag the idle down. A good idle calibration is mostly about the feed forwards doing the heavy lifting so the closed loop only has to trim, rather than the loop chasing every disturbance after the fact.

## The limiters

Understanding the limiters correctly prevents a great deal of confusion. The factory rev limiter and vehicle speed limiter are soft limiters, they are control loops that command the electronic throttle closed, and they do not cut spark or fuel. They are only active while the throttle is in normal control. A park and neutral clip caps torque and adds a speed limit off load so the engine is not revved to the moon in neutral. Separately, the rev limit is also expressed as a crankshaft period that a hard overspeed check compares against, and that hard path is the one that will resort to a fuel cut and a spark retard floor in a true overspeed event. So the limiter a driver normally feels is a throttle closer, a soft lean on the throttle, not a hard cut, and that difference in feel is entirely by design.

## The torque model as the common currency

Tying the throttle, spark, and transmission sections together is a central torque model. It estimates the engine's current torque and inverts the torque to spark relationship to deliver requested torque reductions quickly through spark retard, and more slowly through airflow. The transmission requests torque reduction during shifts, the traction system requests it during wheel slip, and the idle and limiter systems request it as well. This common torque currency is what lets all of these systems cooperate rather than fight, each one speaks in torque, the engine arbitrates the requests, and the throttle and spark carry out the result.

# Part 12, Transmission, the 4R75E

Automatic applications pair this engine with the 4R75E, a four speed automatic, and the PCM contains its full controller, so the same computer that runs the engine also runs the gearbox and coordinates the two.

The behavior rests on several cores. A **range and over rev guard** validates the allowed gear from the selector position, the current gear, and vehicle speed, and inhibits manual downshifts until speed drops into a safe range so the engine cannot be over sped by a careless downshift into too low a gear. The **shift schedule** tables, indexed by throttle and speed, set the speeds at which the transmission changes gear for a given throttle, these are the tables most people think of as shift points. A **torque converter clutch** apply scheduler builds the clutch apply pressure from per gear engagement rate arrays and the measured converter slip, capped at a maximum, which controls how firmly and quickly the converter locks up, and a release path manages unlocking cleanly. A **line and electronic pressure control** stage computes the main hydraulic pressure from a torque term with per shift floors and converts it into the pressure control solenoid command, and this is the primary lever for shift firmness, because higher pressure holds more clutch capacity at the cost of harshness.

The part that most shapes how a shift feels is the **shift torque management**. During the actual gear change the controller coordinates a brief engine torque reduction with the transmission, pulling engine torque through spark while the clutches swap, then restoring it, so the shift is smooth rather than a bang. The timing and depth of that torque pull, set against the pressure schedule, is the difference between a soft factory shift and a firm performance shift. The strategy also runs the shift as a sequence of phases, a fill phase to take up clearance, a torque phase where the oncoming clutch takes load, and an inertia phase where engine speed is pulled to the new gear, each with its own pressure and timing, which is why shift quality tuning is as much about phase timing as about raw pressure. Gear ratio monitoring continuously compares input and output speeds against the expected ratio for the commanded gear, both to confirm shifts completed and to catch slip and mechanical faults.

# Part 13, Engine protection and failure handling

A good controller is judged as much by how it fails as by how it runs, and this module has a deep set of protections that are worth understanding both for safety and because they can quietly limit a modified engine if you do not know they are there.

**Overspeed protection** is the hard rev limit described in Part 5 and Part 11, enforced as a period comparison, capable of cutting fuel and flooring spark if the soft throttle limiter is somehow exceeded. **Over temperature protection** is not a fan table, it is a separate staged cylinder cut path, if coolant or cylinder head temperature climbs into the danger range the module progressively disables cylinders to reduce heat production while still letting the truck limp, trading power for survival. **Oil and charging faults** fold into warnings and derates. **Sensor failures** each have a defined fallback, a failed temperature sensor substitutes a default, a failed airflow sensor shifts the air model to the throttle based method, a failed cam sensor drops cam control to a limp default, so a single lost signal degrades behavior gracefully rather than stopping the engine.

The throttle and torque monitoring described in Part 11 is the most safety critical of these, with the authority to cut power and force the throttle to its mechanical limp position if the engine ever makes more torque than the driver requested. Across all of these the design pattern is the same, detect the fault, substitute a safe default or a reduced authority mode, set a diagnostic code, and keep the vehicle controllable. For a tuner the important awareness is that several of these protections are calibrated with thresholds, and a modified engine that runs hotter, revs higher, or flows more air than stock can walk into a protection that was never meant to limit it, so knowing they exist is the first step to not being surprised by them.

# Part 14, Driver interface and cruise

The steering wheel cruise control buttons are implemented as an analog resistor ladder read on a single converter slot, which means each button produces a distinct voltage on one wire rather than a separate digital signal per button, and on this variant they are not broadcast on the communication bus. An acquire function classifies the ladder voltage into a button code with a short debounce to reject noise and contact bounce, and a decode function produces the button command while also maintaining a hold timer measuring how long a button has been held and flags for held past a threshold, which is how hold to accelerate and hold to coast are distinguished from a tap.

A state machine runs the cruise logic across its off, armed, and engaged states. It is disabled by the brake, the clutch on manual applications, cranking, and over temperature conditions, and it will not engage outside a sensible speed range. When engaged it becomes one more torque requester, asking the engine for the torque needed to hold the set speed, which the torque model carries out through the throttle, the same cooperative mechanism as everything else. Both the acquire and decode functions run every loop regardless of whether cruise is active, so the current button state and hold time are always available to the rest of the strategy. The general operation of the switch and the stock cruise state machine is described here for completeness. Any use of these inputs beyond stock cruise is part of the custom material and is **REDACTED** in Part 21.

# Part 15, Communication

The PCM talks to the rest of the vehicle over a Controller Area Network bus. Two primitives underlie all of it, a frame transmit function and a mailbox read and configure function, together with a queued broadcast loader that schedules the outgoing messages. CAN works by messages with identifiers rather than addressed point to point links, every module hears every message and acts on the ones it cares about, which is why a single broadcast of engine data can feed the cluster, the transmission side, and a scan tool at once.

On the transmit side the PCM broadcasts engine data, including the frame the instrument cluster reads to drive the tachometer, which carries engine speed in its first bytes along with vehicle speed and pedal position, a gear and shift state frame, a status frame carrying the transmission range selector field and various flags, a manual or automatic indicator, and a proprietary multi byte data packet. On the receive side it accepts transmission torque and temperature from the transmission side in an offset encoded form, brake, clutch, and neutral status, antilock brake wheel speeds, and mode flags, and it answers diagnostic requests. The speed, vehicle speed, and pedal fields use a simple binary scaling, so for example the tachometer value is engine speed multiplied by a fixed constant packed into two bytes, which is the one place the internal engine speed becomes a number on a gauge, as noted in Part 3.

# Part 16, Diagnostics, monitors, and OBD

## How a data request is answered

All diagnostic data requests are answered from a single sorted descriptor table of several hundred entries. Each entry carries the parameter or data identifier, a mode byte distinguishing current data from freeze frame data from the enhanced manufacturer specific space, a handler type, a pointer to the live source of the value, and formatting arguments. A request is resolved by a binary search on the mode and identifier, dispatching either to a computed path that reads the live source, applies availability gating, and formats the result, or to a simpler raw path. The table includes the standard legislated parameters such as speed, pressures, airflow, timing, load, commanded ratio, fuel trims, oxygen sensors, and catalyst temperatures, as well as a large set of manufacturer enhanced identifiers. This single table driven design is why the same lookup answers both a generic scan tool and a factory tool, the mode byte selects which world the identifier lives in.

## Trouble codes, maturation, and healing

A diagnostic trouble code is not set the instant something looks wrong, because a single glitch should not light a lamp. Faults mature. A condition must persist or recur across a defined window, often across more than one drive, before the code is confirmed and the malfunction indicator lamp is commanded on. Conversely codes heal, once the condition has been absent for enough clean drives the lamp turns off and the code eventually clears itself. This maturation and healing is handled by a general trouble code manager used throughout the strategy, so every monitor sets and clears codes through the same consistent logic.

## The emissions monitors

Separately from the live data, the emissions monitors are calibration driven state machines that decide when a system has drifted far enough to illuminate the lamp. They do not change how the engine runs, they watch. The **misfire monitor** infers misfires from crankshaft speed deviation, since a cylinder that does not fire fails to accelerate the crankshaft as expected, and the module can often tell which cylinder from where in the rotation the deceleration appears, gated by torque thresholds and speed and coolant windows so rough roads and other disturbances do not create false misfires. The **catalyst efficiency** and **rear oxygen** monitors are gated by coolant, air temperature, vehicle speed, and a minimum rear sensor temperature, and judge converter health by how much the rear sensor mirrors the front. The **evaporative purge** monitor runs a multi phase flow and leak test gated by operating conditions and time limits. The **oxygen sensor heater** monitors watch heater current and duty so a lazy sensor that is not reaching temperature is caught.

## Readiness and the drive cycle

Each monitor reports a readiness status, whether it has had the conditions it needs to run and complete since the codes were last cleared. This is why a vehicle that was just cleared will show monitors as not ready until it has been driven through the right sequence of conditions, the drive cycle, that lets each monitor run. Understanding readiness explains a lot of real world diagnostic behavior, a truck can be running perfectly and still not pass an inspection simply because the monitors have not completed yet.

# Part 17, Accessories and charging

A smart alternator and charging controller provides a load responsive charging voltage rather than a fixed setpoint, raising or lowering the commanded voltage with electrical load and conditions, and it raises the idle target so electrical load does not drag the idle down. Cooling fan control is threshold based on coolant and cylinder head temperature with an idle increase when the electric fan engages, and as noted the over temperature engine protection is a separate staged cylinder cut path rather than a fan table, the fan control and the protection are two different things that happen to watch the same temperatures. Air conditioning engagement adds an idle compensation in drive and neutral and is coordinated so the compressor can be briefly shed under hard acceleration. Exhaust gas recirculation, where fitted, is selected by a hardware type switch and is minimal on this engine family. None of these are glamorous, but they are a large part of why a stock truck idles steadily with the air conditioning on, the lights on, and the fan running, the module is actively compensating for every one of those loads.

# Part 18, From key on to idle, the start sequence

It is worth walking the whole thing once as a story, because it ties the subsystems together in the order they actually wake up.

When the key comes on, the module powers up and the low level startup brings the processor and its peripherals to life, this earliest region is proprietary and is **REDACTED** in Part 21. The strategy then initializes its variables, reads the battery backed learned data so the engine starts with what it learned last time, and sets up the hardware access to the TPU, the QADC, the MIOS, the ASIC, and the CAN controller. Sensors begin converting, temperatures and voltages are read and filtered, and the module establishes baseline conditions, how warm the engine is, what the barometric pressure is, how much fuel pressure it has.

The driver cranks. Now the synchronization state machine from Part 5 goes to work, watching the crankshaft teeth and the camshaft reference, searching for the missing tooth and the phase. Until it is sure, combustion stays disabled. The cranking fuel and spark logic from Parts 7 and 8 takes over with its own temperature based tables, the simultaneous injection mode delivers fuel before full phase is even established, and the cranking spark value fires once there is enough sync to fire safely.

Sync confirms. The global combustion enable flag goes clear to allowed, sequential injection takes over from the cranking mode, spark moves from the cranking value onto the running calculation, and the engine catches. The after start enrichment holds extra fuel for a few seconds and decays it out. The idle control takes the reins and pulls the engine to its target idle, higher at first for a cold engine, settling as it warms. The oxygen sensors heat up, and once they are hot and the engine is warm enough the fuel control closes the loop and the adaptive learning begins trimming. Within a minute or two the whole system has walked from a cold, unsynchronized, open loop start to a warm, synchronized, closed loop idle, every subsystem in this document handing off to the next in order. Seeing it as that sequence makes the individual parts click into place.

# Part 19, The calibration surface

The calibration is the tuning surface, the set of thousands of scalars and single and two dimensional tables that shape the engine's behavior. The mechanics of how a table is read, the breakpoints and the interpolation, are covered in Part 3, this section is an overview of the major groups and what each contains, named by the generic parameter identifiers that describe their function.

**Adaptive fuel learning** contains the master enable for adaptation, the maximum and minimum learned correction limits, the warm up gain, the temperature and load thresholds that allow learning, and the reset behavior. This is the group that quietly undoes fuel changes if adaptation is left on.

**Closed loop fuel** contains the jumpback step and amplitude of the oxygen integrator, the hold time fraction, the modulation limits, the short term trim clamps, the rich and lean authority rates, and the peak to peak amplitude behavior.

**The fuel injector model** contains the low slope, the high slope, the breakpoint between them, the minimum pulse width, the reference pressure, and the compensation tables for breakpoint, slope, and battery voltage versus pressure. This is the group that must be changed when injector size changes.

**Cranking and cold start fuel** contains the cranking ratio, the fuel multiplier versus time since start, the wall wetting lost fuel, the injection delay, and the cranking spark.

**Decel fuel shut off** contains the entry and exit speeds, the load ceiling, the hysteresis thresholds, and the target ratio as fuel returns.

**Spark** contains the main advance tables with their speed and load axes, the failsafe cooling spark modifier, the lugging retard, and the best torque set.

**Borderline and knock spark** contains the borderline knock table, the ethanol content adjustments, the multiplier on the borderline value by ethanol, and the modifier used when a knock sensor has failed. This is where real timing under load is set.

**Knock sensor** contains the per cylinder gains, the knock and noise crank angle windows, the thresholds by speed and load, the retard and recovery rates, and the retard authority.

**Air, load, and speed density** contains the airflow transfer table, the volumetric efficiency tables blended by cam advance, the reference standard air charge, the air density temperature table, the transient fuel terms, and the charge predictors.

**Variable cam timing** contains the speed and load target tables per bank, the oil temperature trim, the authority ramp, the control loop gains, and the oil control valve duty limits.

**Electronic throttle, driver demand, and idle** contains the pedal linearization, the driver demand torque tables indexed by pedal and speed, the torque control loop gains, the maximum throttle versus speed authority cap, the idle target schedules for drive, neutral, air conditioning, and cold, the idle loop opening and closing gains, and the dashpot.

**Limiters** contains the soft rev and speed limit gains and setpoints, the park and neutral clip, and the hard overspeed limit.

**Transmission** contains the shift schedule tables indexed by throttle and speed, the gear ratio error thresholds, the torque converter clutch apply and release and engagement rate arrays, the line and electronic pressure control schedules that set shift firmness, the shift pressure and phase tables, and the torque reduction targets.

**Torque model** contains the torque to spark model coefficients, the torque limits, and the driver demand torque tables.

Further groups cover the alternator and charging, the axle ratio and tire size, evaporative and canister purge, the decel dashpot, ethanol and flexible fuel handling, the fuel pump and returnless pressure control, intake manifold runner control, airflow monitoring, the front and rear oxygen sensors, traction control, transient fuel, and the vehicle identification block.

# Part 20, Memory and regions

At a conceptual level the module's memory divides into a region of flash that holds the startup and low level code, a region that holds the main application strategy, a region that holds the large body of calibration tables and scalars, a region of random access memory that holds all the live values, sensor readings, state machines, and the battery backed learned data, and the input output space where the timing hardware, the converters, the pulse width modulation blocks, the communication controller, and the external chip all appear to the processor. The battery backed portion of RAM is what lets the module remember its fuel learning, its adaptive shift pressures, and its trouble codes across key cycles. The exact boundaries, and anything concerning the startup and low level region in detail, belong to the proprietary material in Part 21 and are not published here. The Ghidra spec in this repository sets up the regions you need to load and analyze a read, and nothing beyond that.

# Part 21, Proprietary internals, explained and REDACTED

This section exists so that the reference is complete in describing what the module contains, while keeping the sensitive implementation details proprietary to Trent's Tuning LLC. For each item below, the general concept is explained, and the Black Oak specific implementation is marked REDACTED. These are redacted on purpose, so that this reference teaches how the module works without handing anyone the keys to the parts that can brick a module, defeat its protections, or reproduce the custom features.

## The bootloader

A bootloader is the first software that runs when the module powers up, before the main engine strategy takes control. Its general job is to initialize the processor, decide whether the main application is valid and should be allowed to run, and provide the controlled entry point that a factory programming tool uses to erase and rewrite the module. It is the gatekeeper between a module that runs its normal code and a module that has dropped into a programming mode. The Black Oak bootloader structure, its entry points, its validation behavior, the way it hands control to the application, and anything that would let someone write over the bootloader region are **REDACTED**.

## The operating system kernel

The kernel is the small real time operating system that schedules the control tasks, services the hardware interrupts, manages timing, and arbitrates access to shared resources so that the many subsystems described above can run together without interfering. The general model is explained in Part 2. The Black Oak kernel internals, its scheduler, and its service interfaces are **REDACTED**.

## Reading and writing the module, reflashing

Reprogramming a module means erasing its flash memory and writing new contents, which is how both a factory update and a custom calibration are installed. In general this is done over the diagnostic connection using a defined sequence of steps that places the module into a programming state, transfers the new data, and verifies it, and it is constrained by how the flash hardware itself can be erased and written. The specific paths by which Black Oak is read and written, the sequences involved, and the practical methods are **REDACTED**.

## Security access, the seed and key

Security access is the lock that prevents unauthorized reprogramming and unauthorized access to protected functions. The general mechanism is a challenge and response. The module issues a random value, the seed, the tool must transform that seed through a secret algorithm into the correct answer, the key, and only a tool that knows the algorithm can produce a key the module accepts. This is what stops arbitrary tools from rewriting the module. The Black Oak seed and key algorithm and the security access details are **REDACTED**.

## Firmware validation and checksum

A checksum is a value computed across a region of the firmware that lets the module verify its own contents have not been corrupted or tampered with. In general the module computes these values at startup or in the background and compares them against stored expected values, and if they do not match it can refuse to run normally and set a fault. For any modified calibration to be accepted, these values must be made consistent with the new contents. The Black Oak checksum scheme, the regions it covers, the algorithms, and how a modified image is made valid are **REDACTED**.

## The custom operating system, custom features, and map switching

Beyond the stock strategy, additional capabilities can be added to the module, such as a true spark based hard cut rev limiter that does not also cut fuel, launch control and two step, and multiple switchable calibration maps that the driver can change between, along with other features the factory strategy does not provide. In general these require new code to be added and hooked into the existing control flow at carefully chosen points so that they cooperate with the stock logic and remain failsafe. The existence of these capabilities is noted here for completeness, and the stock mechanisms they build on, the combustion enable gate, the dwell path, the cruise switch inputs, are described in their own sections so the engineering context is clear. But the design of the custom operating system, its insertion points, the added code, how the multiple switchable maps are selected and stored, and everything that would let someone reproduce the map switching or the other custom features are **REDACTED**.

## Bidirectional controls

Bidirectional control, sometimes called active test or output test, is the ability for a tool to command the module to actuate an output directly, for example to cycle a solenoid, command a fan, or force a component on or off for diagnosis, rather than only reading data from it. In general these are implemented as diagnostic service routines that the module exposes, gated by the appropriate conditions and often by security access. The Black Oak bidirectional control services, their identifiers, and the actuation details are **REDACTED**.

# Part 22, Scope of the teardown

To give a sense of the completeness of the work behind this reference, the teardown identified and named every function in the module and organized them into a deep hierarchy of subsystem groupings, and it defined the full set of calibration scalars and tables across every subsystem. In round terms the module contains on the order of a few thousand distinct functions and well over ten thousand calibration items, organized into several hundred named subsystem groups spanning everything described in this document. Nothing in the image is left as unidentified code. The result is the complete functional picture presented here, from sensor inputs through the control mathematics to the hardware outputs.

The Ghidra spec in this repository is the front door to that work. It will not hand you the finished map, but it will put you in the same correct starting position the teardown began from, so the structure described in this reference is yours to rediscover and verify for yourself.

# Part 23, Full function map

This is a names only map of the control functions by subsystem, so the overall shape of the strategy is visible in one place. The startup, kernel, reflashing, and custom operating system functions are proprietary and are not listed, consistent with Part 21.

**Table and math primitives.** One dimensional table lookup, two dimensional bilinear lookup, floating point clamp, integer clamp, lag filter, float to scaled integer conversion, elapsed time, and diagnostic trouble code set and maturation.

**Analog and sensors.** Converter channel read, digital port read, external chip analog read, scaling group reader, throttle position reader, secondary group reader, charge temperature reader, fuel rail temperature reader, runner control position reader, fuel pump duty reader, per channel rationality filter, front and rear oxygen reads, oxygen rich and lean switch, battery filter, charge and intake temperature processing, and manifold pressure rationality.

**Air and load.** Air charge and load model, fuel air charge calculation, cam corrected volumetric efficiency lookup, speed density pressure model, throttle based load, airflow transfer, pressure validity debounce, pressure source select, airflow rationality gate, key on flow check, transient fuel filter, current charge calculation, charge prediction for both paths, normalization divisor, charge leak compensation, load versus wide open throttle, and the airflow monitor dispatcher with its filter and fault variants.

**Fuel.** Injector pulse width model, closed loop fuel manager, closed loop oxygen control and helper, oxygen limit manager and latch, adaptive learning, cranking mode select, decel fuel shut off entry and apply and transition and enable gate, sequential injector scheduler, bank mode scheduler, overspeed scheduler, injector disable, fuel pump and rail pressure control, and the ethanol and purge compensation paths.

**Spark and dwell.** Advance manager, best torque lookup and alternate, borderline lookup, borderline final, borderline interpolator, advance ramp and limiter, limit arbiter, output apply, base select, best torque and borderline minimum, entry rate limit, torque model limit calculation, torque model inversion, angle scheduler, end of dwell angle write, angle floor write, dwell update, coil and dwell setup, repetitive spark control, misfire mask latch, spark output enable, and the staged cylinder cut manager.

**Synchronization and hardware layer.** Crank capture, speed computation, speed update, vehicle speed update, spark and dwell event scheduler, synchronization state machine across its phases, combustion enable, phase detection, threshold check, speed gate, flag setter, tooth pattern snapshot, tooth gap detect, timing calibration select, status pack, hardware driver enable, injector channel write and clear and bank write and angle write, timing channel mode set and arm and capture read and status read, dwell write, end of dwell angle write, and injector write.

**Knock and cam.** Knock sense setup, detect setup, rate schedule, chip rate update, sensor monitor, test coordinator, and the cam timing advance control, schedule, control loop, duty calculation, output, duty report, and pulse width write.

**Throttle, idle, torque, limiters, transmission.** Throttle angle scheduler, torque demand loop, throttle position acquire and filter, position error monitor, throttle duty output, mode arbitration, drive disable, relay enable state machine, cold start enable gate, fault reporting, shared pulse width channel write, output driver init, idle target schedule, idle control loop, fan and air conditioning idle increase, alternator and charge control, rev limiter, speed limiter, park and neutral clip, torque monitoring, transmission range and over rev guard, converter clutch apply and release pressure, line and electronic pressure control, shift phase sequencing, gear ratio monitor, and shift torque management.

**Communication and diagnostics.** Frame transmit, mailbox operation, broadcast enqueue, engine data and tachometer builder, powertrain broadcast builder, transmission torque and temperature receive, wheel speed receive, gear and status and mode handlers, diagnostic check, service dispatch, parameter responder, parameter get and lookup and value and format and availability and descriptor, diagnostic response transmit, trouble code maturation manager, readiness and drive cycle tracking, misfire and catalyst and purge and heater monitors, and the validation routine, whose details are proprietary.

**Driver switches.** Switch input acquire, cruise switch decode, and the cruise state machine.

# Glossary

**Advance.** How far before top dead center, in crankshaft degrees, the spark fires.

**Air charge.** The mass of air trapped in a cylinder for a combustion event.

**Alpha N.** Estimating air from throttle angle and engine speed.

**Arbiter.** A function that selects a single output from several candidates, here by taking the minimum.

**Big endian.** A convention for the order in which the bytes of a multi byte number are stored in memory, most significant byte first.

**Bank.** One side of a V engine, with its own set of cylinders, oxygen sensor, and camshaft.

**Barometric pressure.** The ambient air pressure, which falls with altitude and is used to correct airflow and idle.

**Borderline.** The most spark advance that is safe before knock, as opposed to the advance that gives maximum torque.

**Calibration.** The data layer, the tables and scalars that shape behavior without changing the program.

**Closed loop.** Control that measures its result and corrects, as opposed to open loop which trusts a model.

**Crankshaft period.** The time between crankshaft tooth edges, the directly measured quantity the module uses in place of revolutions per minute for its most time critical work.

**Dwell.** The time a coil is charged before it fires.

**ECT.** Engine coolant temperature.

**EGO.** Exhaust gas oxygen, the oxygen sensor and the fuel control built on it.

**Hysteresis.** Using different thresholds for turning something on and off so it does not rapidly toggle.

**Interpolation.** Blending between the defined points of a table so the output moves smoothly as the input moves.

**Keep alive memory.** Battery backed memory that retains learned values across key cycles.

**Knock.** Uncontrolled secondary combustion that can damage the engine.

**Load.** Air charge relative to a reference, the main vertical axis of most tables.

**MBT.** The spark advance for maximum brake torque.

**Missing tooth.** The deliberate gap in the crankshaft tooth wheel that gives a once per revolution landmark.

**Open loop.** Control from a model without correction from a sensor.

**PID.** A proportional integral derivative control loop, a standard way to drive an output toward a target.

**PCM.** Powertrain control module, the computer described here.

**Pulse width.** The length of time an output is held on, for an injector the opening time, which sets fuel quantity.

**Readiness.** Whether an emissions monitor has had the conditions it needs to run and complete since codes were last cleared.

**Speed density.** Estimating air from manifold pressure, temperature, and volumetric efficiency rather than from an airflow sensor.

**Stoichiometric.** The chemically ideal air to fuel ratio for complete combustion.

**Strategy.** The program layer, the compiled control code, as opposed to the calibration.

**Transient fuel.** The extra or reduced fuel during rapid throttle changes to account for fuel films on the port walls.

**Volumetric efficiency.** How effectively a cylinder fills with air compared to its theoretical capacity.

# About and contact

This reference is a complete functional teardown of the Ford 5.4L 3V Black Oak 1472K CAN powertrain control module, published so that others working on these trucks have an accurate account of how the module actually works, from the sensors through the control logic to the outputs, rather than treating its tables as a black box. The Ghidra processor spec is included so you can open a read correctly and do that work yourself. Corrections and additions are welcome through the repository issue tracker.

**Trenton Hill, Trent's Tuning LLC**
[trentstuning.com](https://trentstuning.com/)
Performance Tuning that Delivers Proven Results.

Copyright 2026 Trent's Tuning LLC. All Rights Reserved. This document and the engineering work within it are the property of Trent's Tuning LLC and may not be copied, redistributed, or used commercially without written permission.
**Disclaimer.** This material is provided for off road, race, and research use only. Do not apply any of it to a vehicle operated on public roads. Modifying the emissions control or engine management of a street vehicle is a violation of federal law. Nothing here is a guarantee of fitness for any purpose, and the author accepts no responsibility for how it is used. Work on engine control systems carries real risk to equipment and to people, proceed only with the appropriate knowledge and facilities.
