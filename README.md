# FPGA Internship Documentation

This repository contains setup guides, project walkthroughs, and notes from my FPGA internship using **Red Pitaya STEMlab 125-14**, **Vivado**, and **Verilog.**

## Repository Structure

- **Introduction**
    1. [Intro to Red Pitaya](/introduction/red_pitaya.md)
    2. [Setup Guide](/introduction/setup_guide.md)
    3. [Vivado Overview: Mapping Red Pitaya Hardware into Vivado](/introduction/vivado_overview.md)

- **Projects**
    1. [LED Blink](/projects/led_blink.md)
    2. Wire
    3. Oscilloscope
    4. Signal Generator
    5. Counter
    6. Counter + PID

- **Theory**
    1. Verilog
    2. PID

## Notes and Advice:

1. This can be quite overwhelming if you don't have a background in it. There is a lot of new information to take in. Don’t try to memorise everything at once — you’ll pick things up naturally as you do the projects. 

2. Use ChatGPT carefully: it can explain concepts well, but don’t rely on it for Verilog debugging or writing full code. (At least in my experience with version 5.) 

3. If you just want to program the Red Pitaya to do one of the projects, read through the [Setup guide](/introduction/setup_guide.md) and then go to your desired project.

4. If you’re looking at this repository from the QEG’s GitLab, it is worth looking at the [original repository](https://github.com/sneha-balamurali/fpga/tree/main) for any updates.

## Resources:

### [Red Pitaya Documentation:](https://redpitaya-knowledge-base.readthedocs.io/en/latest/learn_fpga/fpga_learn.html)

Use this as a guide. The tutorials there (e.g. the LED blink lesson) say to install Vivado 2020.1, because the original tutorial code was written and tested on Vivado 2020.1. You can install the latest version instead — most things still work if you adapt slightly.

### [Pavel Demin github:](https://pavel-demin.github.io/red-pitaya-notes/)

### [Master's Thesis: Real-Time Processing of Laser-Ultrasonic Signals](https://epub.jku.at/obvulihs/download/pdf/2406394?originalFilename=true )
Author: Thomas Paireder, Johannes Kepler University Linz, 2017

A lot of useful information on implementing real-time signal processing with the Red Pitaya. Many of the methods and design considerations are broadly applicable.

### [Benjamin Millward github:](https://github.com/Bentwin2002/Group_IV_RP/tree/main/Complete_setup/tmp/wip_9/wip_9.srcs/sources_1/new)

Has useful code that is worth looking at later on when you are building your own projects. Includes code for things like a PID.

## Notes

**Author:** Sneha Balamurali

**Disclaimer:** This documentation is based on my own learning. While I’ve done my best to keep it accurate, there may be mistakes — please double-check against official references where needed.