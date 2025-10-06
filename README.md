# FPGA Internship Documentation

This repository contains setup guides, project walkthroughs, and notes from my FPGA internship using **Red Pitaya STEMlab 125-14**, **Vivado**, and **Verilog.**

## Repository Structure

- **Introduction**
    1. [Intro to Red Pitaya](/introduction/red_pitaya.md)
    2. [Setup Guide](/introduction/setup_guide.md)
    3. [Vivado Overview: Mapping Red Pitaya Hardware into Vivado](/introduction/vivado_overview.md)

- **Projects**
    1. [LED Blink](/projects/led_blink.md)
    2. [LED Control (GPIO)](/projects/led_control_gpio.md)
    3. [Signal Passthrough (ADC -> DAC)](/projects/signal_passthrough_adc_dac.md)
    4. [Photon Counter](/projects/photon_counter.md)

- **Theory**
    1. [Verilog](/theory/verilog.md)

## Notes and Advice:

1. This can be quite overwhelming if you don't have a background in it. There is a lot of new information to take in. Follow the repository structure. Don’t try to memorise everything at once, skim through the introduction — you’ll pick things up naturally as you do the projects. 

2. Use ChatGPT carefully: it can explain concepts well, but don’t rely on it for Verilog debugging or writing full code. (At least in my experience with version 5.) 

3. If you just want to program the Red Pitaya to do one of the projects, read through the [Setup guide](/introduction/setup_guide.md) and then go to the tutorial section of your desired project.

4. If you’re looking at this repository from the QEG’s GitLab, it is worth looking at the [original repository](https://github.com/sneha-balamurali/fpga/tree/main) for any changes.

## Resources:

This project draws on a range of resources. Each section of the documentation includes references to the specific materials used (e.g., AMD/Xilinx documentation and Red Pitaya schematics.)  

Listed below are some additional resources:

### [Red Pitaya Documentation:](https://redpitaya-knowledge-base.readthedocs.io/en/latest/learn_fpga/fpga_learn.html)

Use this as a guide. The tutorials there (e.g. the LED blink lesson) say to install Vivado 2020.1, because the original tutorial code was written and tested on Vivado 2020.1. You can install the latest version instead — most things still work if you adapt slightly.

### [Anton Potočnik:](https://antonpotocnik.com/?cat=29)

A good chunk of the FPGA lessons in the Red Pitaya Documentation come from Anton Potočnik's website. Worth looking in there as there is also an extra lesson not in the Red Pitaya Docs and he has some useful links to other resources. If you'd like more projects to practice, look here.

### [Pavel Demin github:](https://github.com/pavel-demin/red-pitaya-notes/tree/master/cores)

A useful reference because they provide ready-made IP cores (e.g. ADC, DAC).

### [Master's Thesis: Real-Time Processing of Laser-Ultrasonic Signals:](https://epub.jku.at/obvulihs/download/pdf/2406394?originalFilename=true )
Author: Thomas Paireder, Johannes Kepler University Linz, 2017

A lot of useful information on implementing real-time signal processing with the Red Pitaya. Many of the methods and design considerations are broadly applicable.

### [hls4ml:](https://fastmachinelearning.org/hls4ml/)

A python based tool that translates machine learning models into FPGA firmware. 

## Author’s Note & Disclaimer

**Author:** Sneha Balamurali

**Disclaimer:** This documentation is based on my own learning. While I’ve done my best to keep it accurate, there may be mistakes — please double-check against official references where needed.