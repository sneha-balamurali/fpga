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
    4. [Pulse Counter](/projects/pulse_counter.md)

- **Theory**
    1. [Verilog](/theory/verilog.md)

## Notes and Advice:

1. This can be quite overwhelming if you don't have a background in it. There is a lot of new information to take in. Follow the repository structure. Don’t try to memorise everything at once, skim through the introduction — you’ll pick things up naturally as you do the projects. 

2. Use AI carefully: it can explain concepts well, but don’t rely on it for Verilog debugging or writing full code. (At least in my experience with ChatGPT-5) 

3. If you just want to program the Red Pitaya to do one of the projects, read through the [Setup guide](/introduction/setup_guide.md) and then go to the tutorial section of your desired project.

## Resources:

This project draws on a range of resources. Each section of the documentation includes references to the specific materials used (e.g., AMD/Xilinx documentation and Red Pitaya schematics.)  

Listed below are some additional resources:

### Foundation Resources

- [**Red Pitaya Documentation:**](https://redpitaya-knowledge-base.readthedocs.io/en/latest/learn_fpga/fpga_learn.html)
    - Use this as a guide. The tutorials there (e.g. the LED blink lesson) say to install Vivado 2020.1, because the original tutorial code was written and tested on Vivado 2020.1. You can install the latest version instead — most things still work if you adapt slightly.

- [**Anton Potočnik Website:**](https://antonpotocnik.com/?cat=29)
    - Many of the FPGA lessons on the Red Pitaya website originate from this website. Their site includes additional projects and background reading not found in the official documentation. I would start here if you want to practice with more FPGA projects and then move onto the Red Pitaya Documentation lessons.

- [**Pavel Demin Github Repository:**](https://github.com/pavel-demin/red-pitaya-notes/tree/master/cores)
    - Invaluable reference containing ready-made IP cores (e.g., the ADC and DAC used in my projects are his cores).
    - Great for learning Verilog by looking at his code.

### Extended Reading and Real-Time Application

- [**Master's Thesis: Real-Time Processing of Laser-Ultrasonic Signals:**](https://epub.jku.at/obvulihs/download/pdf/2406394?originalFilename=true )
    - Author: Thomas Paireder, Johannes Kepler University Linz, 2017
    - A detailed study of real-time signal processing on the Red Pitaya
    - Covers FPGA control, signal acquisition and implementation methods that are broadly applicable to similar hardware setups. 

### Machine Learning and FPGA Co-Design

- [**hls4ml:**](https://fastmachinelearning.org/hls4ml/)
    - An open-source Python framework that converts machine learning models (e.g., Keras, PyTorch) into synthesisable FPGA firmware using High-Level Synthesis (HLS).

## Author’s Note & Disclaimer

**Author:** Sneha Balamurali

**Disclaimer:** This documentation is based on my own learning. While I’ve done my best to keep it accurate, there may be mistakes — please double-check against official references where needed.