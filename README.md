<p align="center">
  <img src="logo.png" alt="Trent's Tuning LLC" width="300">
</p>

<h1 align="center">Black Oak 1472K CAN (TEJF4P5)</h1>
<p align="center"><b>Full Reverse Engineering Writeup of the Ford 5.4L 3V PCM</b></p>
<p align="center">Performance Tuning that Delivers Proven Results</p>
<p align="center"><a href="https://trentstuning.com/">trentstuning.com</a></p>

<p align="center"><b>Written by Trenton Hill, Trent's Tuning LLC</b><br>
Copyright 2026 Trent's Tuning LLC. All Rights Reserved.</p>


## Copyright and ownership (read this first)

This whole writeup and all the reverse engineering in it was done by me, Trenton Hill, and it is the property of Trent's Tuning LLC, and it is copyrighted and protected, 2026 Trent's Tuning LLC, all rights reserved. You can read it and learn from it and reference it for your own knowledge and research, but you can not copy it, repackage it, resell it, or pass it off as your own work, and you can not use it for a commercial product or service without written permission from Trent's Tuning LLC, yk just respect the work that went into this lmao it took forever.

And anything in here that says REDACTED is proprietary Trent's Tuning internal stuff and it stays that way, that is the bootloader, the kernel, anything about reading or writing or flashing the module, the custom OS and any custom features, bidirectional controls, and seed key security, none of that is published here on purpose.


## What this is

Alright so this is basically everything I figured out about the 2004 to 2006 Ford 5.4L 3 valve "Black Oak" PCM, the 1472K CAN version specifically, strategy code TEJF4P5, and I put it all in one place so anybody doing reverse engineering or dev work or tuning on these trucks can actually understand how the computer works from top to bottom instead of poking at tables blind, yk. I spent a stupid amount of time in Ghidra going function by function and table by table and following how it all ties together, and this is the result of all that.

The idea is pretty simple, you read this and you basically know the whole PCM, the logic, the systems, the subsystems, the sensors, the fuel, the spark, the trans, the CAN, all of it, so when you go to work on Black Oak you actually know what you are touching.

Couple heads up before we get into it. This is a race and off road and bench research writeup, none of this is for a street car and I am not responsible for what anybody does with it, use your head. And like I said some stuff says REDACTED on purpose and that stays in house, everything else, the actual engine control logic and the calibration, I am laying it all out.


## Table of contents

