---
layout: post
title: "Neutros Drone Platform: First Uncontrolled Flight"
categories: [Neutros]
tags: [drone]
---
# Neutros Drone Platform: First Uncontrolled Flight

![Neutros Drone v0 Prototype](/assets/neutros_drone_v0.jpg)
> Neutros Drone v0 Prototype (emphasis on prototype)

In this post I describe lessons learned from the first flight test of Neutros, a custom flight controller built from the registers up.

## An Eager Flight Test

What is the bare minimum capability a flight software must have to get off the ground? This wasn't a theoretical question for me — it was a target. And before answering it, there's a question that comes first: why risk the prototype on a flight test where the expected outcome is a crash? What did I hope to gain?

The answer: feedback.

I believe complex software systems that work grow from simple systems that work. This means delivering to a sequence of increasingly sophisticated demos. Demos that work. Demos of coherent and measurable capability, layered on iteratively. This is how I approach building Neutros.

My demo sequence so far:

1. C2 loopback between flight controller (FC) and ground control station (GCS)
2. Live IMU telemetry streaming from FC to GCS
3. Spin one motor
4. Spin four motors
5. Uncontrolled Flight Test (no control laws, just raw inputs)

Each delivered something observable. And each brought me closer to my envisioned system. But most importantly, each demo gave me feedback. Does it work or not work? In what ways does it not work? What unknown-unknowns have presented themselves? This feedback is invaluable and instructive.

## My Very Instructive Feedback

It crashed instantly. Not even in the way I thought it would. And that's the point.

I was completely prepared to lose some parts in a crash if I got to see it fly 50 ft in the air. 100% worth the trade after the amount of work and time I put into this. But real world data amounted to no more than 6 inches.

![Neutros Flight Test One](/assets/uncontrolled_flight.gif)
> Drone immediately flies sideways quickly. I originally thought it would fly upwards.

Neutros immediately flew sideways, sliding across the ground until it collided with the cabinet. I had figured I should have been doing a tethered test. That would have prevented the damage. But I didn't want to miss out on the opportunity for it to get some real air. As you can see, that didn't happen anyway.

A test like this demonstrates the importance of control laws viscerally in a way reading about it doesn't quite capture.

## Tethered Test Rig

Lesson received. I went back, rigged up a tether out of rope and a few dumbbells, and tried again. I was so sure I knew how to create a tethered test rig for this drone. Once again, feedback was instructive.

![Tethered Flight Test](/assets/uncontrolled_flight_tethered.gif)
> Drone thrashing against anchors and crashing.

This failure shows the inaccuracy of my mental models the clearest. I imagined the drone flying directly upwards, being pulled on by the ropes until everything became taut and stable flight was accomplished. I never imagined the oscillations that would lead to the loss of more propellers.

Shortly after lifting off the ground, Neutros began thrashing between the different anchors. As the ropes oscillated between full tension and slack, eventually one of the propellers got tangled, and the drone was grounded. Future test rigs will need to be engineered much more rigorously to prevent this kind of behavior.

But beyond test rigs, another lesson presented itself. At some point during the thrashing, the C2 cable was dislodged, preventing further commands from reaching the flight controller. With no timeout in place, the drone was stuck at wide open throttle. It had no knowledge that it had crashed nor did it know it was out of communications with the ground station. The only way to stop it was to unplug the battery while several props spun quite quickly near the plug.

Of course, my fingers got bit.

This was a real safety issue I had underestimated. And it pointed at something larger than a missing feature, which I'll come back to.

## The Aftermath

![Broken Props on Desk](/assets/the_aftermath.jpg)
> Broken props laid out on desk. All pieces were not accounted for.

7/8 propellers destroyed. Missing pieces. Millions in damages.

But in all seriousness, I am glad I did it. The point of this project for me is to learn, and sometimes the hard way is the best way. This drone gives me an opportunity to break things in a way that isn't possible at work. This drone is small and cheap.

But what happens when your UAV is 500 lbs and flies over people's homes? What happens when it is the size of a real plane?

You can't take the same chances. C2 timeouts aren't something you can "get to later." Failed tests can be expensive and damage project reputation. And above all, human safety must be the highest priority. A flight test like mine demonstrates that more, not less.

## Lessons Learned

There wasn't a real engineering reason to test it without the rig the first time. That was a choice I made in the hopes of seeing something cool. The rig I did make wasn't reasoned through well at all and won't be sufficient for progressing the project further. Three things stand out, and they're related.

**Mental models of dynamics are unreliable until tested.** I imagined upward flight; I got sideways. I imagined a rope-tensioned hover; I got an oscillating thrash. The simulation in my head is not the simulation that matters. This is exactly the kind of confidence that gets bypassed by reality the moment hardware is involved.

**Test rigs are systems and need engineering, not intuition.** A poorly engineered rig doesn't just fail to catch issues — it introduces new failure modes. The dumbbell-and-rope tether did both: it didn't protect the drone from itself, and it created an oscillation regime that wouldn't have existed otherwise. I wouldn't build a flight controller from intuition. Test rigs deserve the same rigor.

**"Eager" is not the same as "fast."** Skipping the rig didn't accelerate the project. It cost a set of props, a tethered test that also failed, and put my fingers near spinning propellers because there was no timeout to fall back on. The hardest discipline isn't writing the tests. It's not bypassing them on flight day. That is the tension I am exploring. How do you move fast and not break things? This is a field that requires slow and careful thinking. Once software like this is fielded, it must work.

For completeness, here's the inventory.

What went wrong:

1. Drone did not fly upwards, but consistently flew sideways
2. Motor mount screws loosened and came out
3. Drone lost comms at full throttle with no timeout
4. Drone crashed and continued at full throttle
5. Test stand did not prevent damage to drone

What went right:

1. Drone left ground for first time under its own power
2. Drone accepted motor speed commands correctly
3. Completely powered from battery, no external power cable
