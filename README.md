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

Several topics in this reference are marked **REDACTED**. For those topics the document still explains what the thing is and what it does in general engineering terms, but the Black Oak specific implementation details are proprietary to Trent's Tuning LLC and are deliberately withheld. This covers the bootloader, the operating system kernel, module reading and writing and reflashing, security access and the seed and key exchange, the firmware validation and checksum, the custom operating system and any custom features, and bidirectional controls.

## How to read this reference

This is written to take someone from new to the subject all the way to an engineering level understanding of the Black Oak 1472K CAN powertrain control module, which from here on is called the PCM. It describes the platform in general rather than any one vehicle or calibration file, so the functional reference itself carries no part specific identifiers. The only addresses anywhere in this repository are the handful of load offsets in the Ghidra section below, which you need in order to open a raw read at all. Terms that may be unfamiliar are explained the first time they appear, and there is a full glossary at the end. If you are brand new, read it top to bottom. If you already know engine control, you can jump to any subsystem, since each section stands on its own.

## Table of contents

* [Repository contents](#repository-contents)
* [Loading this module in Ghidra](#loading-this-module-in-ghidra)
* [Part 1, Foundations](#part-1-foundations)
* [Part 2, Sensing and inputs](#part-2-sensing-and-inputs)
* [Part 3, Air and fuel](#part-3-air-and-fuel)
* [Part 4, Spark and combustion timing](#part-4-spark-and-combustion-timing)
* [Part 5, Timing and synchronization](#part-5-timing-and-synchronization)
* [Part 6, Knock](#part-6-knock)
* [Part 7, Variable cam timing](#part-7-variable-cam-timing)
* [Part 8, Torque based control, throttle, idle, limiters](#part-8-torque-based-control-throttle-idle-limiters)
* [Part 9, Transmission, the 4R75E](#part-9-transmission-the-4r75e)
* [Part 10, Driver interface](#part-10-driver-interface)
* [Part 11, Communication and diagnostics](#part-11-communication-and-diagnostics)
* [Part 12, Accessories](#part-12-accessories)
* [Part 13, The calibration surface](#part-13-the-calibration-surface)
* [Part 14, Memory and regions](#part-14-memory-and-regions)
* [Part 15, Proprietary internals, explained and REDACTED](#part-15-proprietary-internals-explained-and-redacted)
* [Part 16, Scope of the teardown](#part-16-scope-of-the-teardown)
* [Part 17, Full function map](#part-17-full-function-map)
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

## How the PCM actually runs, for newcomers

Inside the running module there is a small operating system that schedules the control tasks. Some tasks run on a fixed time base, for example a background task every several milliseconds. The most important tasks run on an angular base, meaning they are triggered by crankshaft position rather than by the clock, because fuel and spark decisions must be made per cylinder event and the time available shrinks as engine speed rises. Interrupts from the TPU and the timers preempt ordinary work when a tooth arrives or an output must fire. The details of that scheduler are part of the operating system and are covered, at the level appropriate for a public document, in Part 15.

# Part 2, Sensing and inputs

## The analog pipeline

Almost every analog sensor on the engine reaches the strategy through the same pipeline, and learning that pipeline once explains every sensor at the same time. The QADC continuously scans its inputs into a set of result registers. A per channel accessor reads a result and averages a short burst of samples to reject electrical noise. A scaling step then converts the raw converter counts into real engineering units using a gain and an offset, so volts become degrees, or kilopascals, or grams per second. Finally a general purpose rationality filter applies range checking, debounce, slew rate limiting, and a lag filter, and only then does the finished value, together with any fault flags it raised, become the number the rest of the strategy uses.

A subtle but important point is that the channel a reader asks for is a slot in the conversion queue, not a physical pin. The mapping from queue slot to physical pin is set up once at initialization. What the strategy consumes, a given slot producing a given scaled sensor value, is fully defined, and the physical wiring simply sits underneath it.

The oxygen sensors are the one exception to this pipeline. They are not read through the QADC at all, they are read through the external ASIC's own analog to digital path, because their signal conditioning is handled in that chip.

## The sensor set

**Engine speed** begins at the crankshaft. The TPU captures the tooth timing and measures the interval between tooth edges. A speed computation converts that interval into revolutions per minute and also produces the current cylinder event index, which tells the rest of the strategy which cylinder is next in the firing order. A further processing stage performs stall and running detection and filtering, and notably it produces two distinct speed values, a control speed used by the limiters and the fuel system, and a separate map axis speed used to index the tables. Keeping those two separate avoids table lookups jittering on raw speed noise.

**Vehicle speed** is derived from the output shaft or driveshaft signal and is produced both as a precise floating point value and as a compact single byte value for broadcasting.

**Throttle position** on this platform uses two sensors on the electronic throttle body, and the **accelerator pedal** has its own separate redundant sensor pair. These are kept distinct because the pedal expresses what the driver is asking for, while the throttle sensors report where the throttle plate actually is, and the control system constantly compares the two.

**Mass airflow** is measured by a hot film sensor whose voltage is converted to a mass flow rate through a transfer table, which is one of the most important calibration tables in the whole module. **Manifold pressure** on this engine is normally not measured by a sensor at all, it is modeled from the airflow and the volumetric efficiency model, with a true sensor path available behind a configuration switch. **Coolant temperature** and **intake air or charge temperature** are thermistor inputs, and the charge temperature reader includes a failsafe fallback if the sensor reads implausibly. **Battery voltage** is filtered and then used in many places, including coil dwell, injector dead time compensation, throttle control, and alternator control.

The **oxygen sensors**, front and rear on both banks, are read through the ASIC, filtered, and compared against a stoichiometric switch voltage with hysteresis to produce a clean rich or lean indication per bank. Additional dedicated readers cover **fuel rail pressure and temperature**, **intake manifold runner control position**, and the **returnless fuel pump duty feedback**. The **steering wheel cruise switches** are read as an analog resistor ladder on a single converter slot, described in Part 10.

# Part 3, Air and fuel

## Air charge and load, the quantity everything depends on

If you only deeply understand one subsystem, make it this one, because nearly every other table in the module is indexed by the number it produces.

The PCM estimates how much air each cylinder is taking in, the cylinder air charge, using three methods and blending or selecting among them by operating condition. The first is the mass airflow method, which is primary whenever the airflow sensor is valid. The second is the speed density method, which infers air charge from the modeled manifold pressure and a volumetric efficiency model. Volumetric efficiency is the ratio of the air the cylinder actually ingests to the air it theoretically could at the current conditions, and the model blends several volumetric efficiency tables according to the current camshaft advance, so camshaft timing directly reshapes the air estimate. The third is a throttle based method, sometimes called alpha N, which estimates air from throttle angle and speed and is used when the airflow sensor is unavailable or at very low throttle openings.

Around these three methods sits a chain of supporting steps, a manifold pressure validity check, a source selection, an airflow rationality gate, a key on high flow check, a transient fuel filter that handles the extra fuel needed during throttle tip in and the removal of fuel during tip out as fuel films form and evaporate on the port walls, a current charge calculation, and predictive estimators that project the charge a fraction of an engine cycle ahead so that fuel and spark can be scheduled for where the engine will be, not where it was.

The output of all of this is engine load, defined as the cylinder air charge divided by a reference standard air charge. Load is the vertical axis of essentially every fuel, spark, cam, and knock table in the module. The subsystem also computes an air density correction, a normalization divisor, a small leak and offset compensation, and a measure of how close the engine is to wide open throttle. Because load is produced here and consumed everywhere, any error here propagates into fuel and spark across the entire map, which is why airflow and load are the first things to verify after any significant hardware change such as a camshaft or injector swap.

## Fuel delivery

Base fuel quantity comes directly from the air charge and a target air to fuel ratio. That fuel mass is converted into an injector opening time, the pulse width, by a dual slope injector model. Real injectors do not flow linearly across their whole operating range, so the model uses a low slope and a high slope joined at a breakpoint, adds compensation for fuel pressure since higher differential pressure pushes more fuel through the same opening, and enforces a minimum pulse width below which an injector cannot be trusted to open repeatably. The finished pulse width is then handed to the timing hardware to be delivered at the correct crank angle. The module contains two injection schedulers, a sequential scheduler that fires each injector individually in the firing order, and a bank or simultaneous scheduler used during starting, and both can divert into a shared injector disable routine that simply commands no fuel.

## Closed loop fuel trim

Open loop fueling relies on the models being perfect, which they never are, so the PCM measures the result at the oxygen sensor and corrects. A closed loop fuel manager runs this correction for each bank. The short term controller is the classic Ford exhaust gas oxygen integrator. Each time the oxygen sensor switches between rich and lean it jumps the correction by a step, then ramps it gradually, incorporating a transport delay that accounts for the time for exhaust to travel from the valve to the sensor, a hold time, and a bias drawn from speed and load tables along with contributions from the rear sensor and from air conditioning load. The result is a short term fuel multiplier per bank, bounded by a limit manager that also governs open loop defaults. When the system is in open loop, for example at wide open throttle or before the sensors are warm, the trim is forced back to the neutral stoichiometric value.

## Adaptive fuel learning

Short term trim corrects instantly but forgets instantly. The long term adaptive system, usually called keep alive memory or KAM, watches the short term trim and, once it has been stable long enough as judged by a maturity counter, learns the steady state error into cell tables held in battery backed memory, indexed by airflow or load. These learned corrections are clamped to minimum and maximum limits, they adapt faster while the engine is warming up, and they are applied to the injector pulse so that the short term trim can stay centered near zero. A configuration switch selects whether the learning is referenced to measured airflow or to the speed density model. The single most important practical fact about this system is that if adaptive learning is left enabled it will slowly compensate for deliberate fueling changes, effectively undoing them over time, so it is normally disabled while fuel is being developed.

## Cranking, cold start, and warm up

Starting an engine needs its own fuel logic. A mode selector chooses sequential or simultaneous injection based on the temperature at the moment of the start. A family of cranking tables sets the target ratio during cranking, a fuel multiplier that decays with time since the start, a wall wetting or lost fuel term that accounts for fuel that condenses on cold surfaces, an injection delay, and a dedicated cranking spark value. Because the airflow transfer table sits upstream of fuel, load, and spark all at once, it influences every one of these.

## Decel fuel shut off

When the driver lifts off at speed the engine does not need fuel to keep spinning, so the PCM can shut the injectors off entirely to save fuel and reduce emissions. An entry function arms a decel cut when the throttle is closed and engine speed, vehicle speed, the speed to gear ratio, and load are all above their thresholds, with separate lower thresholds for exiting so the function does not rapidly toggle. An apply stage turns that into the actual injector cut with controlled ramp in and ramp out, and a transition stage smooths the tip back in so the return of fuel is not felt as a bump.

## Airflow plausibility monitoring

Separate from the airflow transfer that computes fuel, a monitoring dispatcher runs the plausibility diagnostics for the airflow sensor. Several rate limited filters feed accumulators that set circuit fault codes when the signal is out of range. This path is diagnostic only and does not itself compute fuel, which is a common point of confusion.

# Part 4, Spark and combustion timing

## Computing spark advance

Spark timing is expressed as advance, the number of crankshaft degrees before top dead center at which the mixture is ignited. A per cycle spark advance manager coordinates the whole calculation. It begins with a best torque lookup, the timing that would produce maximum torque if nothing limited it, usually abbreviated MBT. In parallel it computes a borderline value, the most advance that is safe before the onset of knock, built from a knock limited table with added corrections for intake geometry such as the runner control state and for camshaft phase, then interpolated against load and speed and modified by a global octane term. A ramp and limiter slews the working value toward its target using knock recovery rate tables, so timing is restored gradually rather than snapped back after a correction.

The decisive step is the arbiter. It takes the minimum across every candidate, the best torque ceiling, the borderline limit, a torque model ceiling, the accumulated knock retard, and an overspeed floor, and it records a code identifying which term won. The result is written into the per cylinder advance array. Because the arbiter always takes the minimum, on any knock prone combination the borderline table is what actually governs timing under load, since it is usually the lowest of the candidates there. That is the single most important thing to understand about tuning spark on this platform.

The torque model ceiling deserves its own mention. The engine's torque is modeled as a function of spark, and two functions invert that relationship, using coefficient tables indexed by speed and load, to find the exact amount of spark retard from best torque that will deliver a requested torque target. This is how the torque based control system reaches into spark, and it is completely separate from the knock sensor retard.

## Scheduling the spark angle and charging the coils

Once a final advance is chosen it must become physical coil events. An angle scheduler converts the commanded advance into a per cylinder end of dwell angle, applies the firing order remap and the coil pairing arrangement, and sets the per cylinder event masks. A dwell update then runs for each spark event. Dwell is the length of time a coil is charged before it fires, and getting it right matters because too little dwell gives a weak spark while too much wastes energy and heats the coil. The dwell computation uses a measured charge feedback multiplied by a harness compensation term with battery voltage correction, or a fixed dwell with a per cylinder trim, clamps the result between voltage derived minimum and maximum limits, and writes each coil channel and its firing angle to the timing hardware. The ignition is coil on plug, one coil per cylinder. A coil setup function builds the dwell versus voltage relationship and the multi strike configuration, and a repetitive spark controller manages multi strike firing at low speeds.

One architectural detail is worth stating because it is easy to misread. The per cylinder fire mask that appears in this path is a software record used for misfire bookkeeping. It is not the signal the coil hardware fires on. The physical coil charging is governed by the dwell value written to the timing hardware.

## The spark and fuel enable gate

A single global flag enables or disables combustion. A spark and dwell event scheduler checks that flag, along with a spark blanking counter, and when disabled it writes zero dwell to every channel so no coil charges. That same flag is also read by the fuel injector scheduler, which diverts to the injector disable routine when it is set. The consequence is that the one factory global kill cuts spark and fuel together, it is not a spark only mechanism. That flag is owned by the crankshaft synchronization state machine described in Part 5, and during normal running it stays clear.

# Part 5, Timing and synchronization

## Crank and cam synchronization

For the engine to run, the PCM must know exactly where the crankshaft is and which stroke each cylinder is on, which requires both the crankshaft position and the camshaft position, since the crankshaft alone cannot distinguish the compression stroke from the exhaust stroke. A capture routine reads the timing hardware to obtain the current crankshaft angle, the tooth to tooth interval, and the synchronization state, and it correlates the camshaft identification signal to establish full engine phase. A speed computation then derives engine speed from the interval and the cylinder count and produces the firing order index. Because the actual tooth decoding is performed in timing hardware microcode, there is no ordinary decoding routine to inspect, the strategy reads finished results.

There is an elegant detail in how speed is handled for the most time critical decisions. Rather than carrying engine speed as a revolutions per minute value, the module often carries the crankshaft period directly, the time between tooth edges. A faster engine produces a shorter period. Even the rev limit threshold is stored this way, as a period rather than a speed, so that being over the limit simply means a tooth arrived sooner than the threshold interval. Because these periods are always positive, they can be compared with simple integer comparisons in the places where there is no time to spare for anything heavier.

A synchronization state machine owns the global combustion enable flag. It progresses through reset, acquisition, confirmation, and running states, holding combustion disabled while the engine is unsynchronized, during cranking, a stall, or a missing tooth event, and enabling it once synchronization is confirmed. Supporting functions handle the acquisition phases, a tooth pattern snapshot, a missing tooth and gap detector, threshold and speed gating, timing calibration selection, and a diagnostic status word. A hardware enable toggles the ignition driver power in step with the flag.

## The injector and timing channel layer

Underneath the schedulers is a low level hardware abstraction layer, a collection of small functions that write an injector pulse width, clear a channel, perform a bank write, encode an injection angle, set a channel's mode bits, arm a channel, read a channel's captured value, and read its status, along with the dwell and spark angle writers. These are the direct interface to the timing hardware, and naming them is what makes the higher level schedulers legible rather than a wall of raw register operations.

# Part 6, Knock

Knock, also called detonation, is the uncontrolled secondary ignition of the end gas in the cylinder, and sustained knock destroys engines, so detecting and suppressing it is a safety critical function. On this platform the per cylinder knock detection and the retard integration are performed in the dedicated external knock circuit rather than in the main strategy. The firmware configures that circuit and reads its results, which is why the per cylinder knock indications are referenced in relatively few places in the main code, the heavy signal processing happens in the chip.

The firmware side consists of a sense setup that programs the per cylinder gains and the readback scaling into the chip, a detect setup that computes the per cylinder knock thresholds from speed and load tables along with the noise and knock crank angle windows and the enabling conditions of load, coolant, and speed, a rate scheduler that chooses between fast and normal retard and recovery ramp rates based on a transient counter, a rate update that pushes those rates into the chip and rearms the detection window each cylinder event, a sensor monitor that compares the measured knock magnitudes against noise tables to set sensor circuit faults, and a coordinator that prevents the knock self test and the oxygen sensor self test from running at the same time. The accumulated retard then limits spark through the arbiter's knock term, bounded by a retard authority limit.

The practical takeaway is that because the detection and integration happen in hardware, the retard algorithm itself is fixed. Its gains, windows, rates, and authority are all adjustable through calibration, but the fundamental behavior cannot be rewritten by changing a table.

# Part 7, Variable cam timing

This engine uses twin independent variable camshaft timing, with each cylinder bank having its own camshaft position sensor, its own control loop, and its own oil control valve that hydraulically advances or retards that camshaft. Variable cam timing lets the engine optimize valve events across the operating range, improving low speed torque, high speed power, idle quality, and emissions, all from the same hardware.

The control is a five stage chain. An enable and ramp stage builds a normalized authority for how much cam advance is currently allowed, gated by the absence of knock, the absence of faults, and a temperature window, and reduced when knock is detected. A scheduling stage reads speed and load target tables, trims them for oil temperature since oil viscosity affects how quickly the mechanism can move, scales by the authority, and filters to smooth per bank target angles. A control stage, a proportional integral derivative loop, drives each bank's valve duty from the error between the target angle and the measured angle. A duty calculation stage converts that into the actual oil control valve command, with an open loop override and limp defaults for fault conditions. A hardware output stage writes both banks to the on chip pulse width modulation and latches any camshaft sensor faults. Because detected knock pulls cam advance back, this subsystem is also a useful window into how the whole engine responds to knock.

# Part 8, Torque based control, throttle, idle, limiters

## Driver demand as a torque request

A defining characteristic of this generation of control is that the accelerator pedal does not command throttle opening directly, it commands torque. The pedal position is linearized into an angle, then driver demand tables indexed by pedal and engine speed produce a desired engine torque and a desired wheel torque, with separate low range variants. A torque demand control loop converts the desired torque into a throttle area command, applying traction related gains and clipping limits, and feed forward and effective area models convert that area into a throttle angle. A final angle scheduler resolves the commanded throttle angle and applies a maximum throttle versus speed authority cap, which is the lever behind reduced power modes, and publishes the overall control state. Thinking of the whole system as a torque request flowing down to a throttle position, rather than a cable pulling a plate, is the key mental model.

## The throttle position servo

The electronic throttle must be held precisely at the commanded angle against a return spring and airflow forces, which requires a closed loop position servo. An acquire and filter stage produces two filtered throttle position feedbacks. A position error monitor compares commanded against actual and raises throttle faults on persistent error. A duty output stage converts the commanded angle into a motor pulse width, with fixed safe duties in fault modes, and drives one channel of the external ASIC, since the throttle motor is driven through an H bridge on that chip. Supporting this are mode arbitration, a drive disable that releases the plate to its limp home spring position, a relay enable state machine, a cold start enable gate, and fault reporting through a diagnostic maturation manager. Electronic throttle is safety critical, which is why it is wrapped in so much monitoring.

## Idle control

On this platform there is no separate idle air valve, idle air is metered by the same electronic throttle. A target function schedules the desired idle speed from base values for drive and neutral, with variants for air conditioning and cold start, a barometric correction, and a lag when shifting into gear. An idle control loop, with separate gains for opening and closing the throttle chosen by the direction of the error, outputs an idle airflow demand that is summed with the driver demand and bounded by idle limits and the closed throttle calibration. Feed forward idle airflow tables and a decel to idle dashpot support it, and a compensation raises the target under fan and air conditioning load so accessory engagement does not drag the idle down.

## The limiters

Understanding the limiters correctly prevents a great deal of confusion. The factory rev limiter and vehicle speed limiter are soft limiters, they are control loops that command the electronic throttle closed, and they do not cut spark or fuel. They are only active while the throttle is in normal control. A park and neutral clip caps torque and adds a speed limit off load. Separately, the rev limit is also expressed as a crankshaft period that a hard overspeed check compares against, and that hard path is the one that will resort to a fuel cut and a spark retard floor in a true overspeed event. So the limiter a driver normally feels is a throttle closer, not a cut.

## The torque model

Tying the throttle, spark, and transmission sections together is a central torque model. It estimates the engine's current torque and inverts the torque to spark relationship to deliver requested torque reductions quickly through spark retard, and more slowly through airflow. The transmission requests torque reduction during shifts, the traction system requests it during wheel slip, and the idle and limiter systems request it as well. This common torque currency is what lets all of these systems cooperate rather than fight.

# Part 9, Transmission, the 4R75E

Automatic applications pair this engine with the 4R75E, a four speed automatic, and the PCM contains its full controller. Four cores define its behavior. A range and over rev guard validates the allowed gear from the selector position, the current gear, and vehicle speed, and inhibits manual downshifts until speed drops into a safe range so the engine cannot be over sped by a careless downshift. A torque converter clutch apply scheduler builds the clutch apply pressure from per gear engagement rate arrays and the measured slip, capped at a maximum, which controls how firmly and quickly the converter locks. A line and electronic pressure control stage computes the main hydraulic pressure from a torque term with per shift floors and converts it into the pressure control solenoid command, and this is the primary lever for shift firmness, since higher pressure holds more clutch capacity at the cost of harshness. A shift torque management function coordinates engine torque reduction with the transmission during the actual shift, which is what makes a shift feel soft or firm from the engine side. The shift points themselves, the speeds at which the transmission changes gear for a given throttle, live in the shift schedule tables indexed by throttle and speed.

# Part 10, Driver interface

The steering wheel cruise control buttons are implemented as an analog resistor ladder read on a single converter slot, which means each button produces a distinct voltage rather than a digital signal, and on this variant they are not broadcast on the communication bus. An acquire function classifies the ladder voltage into a button code with a short debounce, and a decode function produces the button command while also maintaining a hold timer measuring how long a button has been held and flags for held past a threshold. A state machine runs the cruise logic across its off, armed, and engaged states and is disabled by the brake, the clutch, cranking, and over temperature conditions. Both the acquire and decode functions run every loop regardless of whether cruise is active, so the current button state and hold time are always available to the rest of the strategy.

# Part 11, Communication and diagnostics

## The communication bus

The PCM talks to the rest of the vehicle over a Controller Area Network bus. Two primitives underlie all of it, a frame transmit function and a mailbox read and configure function, together with a queued broadcast loader. On the transmit side the PCM broadcasts engine data, including the frame the instrument cluster reads to drive the tachometer, which carries engine speed in its first bytes along with vehicle speed and pedal position, a gear and shift state frame, a status frame carrying the transmission range selector field and various flags, a manual or automatic indicator, and a proprietary multi byte data packet. On the receive side it accepts transmission torque and temperature from the transmission controller in an offset encoded form, brake, clutch, and neutral status, antilock brake wheel speeds, and mode flags, and it answers diagnostic requests. The speed, vehicle speed, and pedal fields use a simple binary scaling, so for example the tachometer value is engine speed multiplied by a fixed constant packed into two bytes.

## Diagnostics and monitors

All diagnostic data requests are answered from a single sorted descriptor table of several hundred entries. Each entry carries the parameter or data identifier, a mode byte distinguishing current data from freeze frame data from the enhanced manufacturer specific space, a handler type, a pointer to the live source of the value, and formatting arguments. A request is resolved by a binary search on the mode and identifier, dispatching either to a computed path that reads the live source, applies availability gating, and formats the result, or to a simpler raw path. The table includes the standard legislated parameters such as speed, pressures, airflow, timing, load, commanded ratio, fuel trims, oxygen sensors, and catalyst temperatures, as well as a large set of manufacturer enhanced identifiers.

Separately, the emissions monitors are calibration driven state machines that decide when to illuminate the malfunction indicator lamp, they do not change how the engine runs. The misfire monitor infers misfires from crankshaft speed deviation, since a cylinder that does not fire fails to accelerate the crankshaft as expected, gated by torque thresholds and speed and coolant windows. The catalyst efficiency and rear oxygen monitors are gated by coolant, air temperature, vehicle speed, and a minimum rear sensor temperature. The evaporative purge monitor runs a multi phase flow test gated by operating conditions and time limits. The oxygen sensor heater monitors watch heater current and duty.

# Part 12, Accessories

A smart alternator and charging controller provides a load responsive charging voltage and raises the idle target so electrical load does not drag the idle down. Cooling fan control is threshold based on coolant and cylinder head temperature with an idle increase when the electric fan engages, and notably the over temperature engine protection is a separate staged cylinder cut path rather than a fan table. Air conditioning engagement adds an idle compensation in drive and neutral. Exhaust gas recirculation, where fitted, is selected by a hardware type switch and is minimal on this engine family.

# Part 13, The calibration surface

The calibration is the tuning surface, the set of thousands of scalars and single and two dimensional tables that shape the engine's behavior. Scalars are single values, a threshold or a gain. Tables are grids of values, most commonly indexed by engine speed on one axis and load on the other, with the stored values being real floating point numbers. A two dimensional table is read by interpolating between the four surrounding cells, so the engine gets a smooth value between the defined breakpoints rather than a stair step. The axis breakpoints themselves are stored alongside the table so that the spacing of the grid can be uneven, finer where the engine is sensitive and coarser elsewhere.

What follows is an overview of the major calibration groups and the kind of parameters each contains, named by the generic parameter identifiers that describe their function.

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

# Part 14, Memory and regions

At a conceptual level the module's memory divides into a region of flash that holds the startup and low level code, a region that holds the main application strategy, a region that holds the large body of calibration tables and scalars, a region of random access memory that holds all the live values, sensor readings, state machines, and the battery backed learned data, and the input output space where the timing hardware, the converters, the pulse width modulation blocks, the communication controller, and the external chip all appear to the processor. The exact boundaries, and anything concerning the startup and low level region in detail, belong to the proprietary material in Part 15 and are not published here. The Ghidra spec in this repository sets up the regions you need to load and analyze a read, and nothing beyond that.

# Part 15, Proprietary internals, explained and REDACTED

This section exists so that the reference is complete in describing what the module contains, while keeping the sensitive implementation details proprietary to Trent's Tuning LLC. For each item below, the general concept is explained, and the Black Oak specific implementation is marked REDACTED.

## The bootloader

A bootloader is the first software that runs when the module powers up, before the main engine strategy takes control. Its general job is to initialize the processor, decide whether the main application is valid and should be allowed to run, and provide the controlled entry point that a factory programming tool uses to erase and rewrite the module. It is the gatekeeper between a module that runs its normal code and a module that has dropped into a programming mode. The Black Oak bootloader structure, its entry points, its validation behavior, and the way it hands control to the application are **REDACTED**.

## The operating system kernel

The kernel is the small real time operating system that schedules the control tasks, services the hardware interrupts, manages timing, and arbitrates access to shared resources so that the many subsystems described above can run together without interfering. In general terms it is what turns a pile of individual control functions into a coordinated system that meets its timing deadlines every engine cycle. The Black Oak kernel internals, its scheduler, and its service interfaces are **REDACTED**.

## Reading and writing the module, reflashing

Reprogramming a module means erasing its flash memory and writing new contents, which is how both a factory update and a custom calibration are installed. In general this is done over the diagnostic connection using a defined sequence of steps that places the module into a programming state, transfers the new data, and verifies it, and it is constrained by how the flash hardware itself can be erased and written. The specific paths by which Black Oak is read and written, the sequences involved, and the practical methods are **REDACTED**.

## Security access, the seed and key

Security access is the lock that prevents unauthorized reprogramming and unauthorized access to protected functions. The general mechanism is a challenge and response. The module issues a random value, the seed, the tool must transform that seed through a secret algorithm into the correct answer, the key, and only a tool that knows the algorithm can produce a key the module accepts. This is what stops arbitrary tools from rewriting the module. The Black Oak seed and key algorithm and the security access details are **REDACTED**.

## Firmware validation and checksum

A checksum is a value computed across a region of the firmware that lets the module verify its own contents have not been corrupted or tampered with. In general the module computes these values at startup or in the background and compares them against stored expected values, and if they do not match it can refuse to run normally and set a fault. For any modified calibration to be accepted, these values must be made consistent with the new contents. The Black Oak checksum scheme, the regions it covers, the algorithms, and how a modified image is made valid are **REDACTED**.

## The custom operating system and custom features

Beyond the stock strategy, additional capabilities can be added to the module, such as a true spark based hard cut rev limiter that does not also cut fuel, launch control and two step, multiple switchable calibration maps, and other features the factory strategy does not provide. In general these require new code to be added and hooked into the existing control flow at carefully chosen points so that they cooperate with the stock logic and remain failsafe. The existence of these capabilities is noted here for completeness. Their design, their insertion points, the added code, and everything about the custom operating system that hosts them are **REDACTED**.

## Bidirectional controls

Bidirectional control, sometimes called active test or output test, is the ability for a tool to command the module to actuate an output directly, for example to cycle a solenoid, command a fan, or force a component on or off for diagnosis, rather than only reading data from it. In general these are implemented as diagnostic service routines that the module exposes, gated by the appropriate conditions and often by security access. The Black Oak bidirectional control services, their identifiers, and the actuation details are **REDACTED**.

# Part 16, Scope of the teardown

To give a sense of the completeness of the work behind this reference, the teardown identified and named every function in the module and organized them into a deep hierarchy of subsystem groupings, and it defined the full set of calibration scalars and tables across every subsystem. In round terms the module contains on the order of a few thousand distinct functions and well over ten thousand calibration items, organized into several hundred named subsystem groups spanning everything described in this document. Nothing in the image is left as unidentified code. The result is the complete functional picture presented here, from sensor inputs through the control mathematics to the hardware outputs.

The Ghidra spec in this repository is the front door to that work. It will not hand you the finished map, but it will put you in the same correct starting position the teardown began from, so the structure described in this reference is yours to rediscover and verify for yourself.

# Part 17, Full function map

This is a names only map of the control functions by subsystem, so the overall shape of the strategy is visible in one place. The startup, kernel, reflashing, and custom operating system functions are proprietary and are not listed, consistent with Part 15.

**Table and math primitives.** One dimensional table lookup, two dimensional bilinear lookup, floating point clamp, integer clamp, lag filter, float to scaled integer conversion, elapsed time, and diagnostic trouble code set and maturation.

**Analog and sensors.** Converter channel read, digital port read, external chip analog read, scaling group reader, throttle position reader, secondary group reader, charge temperature reader, fuel rail temperature reader, runner control position reader, fuel pump duty reader, per channel rationality filter, front and rear oxygen reads, oxygen rich and lean switch, battery filter, charge and intake temperature processing, and manifold pressure rationality.

**Air and load.** Air charge and load model, fuel air charge calculation, cam corrected volumetric efficiency lookup, speed density pressure model, throttle based load, airflow transfer, pressure validity debounce, pressure source select, airflow rationality gate, key on flow check, transient fuel filter, current charge calculation, charge prediction for both paths, normalization divisor, charge leak compensation, load versus wide open throttle, and the airflow monitor dispatcher with its filter and fault variants.

**Fuel.** Injector pulse width model, closed loop fuel manager, closed loop oxygen control and helper, oxygen limit manager and latch, adaptive learning, cranking mode select, decel fuel shut off entry and apply and transition and enable gate, sequential injector scheduler, bank mode scheduler, overspeed scheduler, and injector disable.

**Spark and dwell.** Advance manager, best torque lookup and alternate, borderline lookup, borderline final, borderline interpolator, advance ramp and limiter, limit arbiter, output apply, base select, best torque and borderline minimum, entry rate limit, torque model limit calculation, torque model inversion, angle scheduler, end of dwell angle write, angle floor write, dwell update, coil and dwell setup, repetitive spark control, misfire mask latch, spark output enable, and the staged cylinder cut manager.

**Synchronization and hardware layer.** Crank capture, speed computation, speed update, vehicle speed update, spark and dwell event scheduler, synchronization state machine across its phases, combustion enable, phase detection, threshold check, speed gate, flag setter, tooth pattern snapshot, tooth gap detect, timing calibration select, status pack, hardware driver enable, injector channel write and clear and bank write and angle write, timing channel mode set and arm and capture read and status read, dwell write, end of dwell angle write, and injector write.

**Knock and cam.** Knock sense setup, detect setup, rate schedule, chip rate update, sensor monitor, test coordinator, and the cam timing advance control, schedule, control loop, duty calculation, output, duty report, and pulse width write.

**Throttle, idle, torque, limiters, transmission.** Throttle angle scheduler, torque demand loop, throttle position acquire and filter, position error monitor, throttle duty output, mode arbitration, drive disable, relay enable state machine, cold start enable gate, fault reporting, shared pulse width channel write, output driver init, idle target schedule, idle control loop, fan and air conditioning idle increase, alternator and charge control, rev limiter, speed limiter, park and neutral clip, transmission range and over rev guard, converter clutch apply pressure, line and electronic pressure control, and shift torque management.

**Communication and diagnostics.** Frame transmit, mailbox operation, broadcast enqueue, engine data and tachometer builder, powertrain broadcast builder, transmission torque and temperature receive, wheel speed receive, gear and status and mode handlers, diagnostic check, service dispatch, parameter responder, parameter get and lookup and value and format and availability and descriptor, diagnostic response transmit, catalyst monitor, and the validation routine, whose details are proprietary.

**Driver switches.** Switch input acquire, cruise switch decode, and the cruise state machine.

# Glossary

**Advance.** How far before top dead center, in crankshaft degrees, the spark fires.

**Air charge.** The mass of air trapped in a cylinder for a combustion event.

**Alpha N.** Estimating air from throttle angle and engine speed.

**Arbiter.** A function that selects a single output from several candidates, here by taking the minimum.

**Big endian.** A convention for the order in which the bytes of a multi byte number are stored in memory.

**Bank.** One side of a V engine, with its own set of cylinders, oxygen sensor, and camshaft.

**Borderline.** The most spark advance that is safe before knock, as opposed to the advance that gives maximum torque.

**Calibration.** The data layer, the tables and scalars that shape behavior without changing the program.

**Closed loop.** Control that measures its result and corrects, as opposed to open loop which trusts a model.

**Dwell.** The time a coil is charged before it fires.

**ECT.** Engine coolant temperature.

**EGO.** Exhaust gas oxygen, the oxygen sensor and the fuel control built on it.

**Hysteresis.** Using different thresholds for turning something on and off so it does not rapidly toggle.

**Keep alive memory.** Battery backed memory that retains learned values across key cycles.

**Knock.** Uncontrolled secondary combustion that can damage the engine.

**Load.** Air charge relative to a reference, the main vertical axis of most tables.

**MBT.** The spark advance for maximum brake torque.

**Open loop.** Control from a model without correction from a sensor.

**PID.** A proportional integral derivative control loop, a standard way to drive an output toward a target.

**PCM.** Powertrain control module, the computer described here.

**Pulse width.** The length of time an output is held on, for an injector the opening time, which sets fuel quantity.

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
