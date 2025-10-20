# FPGA Internship Documentation

This repository is documentation for my eight-week summer internship with the Quantum Engineering Group. It includes setup guides, project walkthroughs, and technical notes developed while working with the **Red Pitaya STEMlab 125-14 (Gen 1)** using **Vivado** and **Verilog.**

![high_level_system_overview](/images/read_me/high_level_system_overview.png)

*Figure 1:* The diagram illustrates how the Red Pitaya STEMlab 125-14 interfaces with a host PC and external signal instruments. Solid arrows represent data, signal, and command flow between the host PC, Red Pitaya, and connected instruments, while dashed arrows indicate an optional closed-loop feedback path where outputs are fed back as inputs for adaptive control.

## Repository Structure

- **Introduction**
    1. [Introduction to Red Pitaya](/introduction/red_pitaya.md)
        - Hardware overview
    2. [Setup Guide](/introduction/setup_guide.md)
        - Step-by-step installation and connection guide
        - How to communicate with the Red Pitaya via SSH
    3. [Vivado Overview: Mapping Red Pitaya Hardware into Vivado](/introduction/vivado_overview.md)
        - Getting started with Vivado
        - Basics of Block Design and IP integration

- **Projects**
    1. [LED Blink](/projects/led_blink.md)
        - First FPGA project: creating a simple LED blink circuit 
    2. [LED Control (GPIO)](/projects/led_control_gpio.md)
        - Controlling on-board LEDs via computer using AXI GPIO
    3. [Signal Passthrough (ADC -> DAC)](/projects/signal_passthrough_adc_dac.md)
        - Implementing a analog signal passthrough between ADC and DAC channels
    4. [Pulse Counter](/projects/pulse_counter.md)
        - Detecting and counting LVTTL pulses (e.g., photon or TTL events) via digital E1 pins using FPGA logic

- **Theory**
    1. [Verilog](/theory/verilog.md)
        - Resources for learning Verilog HDL fundamentals

## Notes and Advice:

1. This can be quite overwhelming if you don't have a background in it. If you're completely new to FPGAs, I recommend following the order shown above and treating each project as a building block. 

2. Use AI assistants carefully: it can explain concepts well, but don’t rely on it for Verilog debugging or writing full code. (At least in my experience with ChatGPT-5) 

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

- [**AMD Technical Information Portal**](https://docs.amd.com/)
    - Many of the documents referenced throughout this repository are sourced from the AMD library. These include user guides, product specifications, and technical reference manuals for Vivado IP and Zynq devices.

- [**FPGA-101**](https://nandland.com/fpga-101/)
    - A great resource for FPGA fundamentals. Their explanations of metastability and clock domain crossing were particularly helpful when implementing the two-flip-flop synchroniser and rising-edge detector described in the [further steps](/projects/pulse_counter.md#further-steps) section of my pulse counter.

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