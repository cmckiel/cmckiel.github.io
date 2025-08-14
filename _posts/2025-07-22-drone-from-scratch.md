---
layout: post
title: "A Drone From Scratch"
---

# A Drone From Scratch

Could I really start with nothing but the bare-metal? Could I write my own drivers? My own control algorithm? Would the drone really liftoff?

## A learning experience

This project excited me from the very beginning. And as of July 2025 the project is ongoing and will be for a while longer. I had been wanting to finish my Test Driven Development experiment creating a circular buffer. I had been so curious about implementing an interrupt based uart driver. I had always wanted to write a PID controller by hand. The flying drone is a great goalpost, but the real gold of this project is all the opportunity for learning along the way. And that is one thing to emphasize, this is a learning experience for me above all else.

If I were building a product, I would never build a Hardware Abstraction Layer by hand, I'd pull in ST's HAL. I wouldn't implement an interrupt based uart driver with custom ring buffers, I would use DMA. And for a project with multiple real-time requirements like control loops and command processing, I would most likely use an RTOS instead of designing a super loop architecture. But I have worked on an RTOS project before, and I wanted to try out a super loop with interrupts, and so that's the way I am going to do it.

## An exploration

This project isn't just about what I build, but also how. This is an exploration of modern embedded development seeking information on one question in particular, "How can we move fast like a modern software startup and maintain the safety and verification required for most hardtech?". How do we move fast and not break things?

In this project I use a Docker container for building software, which allows reproducible builds and environments, and let's new developers get building almost instantly. No more manually setting environment variables, or troubleshooting weird file path differences, or taking an entire week to get a new teammate building the project. Bring on people in less than an hour. Source control the environment.

I also build my software for both target hardware and desktop. Desktop building is possible by simulating the target hardware. This is critical for fast iteration. Access to target hardware is often limited. Devs have to share, request time, or sometimes the target hardware isn't physically available yet (still in production or being shipped). Desktop unit tests have helped me make short work of tasks like ensuring registers are accessed as described by the datasheet or verifying hardware independent logic. I have had difficulty using desktop testing for things like concurrency, timing issues, or hardware integration issues.

To cover the weaknesses of desktop testing, hardware-in-the-loop testing becomes a necessity. It is also exciting! Running code on the real hardware is awesome. I wanted a test bench available for myself during this project. It has allowed me to debug issues faster, set up experiments (like a 5 hour uart stress test), and integrate into my CI pipeline.

I wanted my project to have a CI pipeline. No one wants to manually build two targets, run two static analyzers, run all desktop unit tests, and deploy to hardware and run the HIL test suite every time they check in code. A CI pipeline does all this and more on every check-in. Wonderful. A modern dev environment wouldn't feel complete without it.

## The Initial Scope of Work

The End Goal:

> Upon receiving the START_MISSION command, the drone shall autonomously take-off. It shall rise to a height of 1 meter above the take-off point and hover for 5 seconds. Without further instruction, the drone shall complete 1 counter-clockwise turn followed by 1 clockwise turn. The drone shall stabilize within 2 seconds after which the drone shall land in a controlled manner within 10 cm of the take-off point.

The drone will fly up, spin, and land all by itself after you tell it to start. This will be a built in mission for now. And yes, this drone doesn't do much, yet. The reason I have phrased everything in that way above is because it sets a clear, testable goal and I will know exactly when I have accomplished it. I have estimated that this initial goal will take approximately 6 months to complete given my plan to build everything from scratch, to trust nothing, and to test everything.

## Github Links

- [Circular Buffer](https://github.com/cmckiel/circular_buffer) : A circular buffer created for embedded environments exclusively with TDD.
- [HAL](https://github.com/cmckiel/hal) : A Hardware Abstraction Layer containing all the hardware drivers and interfaces that the drone software will build upon.