* [What Black Oak even is](#what-black-oak-even-is)
* [The chip and how the firmware is built](#the-chip-and-how-the-firmware-is-built)
* [How I actually reversed it, the xref problem](#how-i-actually-reversed-it-the-xref-problem)
* [Memory map](#memory-map)
* [Sensors and the ADC pipeline](#sensors-and-the-adc-pipeline)
* [Air charge and load, the backbone](#air-charge-and-load-the-backbone)
* [Fuel](#fuel)
* [Spark](#spark)
* [Crank and cam sync and RPM](#crank-and-cam-sync-and-rpm)
* [Knock](#knock)
* [Variable cam timing, Ti VCT](#variable-cam-timing-ti-vct)
* [Electronic throttle, driver demand, idle, torque, limiters](#electronic-throttle-driver-demand-idle-torque-limiters)
* [Transmission, 4R75E](#transmission-4r75e)
* [Cruise and steering wheel switches](#cruise-and-steering-wheel-switches)
* [CAN bus](#can-bus)
* [OBD II and the monitors](#obd-ii-and-the-monitors)
* [Accessories](#accessories)
* [Calibration, the tables and scalars](#calibration-the-tables-and-scalars)
* [The numbers, what got done](#the-numbers-what-got-done)
* [The REDACTED stuff](#the-redacted-stuff)
* [Full function map](#full-function-map)
* [About and contact](#about-and-contact)


## What Black Oak even is

Black Oak is the Ford name for the PCM strategy family on the 04 to 06 F150 and some other stuff with the 5.4 3 valve, and 1472K CAN is the specific version I worked on, strategy code TEJF4P5. It is the computer that runs the whole engine and the trans, so fuel, spark, cam timing, the electronic throttle, the torque model, the limiters, closed loop fuel, OBD, CAN broadcasting, all of it lives in here.

The thing to get right away is that it is really two pieces, there is the **strategy** which is the actual program and code, and then there is the **calibration** which is thousands of tables and scalars, and Ford writes the code once and just swaps the calibration per vehicle, yk so the same strategy runs a bunch of different trucks and only the numbers change. That is why when people say tune they mean the calibration side, but to really know what a table does you gotta understand the code that reads it, and that is the part nobody usually has, so that is what most of this writeup is.


## The chip and how the firmware is built

Black Oak runs on a **Freescale MPC5xx PowerPC**, it is 32 bit, big endian, and it has a hardware floating point unit, and that last part actually matters because basically every calibration value in here is an **IEEE 754 float**, not some scaled integer like alot of older ECUs, so the tables are real floating point numbers which is nice.

The chip does all the hard real time stuff in its on chip peripherals instead of in plain code, so you got:

* **TPU (Time Processor Unit)**, this handles the crank and cam tooth capture and the injector and coil output timing, it is basically its own little timing coprocessor and it runs microcode, so there is no normal crank ISR to read, you just read the results the TPU hands you.
* **QADC**, dual queued A to D, this scans all the analog sensors.
* **MIOS**, the on chip timer and PWM block, used for stuff like the cam oil control valves.
* **TouCAN**, the CAN controller.
* an external **smart I/O ASIC**, extra PWM outputs like the throttle and solenoids, the oxygen sensor A to D, and a dedicated **knock detection ASIC**.

So one thing that trips people up, the PCM has **two different PWM output systems**, the on chip MIOS and the external ASIC, and **two different analog front ends**, the QADC and the ASIC A to D for the O2s, and you gotta know which one you are looking at or nothing makes sense, yk.


## How I actually reversed it, the xref problem

Ok so this is the part I am actually proud of and it is the whole reason this writeup even exists, because without this you basically can not reverse engineer Black Oak at all.

**Problem 1, the banked flash.** A full read of the flash is a couple arrays stuck together in address order, so the second array's real runtime address is NOT where it sits in the file, and if you just load the bin raw into Ghidra every single reference the code makes into the calibration points off into nothing, and the disassembler builds almost no cross references, so it looks like the code barely even touches its own tables which is obviously wrong. So step one is rebuilding the memory map so every byte sits where the CPU actually sees it, plus feeding Ghidra the small data base register values the compiler assumed, and that alone brings back most of the analysis. Honestly if you skip this nothing else works.

**Problem 2, the tables hide on purpose.** Most of the tables are not reached by a pointer word you can just follow, the code **builds the address inline** right before it uses it, like it loads the top half and the bottom half of the address into a register with immediates and then goes and reads a little **descriptor**, and that descriptor is what points at the table's axes and its actual cell data. A 2D descriptor has an x count, a y count, a pointer to the X axis breakpoints, a pointer to the Y axis, and a pointer to the grid of cells, and a 1D one has a count and a pointer to interleaved breakpoint and value float pairs. And there is really just two lookup primitives doing all of it, a **1D interpolator** and a **2D bilinear interpolator**, and the 2D one is almost always called like lookup(rpm, load, table). But because the address is baked into the instruction as an immediate, stock Ghidra just sees a pile of constants and zero xrefs, so you get thousands of defined tables and almost no references and you are stuck.

So what I did, I wrote a **custom constant propagation scanner** that walks every instruction, rebuilds the control flow from the entry points and from every function pointer stored in flash, tracks each register as a known value as it flows through the code, records every address the code actually builds, and then follows the descriptor to axis to grid pointer chain to see where each map really gets read. That is what turned it from no xrefs into near complete coverage, and it recovered a TON of references (numbers are down in the [numbers section](#the-numbers-what-got-done)), and that is what made it possible to actually reverse engineer functions and know which table feeds which system, yk that is the key that unlocked everything else.


## Memory map

High level on purpose, since the exact flashing regions tie into stuff I am not publishing:

* **Low flash and bootloader region**, REDACTED
* **Main application code**, the strategy, all the engine and trans control functions
* **Upper flash and calibration band**, this is where the big block of tables and scalars I worked on lives, it is a whole band of calibration data separate from the classic table region
* **RAM**, all the live variables, sensor values, state machines, the keep alive stuff, extra
* **IO and peripheral space**, the TPU, QADC, MIOS, CAN, and the external ASIC all map in here

The processor is big endian and the image base and the block boundaries all got confirmed against the code itself so the map is accurate, and anything past that about read and write and the flashing side is REDACTED.


## Sensors and the ADC pipeline

### The pipeline

All the analog inputs come in through a scanned queue, and the flow is pretty consistent, the QADC result registers get read by a per slot accessor that averages a 3 sample triple, then a group reader scales the raw counts into real engineering units with a gain and offset, and then a generic **rationality filter** does the over and under range debounce, the slew limiting, and a lag filter before the final value and its fault flags land in RAM. There is also a digital port reader for QADC pins being used as on and off inputs. The O2 sensors are the odd one out, they do not go through the QADC, they come in through the external ASIC's own A to D.

One subtle thing, the QADC channel a reader asks for is a slot in the conversion queue, not a physical pin, the pin to slot mapping is in the command word table set up at init, so what the firmware actually uses, slot to scaled sensor, is fully known, the physical pin layer just sits on top of it.

### What it reads

* **Engine speed**, comes off the crank through the TPU, a capture ISR turns the tooth timing into a crank period, then an RPM compute function turns that into RPM and the current cylinder event index, and a speed update does stall and run detect and filtering and gives you the **control RPM** for limiters and fuel and the **map axis RPM** for indexing tables, and those are two separate things yk.
* **Vehicle speed**, from the driveshaft signal, both a float and a 0 to 255 byte version.
* **Throttle position**, dual sensors, TPS1 and TPS2, for the electronic throttle, filtered fast and slow and checked against min and max voltage.
* **Accelerator pedal (APP)**, its own redundant pair, totally separate from the throttle sensors, this is driver demand.
* **Mass airflow (MAF)**, a transfer function runs a volts to mass breakpoint table, that is the MAF cal.
* **Manifold pressure (MAP)**, on this truck MAP is normally **modeled** from the cam corrected VE model, not read off a sensor, there is a real MAP path but it is behind a config switch.
* **Coolant (ECT)** and **air charge and intake temp (ACT and IAT)**, the ACT one has a failsafe fallback if it goes bad.
* **Battery voltage**, filtered and then used all over, dwell, injector dead time, throttle, alternator, extra.
* **O2 sensors front and rear, both banks**, through the ASIC A to D, each one filtered and compared to a stoich switch voltage with hysteresis to get a rich or lean state per bank.
* **Fuel rail pressure and temp, IMRC position, and the returnless fuel pump duty feedback**, each on its own QADC reader.
* **Cruise and steering wheel switches**, an analog resistor ladder on one QADC slot, more on that later.


## Air charge and load, the backbone

This is probably the most important subsystem in the whole thing because everything downstream hangs off it, yk so pay attention to this one.

The PCM figures out cylinder air three different ways and picks or blends them by where you are operating, there is the **MAF** path, the main one when the MAF is good, a **speed density** path, the modeled MAP from the VE model, and a **throttle based alpha N** path, used when the MAF is invalid or at low throttle. And the VE core blends a few volumetric efficiency tables by the current **cam advance fraction**, so your cam timing literally reshapes the airflow estimate, that is a big deal on a 3V with Ti VCT.

There is a whole chain of little steps in the air charge calc, a MAP validity debounce, a MAP source select, a MAF rationality gate, a key on high flow check, the **transient fuel filter**, that is your accel and decel tip in tip out wall wetting comp, a current charge calc, and then quadratic charge **predictors** that anticipate the charge a little bit ahead, one for the SD path and one for the throttle path, and then an arbitration picks current or filtered or anticipated based on a couple cal switches.

And then the load model spits out the single most important number in the strategy, **engine load equals cylinder air charge divided by a standard air charge scaler**, and that load value is the Y axis of basically every fuel, spark, cam, and knock table, so it is used everywhere. It also does an air density correction off an air temp by coolant temp table, an air drift divisor, an air charge leak and offset comp, and a normalized load versus WOT term. So yeah if your airflow model is off after a cam swap or injectors or whatever, this is the area that has gotta get fixed or everything downstream is wrong.


## Fuel

### Delivery

Base fuel comes from the air charge and a target air fuel ratio, and then it gets turned into an injector pulse width by a **dual slope injector model**, so there is a low slope and a high slope with a breakpoint between them (ALOSL, AHISL, FUEL_BKPT), plus fuel pressure compensation and a minimum pulse width (MINPW), and then it gets scheduled out to the injectors through the timing module. There is two injector schedulers, a **sequential** one and a **bank and simultaneous** one that is used at start, and a dedicated **injector disable (fuel cut)** routine that both of them can divert into.

### Closed loop

There is a top level **closed loop fuel manager** that runs the trim per bank. The **short term** part is the classic Ford EGO jumpback and ramp integrator, so on every O2 switch it jumps the correction and then ramps it with a transport delay and a hold time percentage, and it adds a closed loop bias from rpm by load tables plus rear O2 and A/C bias terms, and the output is your per bank short term fuel multiplier, and a limit manager bounds it and handles the open loop defaults and the hold behavior, and when it is open loop it forces the trim back to stoich.

Then the **long term adaptive (KAM)** learner watches the short term trim, and once it has been stable long enough, a maturity counter, it learns the steady state error into per bank cell tables in keep alive RAM, indexed by airflow and load, clamped to min and max adapt limits (MAXADP, MINADP), with a faster gain while the engine is still warming up. And a config switch picks whether it references MAF mass or speed density. **This is the one to know about when tuning**, if you leave the adaptive learn switch on, KAM will slowly learn out your fueling changes over time, so you turn it off (FADPT_SW and KAM_NO_RESET type stuff) when you are dialing in fuel so it stops fighting you, yk.

### Cranking and cold start

A crank fuel mode selector picks sequential or simultaneous injection based on start temp, and then there is a family of crank tables for cranking lambda, a fuel multiplier versus time since start, the wall wetting lost fuel, injection delay, and cranking spark, extra. The MAF transfer table sits upstream of all of this so it affects fuel and load and spark all at once.

### Decel fuel shut off (DFCO)

An entry function sets a decel cut active flag when the throttle is closed and RPM and vehicle speed and n over v and load are all over their enable thresholds, and there is separate lower exit thresholds for hysteresis so it does not chatter. Then an apply and ramp function turns that into the actual fuel cut the injector scheduler obeys, with ramp in and ramp out counters, and a transition function smooths the tip in and tip out around it. Cals like DFSRPM_PC, DFLOD, DFSECT, DFS_PRV_N, and LAM_REACT live here.

### MAF monitor

There is a separate dispatcher that runs the MAF plausibility diagnostics, a few rate limited filter variants feed accumulators and if they go out of range they set the MAF circuit codes, but that is diagnostic only, it does not compute fuel, so do not confuse it with the MAF transfer.


## Spark

### Advance

A per cycle **spark advance manager** runs the chain. A **best torque (MBT) lookup** gives the MBT ceiling, that is the ideal timing if fuel and knock let you have it. Then a separate **borderline, knock limited, lookup** builds a per load borderline curve and adds intake geometry (charge motion and IMRC) and cam phase corrections, and a borderline final step interpolates it against load and RPM and adds a global octane modifier. A **ramp and limiter** slews the working spark toward the limit using the knock recovery rate tables and keeps track of which limit owns the timing right now.

And then the **arbiter** takes the **minimum** across every term, MBT, borderline, a torque model ceiling, knock retard, and an overspeed floor, and it records a little **source code** for which one won, and an output apply stage pushes the final number into the per cylinder advance array. So the big thing to get here, on a knock prone combo the borderline table (FN2200A Borderline Knock Table and friends) is what actually sets your timing under load, because the arbiter takes the min so borderline wins whenever it is lower than MBT, yk that is where you are really tuning timing.

The **torque based spark ceiling** is one of those arbiter terms and it is separate from the knock retard, two functions invert a parabolic torque versus spark model, coefficients by rpm and load, to find the spark delta off MBT that meets the torque target, so the torque model can pull timing too, not just knock.

### Angle scheduling and dwell

An **angle scheduler** turns the commanded advance into a per cylinder end of dwell, the spark angle, applying the firing order remap and the wasted spark and coil pairing, and setting the per cylinder event masks. Then a **dwell update** runs per spark event, it computes the coil charge time from a measured charge feedback times a harness multiplier with battery voltage comp, or a fixed dwell times a per cylinder error multiplier, clamps it between voltage derived min and max limits, and writes each coil channel and its angle to the TPU. It is coil on plug. There is also a coil setup that builds the dwell versus voltage curve and the multi strike config, and a repetitive spark controller for the multi strike count.

One nuance worth stating plainly, the per cylinder fire mask in the spark path is a **software shadow used for misfire bookkeeping**, it is not the signal the coil hardware fires on, the actual coil charging is governed by the dwell written to the TPU. How any of that gets leveraged for a custom spark strategy is REDACTED.

### The spark and fuel enable gate

There is a spark and dwell event scheduler that gates every event on a global spark disable flag plus a spark blank counter, and when it is disabled it writes zero dwell to all channels. But here is the thing, that same global flag is ALSO read by the fuel injector scheduler which then diverts to the injector disable routine, so the one factory global kill cuts **spark and fuel together**, it is not a spark only cut. That flag is owned entirely by the crank sync state machine so during normal running it just stays clear. Anything about building a cleaner custom cut off of this is REDACTED.


## Crank and cam sync and RPM

A **crank capture ISR** reads the TPU capture register for the current crank angle, the tooth to tooth period, and the sync state, and it correlates the cam (CID) capture for full crank and cam phase. The **RPM compute** derives RPM from the period and the cylinder count and gives the firing order cylinder index. Since the actual tooth decode is TPU microcode there is no big decompilable crank ISR, you are just reading the TPU result registers.

Little detail I think is slick, the PCM does not really keep RPM around for the time critical stuff, it keeps the **crank period**, the time between crank edges, and faster engine means shorter period, and the rev limit is even stored as a period not an RPM, so over the limiter really means the crank edge showed up too soon, and because those periods are positive floats the code can compare them as plain integers in the spots where it can not afford the float unit. Just a cool bit of how Ford did it.

A **crank sync state machine** owns that global spark and fuel enable flag, it goes reset then acquire then confirm then running, holding the kill flag while it is unsynced (cranking, stall, missing teeth) and clearing it once sync is confirmed, and there is helpers for the acquire sub phases, a tooth pattern snapshot, a missing tooth and gap detector, threshold and RPM gating, timing cal select, and a status word, plus a hardware driver enable that toggles the ignition driver power right along with the flag.

Under all of it is a low level **injector and TPU channel HAL**, little functions to write an injector pulse width, clear a channel, do a bank write, encode an injection angle, set a channel's mode bits, arm a channel, read a channel capture, read status, plus the dwell and angle writers, and naming that whole HAL is what makes every scheduler's raw register pokes actually readable, yk otherwise it is just a wall of register writes.


## Knock

The per cylinder knock detection and the retard integration are **done in the external knock ASIC**, not in the main code, the firmware just configures the chip and reads the results back, and that is why the per cylinder knock flags have barely any references in the code, cause the real work happens in silicon.

* a **sense setup** programs the per cylinder channel gains and the integrator read back scaling into the ASIC.
* a **detect setup** computes the per cylinder knock thresholds from rpm and load maps, the noise and knock crank angle windows, and the enable gate, load and coolant and RPM windows plus closed loop ready, and aggregates the per cylinder knock bits.
* a **rate scheduler** picks fast or normal recover and retard ramp rates off a transient counter, and a **rate update** pushes those into the ASIC and rearms the window each cylinder event.
* a **sensor monitor** compares the ASIC's knock magnitudes against noise window tables and sets the knock sensor circuit codes, that is sensor health, not the retard.
* a **test coordinator** keeps the knock test and the O2 self test from stepping on each other.

The accumulated retard then limits spark through the arbiter's knock term under a spark retard authority limit. So the thing to know, because the integration is in hardware the retard **algorithm** is fixed, you can calibrate the gains and windows and rates and authority, but you can not rewrite the knock strategy itself without touching the ASIC, it is not just a table.


## Variable cam timing, Ti VCT

Dual bank, each bank has its own cam sensor, PID loop, and oil control valve solenoid, and it is a 5 stage chain:

1. an **enable and ramp** stage builds a normalized cam advance authority, gated by no per cylinder knock and no faults and a temp window, or forced on by a switch, and detected knock pulls the cam advance back.
2. a **schedule** stage reads the rpm and load target tables, trims by oil temp, scales by the authority, and lag filters to the per bank target angles.
3. a **PID** stage drives the bank duty off the error between target and measured cam angle.
4. a **duty calc** stage turns that into an oil control valve duty, with an open loop override and limp defaults.
5. a **hardware output** stage writes both banks to the on chip MIOS PWM and latches the cam sensor faults.

There is also a reporting helper that mirrors the duty for the datastream. And since detected knock disables cam advance, this subsystem is actually a good reference for how the engine reacts to knock in general.


## Electronic throttle, driver demand, idle, torque, limiters

### Driver demand, pedal to torque to throttle

The pedal is read as a **torque request**, not a throttle position, so there is a linearization from pedal travel to an angle, then driver demand maps, pedal by RPM, make a desired engine torque and wheel torque, with low range variants, then a **torque demand PID** turns desired torque into a throttle area command, with traction gains in the traction state and clip limits, and feed forward and effective area models turn area into a throttle angle. An **angle scheduler** resolves the final commanded throttle angle out of all that and applies a **max throttle versus RPM authority cap**, which is basically the reduced power lever, and publishes the control state.

### Position servo

An acquire and filter makes the two filtered TP feedbacks, a **position error monitor** compares desired versus actual and sets throttle faults on steady state error, and a **duty output** turns the commanded angle into a PWM duty, with fixed limp duties in the fault modes, and calls the shared PWM channel driver that writes period and duty to one smart I/O ASIC channel, the throttle H bridge is one of those channels. Then there is mode arbitration, an H bridge drive disable that lets it release to the limp spring, a relay enable state machine, and a cold start enable gate, and faults get reported through a DTC maturation manager.

### Idle

On this platform **idle air IS the throttle**, there is no separate idle air valve, so a target function schedules the desired idle RPM from drive and neutral base values, with A/C on and cold start variants, a baro correction, and an in gear lag, and an ISC PID with separate opening and closing gains, picked by the error direction, outputs an idle throttle demand that gets summed with driver demand, bounded by idle min and max and the closed throttle cal. There is feed forward idle airflow tables and a decel to idle dashpot too, and a fan and A/C idle up that raises the target under accessory load.

### Limiters, important one

The factory rev limiter and speed limiter are **soft**, they are PI controllers that just command the electronic throttle closed, they do NOT cut spark or fuel, and they only work while the throttle is in normal control. There is also a park and neutral clip that caps torque and adds an RPM limit off load. Separately from the soft limiter, the rev limiter also expresses its limit RPM as a crank period that a hard overspeed test checks against, and that hard path is the one that will actually resort to a fuel cut and a spark retard floor at true overspeed. So just know the rev limiter you feel normally is a throttle closer and not a cut, and that matters alot if you want a real limiter, but how you would build a proper spark based hard cut off of any of this is REDACTED.


## Transmission, 4R75E

Automatic applications run the 4R75E, and there is 4 main cores:

* a **range and over rev guard** validates the allowed gear from selector position, current gear, and vehicle speed, and inhibits manual downshifts until speed drops so you can not grenade it.
* a **TCC apply pressure** scheduler builds the torque converter clutch pressure from per gear lockup rate arrays and slip, capped by a max.
* a **line and EPC pressure** controller computes main pressure from a torque term with per shift floors and turns it into the pressure control solenoid command, and that is your main **shift firmness** lever.
* a **shift torque management** function coordinates the spark and torque reduction with the trans during the actual shift, that is what makes a shift soft or firm from the engine side.

The actual up and down shift points are in the shift schedule tables, throttle by speed. There is a bunch of trans cals, gear ratio error thresholds, TCC slip and apply and release schedules, shift pressure and shift phase stuff, torque reduction targets, extra, and I have got all of it mapped, see the [cal section](#calibration-the-tables-and-scalars).


## Cruise and steering wheel switches

The steering wheel cruise buttons are an **analog resistor ladder** on a single QADC slot, so they are NOT on the CAN bus on this read, which is kinda handy to know. An acquire function classifies the ladder voltage into a button code with a 2 sample debounce, a decode function makes the button command and already keeps a **hold timer**, seconds since the button last changed, plus held past threshold flags, and a state machine runs the cruise logic, off and armed and engaged, and it gets shut off by brake, clutch, cranking, and over temp. And both the acquire and decode run every loop no matter what cruise is doing, so the button position and the hold time are always sitting there available.


## CAN bus

Two main primitives, a transmit frame function and a mailbox read and config function, plus a queued broadcast loader.

On the transmit side the PCM broadcasts engine data, including the frame the instrument cluster reads for the **tach**, engine RPM is in the first two bytes, with vehicle speed and pedal position, a gear and shift state frame, a status frame with the PRNDL bitfield and a bunch of flags, a manual or auto indicator, and a proprietary multi byte datastream packet. On the receive side it takes transmission torque and temp from the TCM, offset encoded, brake and clutch and neutral status, ABS wheel speeds, and mode flags, and it answers diagnostic requests on the OBD request IDs. The RPM and speed and pedal fields use a fixed power of two binary scaling, so the tach is literally RPM times a constant in two bytes.


## OBD II and the monitors

All the diagnostic data requests get answered out of one **sorted descriptor table** of a few hundred entries, and each descriptor has the PID or DID, an internal mode byte, current data or freeze frame or the enhanced data identifier space, a handler type, a computed callback type versus small raw or table codes, a pointer to the live source channel, and formatter args. The lookup is a binary search keyed on mode and identifier, and it dispatches either to a computed value path, reads the live source, does availability gating, formats through the OBD transfer function, or a raw path. The table has got all the standard mode 1 and mode 2 PIDs, RPM, MAP, MAF, timing, load, commanded AFR, fuel trims, O2s, cat temps, extra, and a big set of Ford enhanced DIDs.

The OBD **monitors** are calibration driven state machines that decide when the light comes on, they do not change how the engine runs:

* **Misfire**, inferred from crank speed, torque, deviation, with torque thresholds, an RPM window, a coolant window, and a failure action switch.
* **Catalyst efficiency and rear O2**, window gated by coolant, air temp, vehicle speed, and a minimum rear sensor temp.
* **Evap purge**, a multi phase flow test gated by idle and cruise enables, a load ceiling, and per phase time limits.
* **O2 heater**, current and duty cycle monitors with their own enables.


## Accessories

A **smart alternator and charge** controller gives a load responsive charging voltage and an idle RPM adder so electrical load does not sag the idle. **Cooling fan** control is threshold based off coolant and head temp with an electric fan idle up, and the over temp protection is a separate staged cylinder cut path, not a fan table. **A/C** adds an idle up comp in drive and neutral. **EGR** where it is even fitted is picked by a hardware type switch and is pretty minimal on this engine family.


## Calibration, the tables and scalars

Ok so this is the tuning side, and there is a LOT, like thousands of scalars and tables, and I mapped them by what function reads them so I know what each one actually does, not just the name. I am not gonna paste all 13k plus of them here obviously but here is the groups and the important ones per system, yk:

* **Adaptive learning (KAM)**, FADPT_SW (adaptive on and off), MAXADP and MINADP (max and min learned correction), AMP_MUL, FAST_ADPT_GN (warm up gain), VECT3 and VECT5 (min temps), KAM_NO_RESET, DECEL_CELL_SW, extra. This is the stuff that learns out your changes if you leave it on.
* **Closed loop fuel**, JUMP_FRAC and AMP_MULT (the EGO jumpback), HOLD_TM_PCT, LAM_MOD_MIN and LAM_MOD_MAX, DYN_LMIN and DYN_LMAX (short term trim clamps), MAX_ROA_L and MAX_ROA_R (rate of authority lean and rich), and the Peak to Peak amplitude table FN1352 with its filter PTPAMP_FK.
* **Fuel injector model**, FUEL_BKPT (breakpoint), AHISL (hi slope), ALOSL (low slope), MINPW (min pulse width), RF_DPREF_CAL (reference pressure drop), and the comp tables FNPW_BKCOMP and FNPW_HSCOMP and FNPW_LSCOMP and FNPW_OFFCOMP, breakpoint and slope and battery and pressure comp. These are what you change for bigger injectors.
* **Crank and cold start fuel**, cranking lambda, the startup fuel versus time tables (FN018xx families), wall wetting lost fuel, injection delay, extra.
* **DFCO**, DFSRPM_PC, DFLOD, DFSECT, DFS_PRV_N, LAM_REACT.
* **Spark**, the main spark tables with their rpm (Xnorm) and load (Ynorm) axes (FN016x and FN017x and FN012x families), failsafe cooling spark modifier FN701, lugging retard FN723C plus SPK_LUG_TIM, and the whole MBT set.
* **Borderline and knock spark**, FN2200A (the Borderline Knock Table, this is the big one), FN2270 (borderline adj for 100% ethanol), FN727 (BKT multiplier by % ethanol), SPK_FMM_ADJ (modifier for a failed knock sensor). This is where real timing under load tuning happens.
* **Knock sensor**, the per cylinder gains, the knock and noise window angles, the thresholds by rpm and load, the retard and recover rates, and the retard authority.
* **Air and load and speed density**, the MAF transfer table, the VE tables that blend by cam advance, the standard air charge scaler, the air density temp by temp table, the transient fuel (ACDC) stuff, the charge predictors.
* **Variable cam**, the rpm and load cam target tables per bank, the oil temp trim, the authority ramp, the PID gains, and the OCV duty limits.
* **Electronic throttle and driver demand and idle**, the pedal linearization, the driver demand torque maps (pedal by rpm), the torque demand PID gains, the max throttle versus rpm authority cap, the idle target schedules (drive and neutral and AC and cold), the ISC PID open and close gains, the dashpot.
* **Limiters**, the soft rev limit and speed limit PI gains and setpoints, the park and neutral clip, and the hard overspeed limit, stored as a period.
* **Trans**, shift schedule tables (throttle by speed), gear ratio error thresholds (GR1ERR_LVL through GR4ERR_LVL, GRV_ERR_TIM), TCC apply and release and lockup rate arrays, line and EPC pressure (shift firmness), shift pressure and phase tables, and the torque reduction targets.
* **Torque model**, the torque versus spark model coefficients, the torque limits, and the driver demand torque tables.
* **Other groups mapped**, alternator and charge, axle (ratio and tire), canister purge and evap, dashpot, ethanol and flex fuel, fuel pump and returnless, IMRC, MAF monitor, O2 front and rear, traction control, transient fuel, and the VID block (vehicle ID and part info).

Thats the gist of what is tunable and what each group does, yk.


## The numbers, what got done

Just to put some numbers on it since people always ask how deep this actually went:

* Functions, all named, 0 left as FUN_ , **2,510**
* Named symbols total, around **26,600**
* Namespaces, organized by subsystem, **693**
* **Cross references recovered, 117,852** (19,414 into the calibration and table band, and 50,100 into RAM)
* Calibration items defined, **13,577** (12,421 scalars and 1,156 tables)
* Instructions disassembled, around **179,000**

And the big one, a raw headerless bin loaded into stock Ghidra gives you basically **zero** references into the calibration band because the addresses are built in code, so those roughly 19,414 cal references and 50,100 RAM references did not exist until the custom scanner rebuilt every inline built map address, and that is the whole thing that made all of this possible, yk that is the part I am proud of lmao.


## The REDACTED stuff

So like I said up top, there is a chunk of this I am not publishing, it is Trent's Tuning internal and it stays that way. If you are wondering about any of the following, the answer here is just REDACTED:

* **Bootloader**, REDACTED
* **Kernel and OS internals**, REDACTED
* **Reading and writing and flashing the module**, how the bin comes off and goes back on, REDACTED
* **Seed and key and security access**, REDACTED
* **Checksum and how a modified bin is made valid**, REDACTED
* **The custom OS and any custom features built on it**, custom limiters, map switching, launch, any of that, REDACTED
* **Bidirectional controls and actuator commands**, REDACTED

Everything above that line in this readme is the stock engine control logic and the calibration, which is the part that is useful to actually understand the PCM, the proprietary dev and flashing side is not here on purpose.


## Full function map

Names only, grouped by subsystem, so you can see the shape of the whole thing (the bootloader and kernel and flashing and custom OS functions are REDACTED and not listed):

* **Table and math primitives**, 1D table lookup, 2D bilinear lookup, float clamp, int clamp, float lag filter, float to scaled int, elapsed time, DTC set and maturation.
* **ADC and sensors**, QADC channel read, QADC digital port read, ASIC A to D volts read, scale group reader, throttle TP reader, secondary group B reader, ACT reader, fuel rail temp reader, IMRC reader, fuel pump duty reader, per channel rationality filter, front O2 read, rear O2 read, O2 rich and lean switch, battery filter, ACT and IAT process, MAP rationality.
* **Air and load**, aircharge and load model, fuel aircharge calc, cam corrected VE lookup, speed density MAP model, alpha N throttle load, MAF transfer, MAP valid debounce, MAP source select, MAF rationality gate, MAF key on flow check, transient fuel (ACDC) filter, current aircharge calc, aircharge anticipate (SD and TP), air drift divisor, aircharge leak comp, load fraction versus WOT, MAF monitor dispatcher and filter and fault variants.
* **Fuel**, injector pulse width model, closed loop fuel manager, closed loop O2 (LAMBSE) control and helper, LAMBSE limit manager and latch, KAM adaptive learn, crank fuel mode select, DFCO entry and apply and transition and enable gate, sequential injector scheduler, bank mode scheduler, overspeed scheduler, injector disable (fuel cut).
* **Spark and dwell**, advance manager, MBT lookup and alt, borderline lookup, borderline final, borderline interp, advance ramp and limiter, limit arbiter, output apply, base select, MBT and borderline min, entry rate limit, torque model limit calc, torque model invert, angle scheduler, EOD angle write, angle floor write, dwell update, coil and dwell setup, repetitive spark control, misfire mask latch, spark output enable all, staged cylinder cut manager.
* **Sync and HAL**, crank capture ISR, RPM compute, RPM speed update, vehicle speed update, spark and dwell event scheduler, crank sync reset and acquire and confirm and disable and enter run and restart, spark and fuel enable (synced and running), sync phase a and phase b, threshold check, RPM gate, flag setter, pattern snapshot, tooth gap detect, timing cal select, status pack, hardware driver enable, injector channel write and clear and bank write and angle write, TPU channel mode set and arm and capture read and status read, dwell write, EOD angle write, injector write.
* **Knock and VCT**, knock sense setup, detect setup, retard rate schedule, ASIC rate update, sensor monitor, EGO test coordinator, VCT advance control, schedule, PID, duty calc, PWM output, duty report, MIOS PWM write.
* **Throttle and idle and torque and limiters and trans**, throttle angle scheduler, torque demand PID, TP acquire and filter, position error monitor, throttle duty output, mode arbitration, drive disable, relay enable state machine, cold start enable gate, fault DTC report, shared PWM channel write, output driver init, idle target schedule, idle ISC PID, fan and AC idle up, alternator and charge control, rev limiter PI, speed limiter PI, park and neutral clip, trans range and over rev guard, TCC apply pressure, line and EPC pressure, shift torque management.
* **CAN and OBD**, CAN transmit frame, CAN mailbox op, broadcast enqueue, engine data (tach) TX builder, powertrain broadcast builder, TCM torque RX, TCM temp RX, wheel speed RX, gear and status and mode handlers, OBD diag check, OBD service dispatch, mode 1 and 2 PID responder, PID get and lookup and value and format and available and current descriptor, diagnostic response TX, rear O2 catalyst monitor, checksum routine (details REDACTED).
* **Driver switches**, switch input acquire, speed control switch decode, speed control state machine.


## About and contact

This is my full teardown of the Black Oak 1472K CAN PCM (TEJF4P5), done the long way in Ghidra, and I put it out so other people doing reverse engineering or dev or tuning on these 3V trucks have an actual reference for how the computer works instead of poking at tables blind, yk. If something is wrong or you wanna add to it feel free to open an issue or hit me up.

**Trenton Hill, Trent's Tuning LLC**
[trentstuning.com](https://trentstuning.com/)
Performance Tuning that Delivers Proven Results.

Copyright 2026 Trent's Tuning LLC. All Rights Reserved. This document and the reverse engineering work in it are the property of Trent's Tuning LLC and may not be copied, redistributed, or used commercially without written permission.

**Disclaimer:** everything here is for off road, race, and research use only, do not use any of this on a vehicle operated on public roads, modifying emissions equipment or engine control on a street vehicle is against federal law, and I am not responsible for anything anybody does with this info, tune at your own risk and know what you are doing.
