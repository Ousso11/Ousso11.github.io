---
title: "Command-Collecting Robot: Embedded Systems Project for Restaurant Automation"
excerpt: "A robotic system developed for real-time order collection in a restaurant environment, combining line-following, object detection, and GUI-based control. The robot uses image processing, multithreading, and Bluetooth communication to collect orders from tables and transmit them for analysis and optimization."
collection: course_projects
share: false
related: false
---

## Problem Statement & Motivation

As part of the **Embedded Systems and Robotics** course at EPFL, this mini-project aimed to implement a real-world application using the **e-puck2 robot**. The challenge: create a robot capable of autonomously navigating a restaurant to collect orders from clients.

The project was born from the idea of putting robotics at the service of people, particularly in the hospitality sector. The robot should be able to detect paths, avoid obstacles (like tables), and scan commands encoded in black lines of different widths on the ground.

## Our Method

We designed and implemented a system combining hardware and software components, integrating sensors, motors, and computer vision. The software stack is split between onboard C modules running on the robot, and Python modules running on a computer with a GUI.

### 1. Robot Design & Sensors

- **Camera with mirror**: Detects the line and reads the "barcode" (black line width).
- **Infrared proximity sensor**: Detects tables (obstacles).
- **Stepper motors**: Handle movement and turns.
- **ESP32 module**: Handles Bluetooth communication between the robot and GUI.

### 2. Software Architecture

- C-based firmware using **ChibiOS** with multithreaded architecture for:
  - Line-following using **PID control**
  - Real-time obstacle detection
  - Image capture and processing

- Python-based interface:
  - GUI built with **CustomTkinter**
  - Live robot position, speed, command, and task tracking
  - CSV data export for path, table positions, and scanned commands

### 3. Modes of Operation

- **Scan Mode**: Maps the restaurant by following the line and scanning tables.
- **Work Mode**: Follows the mapped path to continue order collection.
- **Stop Mode**: Emergency stop via GUI.

### 4. Optimization Techniques

- **Memory**: Compact data struct using bitwise encoding.
- **Thread Management**: Thread minimization and synchronization for real-time performance.
- **Computation**: Integer math for PID, optimized image analysis via line intensity thresholds.

## Results

The robot successfully performed:

- **Navigation** via black line detection
- **Order recognition** based on line width (Burger, Pizza, Pasta)
- **Table identification** via proximity sensor
- **Real-time GUI feedback** and **data logging** for analysis

The final application enables predictive analytics and restaurant workflow optimization using stored CSV data (commands, paths, tables).

## What's Missing / Future Work

- Integration with **menu recommendation systems**
- **Wireless charging** and **autonomous docking**
- **Multilingual voice interface** for human interaction
- More advanced **barcode or QR scanning**


## Publication & Resources
- **Authors**: Oussama Gabouj, Adam Ben SLama
- **Report PDF**: [Download Full Project Report (PDF)](https://Ousso11.github.io/files/RobotiqueProject.pdf)
