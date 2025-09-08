# Definitive MIMO Imaging Suite for Biomedical Analysis: Technical Report

![Python Version](https://img.shields.io/badge/python-3.9+-blue.svg)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![Status](https://img.shields.io/badge/status-active-success.svg)

---

## Authorship and Contribution

This project was a collaborative effort between:

*   **Shahzaib Ur Rehman:** Project lead, conceptual design, and supervision (70% contribution).
*   **Google's Gemini:** Core code generation, implementation of features, and UI refinement (30% contribution).

The synergy between human-led ideation and AI-powered implementation was key to the rapid development and advanced feature set of this application.

---

## Abstract

This report details the design, functionality, and technical implementation of the **Definitive MIMO Imaging Suite**, a comprehensive graphical software application for simulating and analyzing Multiple-Input Multiple-Output (MIMO) radar imaging for biomedical applications, with a primary focus on non-invasive skin cancer detection. The suite integrates two distinct simulation cores: a fast, parametric **Analytical Model** based on the radar range equation for system-level analysis, and a high-fidelity **Finite-Difference Time-Domain (FDTD) Model** powered by the Meep library for physically accurate electromagnetic wave simulation. The software implements several key signal processing algorithms, including Delay-and-Sum (DAS) beamforming for 2D image reconstruction and high-resolution angular spectrum estimation techniques (MUSIC, Capon). The suite is presented as a multi-tabbed graphical user interface (GUI) providing specialized tools for direct imaging, algorithm validation, side-by-side model comparison, and in-depth parametric analysis of system performance. This tool serves as a powerful testbed for researchers, educators, and engineers to explore the capabilities and limitations of MIMO radar for near-field medical imaging.

---

## 1. Introduction

### 1.1. Motivation
The early and accurate detection of cutaneous malignancies, such as melanoma, is critical for improving patient outcomes. Traditional diagnostic methods, while effective, often rely on invasive biopsies. There is a compelling need for non-invasive, non-ionizing, and cost-effective imaging modalities that can accurately characterize subsurface tissue structures. Millimeter-wave (mmWave) radar technology presents a promising solution due to its ability to penetrate superficial tissue layers and differentiate between tissues with varying dielectric properties.

### 1.2. MIMO Radar for Medical Imaging
Multiple-Input Multiple-Output (MIMO) radar systems utilize multiple transmitting (Tx) and receiving (Rx) antennas to form a large virtual aperture. This technique significantly enhances spatial resolution without requiring a physically large antenna array, making it ideal for compact, near-field imaging devices. By processing the signals from each Tx-Rx pair, it is possible to computationally reconstruct a high-resolution image of the subsurface, revealing anomalies like tumors which exhibit different permittivity and conductivity compared to healthy tissue.

### 1.3. Project Objectives
The primary objective of this project was to develop a versatile and user-friendly software suite to bridge the gap between theoretical system design and practical performance analysis. The key goals were:
*   To create a dual-core simulation environment integrating both fast analytical models and physically accurate FDTD models.
*   To implement and visualize standard and advanced image reconstruction algorithms.
*   To build intuitive tools for analyzing critical system trade-offs, such as detection depth versus frequency, power, and material properties.
*   To package the entire system in a responsive, multi-threaded GUI that serves as an accessible tool for research and education.

---

## 2. System Architecture & Design

### 2.1. Software Stack
The application is built entirely in Python 3, leveraging a powerful stack of open-source scientific libraries:
*   **Python 3.9+:** The core programming language.
*   **Tkinter:** For building the native, cross-platform graphical user interface.
*   **Matplotlib:** For generating all 2D plots and visualizations, which are embedded directly into the GUI.
*   **NumPy & SciPy:** For high-performance numerical computation, signal processing, and handling of physical constants.
*   **Meep:** The core FDTD engine for solving Maxwell's equations.

### 2.2. Graphical User Interface (GUI)
The GUI is designed with a tabbed layout (`ttk.Notebook`) to logically separate the distinct functionalities. This modular approach allows users to focus on one specific task at a time, from initial simulation to comparative analysis. All parameters are exposed through intuitive widgets like entry boxes, dropdown menus, and checkboxes. A scrollable frame is used in each tab to ensure that a large number of parameters can be configured without cluttering the interface.

### 2.3. Asynchronous Task Handling
A critical design feature is the prevention of GUI freezing during computationally intensive tasks. All simulations (both Analytical and FDTD) are executed in separate background threads using Python's `threading` library.
*   A polling mechanism (`root.after()`) is used to periodically check the status of the background thread from the main GUI thread.
*   A `threading.Event` object is implemented for each simulation tab. This acts as a thread-safe flag that the "Stop" button can set. The long-running loop in the simulation thread periodically checks this flag and terminates cleanly if it has been set, providing robust user control over the application.

---

## 3. Simulation Methodologies

The suite's power lies in its two complementary simulation methodologies.

### 3.1. Analytical Model
This model provides a rapid, first-order approximation of the system's performance.
*   **Theoretical Basis:** It is built upon the radar range equation, which calculates the received signal power based on transmitted power, antenna gain, target reflectivity (Radar Cross-Section, RCS), and path loss.
*   **Path Loss Calculation:** The model accounts for both Free-Space Path Loss (FSPL) and material-dependent attenuation. It calculates attenuation through a multi-layered medium by segmenting the signal's path through each layer and applying the specific loss (derived from permittivity and conductivity) for that segment.
*   **Noise Modeling:** System noise is modeled based on thermal noise (`kTB`) and the receiver's noise figure, allowing for realistic Signal-to-Noise Ratio (SNR) calculations.
*   **Use Case:** Ideal for rapid parametric sweeps, system design trade-offs, and validating the logic of beamforming algorithms without the computational overhead of FDTD.

### 3.2. Finite-Difference Time-Domain (FDTD) Model
This model provides a high-fidelity, physically accurate simulation by numerically solving Maxwell's curl equations in the time domain.
*   **Meep Engine:** The implementation relies on the open-source Meep library.
*   **Simulation Setup:** A 2D computational domain is defined, surrounded by Perfectly Matched Layers (PML) to absorb outgoing waves and prevent reflections from the boundaries. Geometric objects (skin layers, tumors) are defined with their specific dielectric properties. A Gaussian-pulsed source is used to excite the domain, and DFT monitors at the receiver locations record the electric field over time.
*   **Data Acquisition:** The simulation is run for both a "healthy" baseline scenario (no tumor) and an "unhealthy" scenario. The differential signal (unhealthy - healthy) is then used for image reconstruction, effectively performing background subtraction to isolate the target's response.
*   **Use Case:** Serves as the "ground truth" for validating the analytical model and for generating highly realistic imaging results that account for complex wave phenomena like scattering, diffraction, and multipath effects.

---

## 4. Signal Processing & Image Reconstruction

### 4.1. MIMO Virtual Array
The signals from all `NumTx * NumRx` channels are treated as originating from a single, large virtual array. This synthesized aperture is the key to the high spatial resolution achieved by the system.

### 4.2. Delay-and-Sum (DAS) Beamforming
DAS is the primary algorithm used for 2D image reconstruction. The process involves:
1.  Defining a grid of pixels in the region of interest.
2.  For each pixel, coherently summing the signals from all MIMO channels.
3.  Before summing, each signal is phase-shifted by an amount that precisely compensates for the two-way propagation delay from the transmitter to that pixel and back to the receiver.
4.  This process effectively focuses the array's energy at each pixel, causing signals originating from that location to add constructively while signals from other locations add destructively. The result is an image of the scene's reflectivity.

### 4.3. High-Resolution Angular Spectrum Estimation
For 1D direction-of-arrival analysis, the suite includes two advanced, subspace-based algorithms:
*   **MUSIC (MUltiple SIgnal Classification):** This algorithm provides significantly higher angular resolution than DAS by exploiting the orthogonality between the "signal subspace" and the "noise subspace" of the received data's covariance matrix.
*   **Capon (MVDR):** The Capon method, or Minimum Variance Distortionless Response beamformer, also offers improved resolution by adaptively minimizing the contribution of noise and interference from directions other than the one being observed.

---

## 5. Application Modules (Tabs)

### 5.1. Screenshots

**Tab 1: Main FDTD Simulation for Skin Imaging**
![Screenshot of the main FDTD simulation tab showing a reconstructed tumor image](https://github.com/your-username/your-repository-name/blob/main/images/Screenshot%20From%202025-09-08%2021-27-47.png?raw=true)

**Tab 4: Comparative Analysis (Analytical vs. FDTD)**
![Screenshot of the TestbedV2 tab showing side-by-side results of the analytical and FDTD models](https://github.com/your-username/your-repository-name/blob/main/images/Screenshot%20From%202025-09-08%2022-52-46.png?raw=true)

**Tab 6: Parametric Analysis Heatmap**
![Screenshot of the Parametric Analysis tab displaying an SNR heatmap](https://github.com/your-username/your-repository-name/blob/main/images/Screenshot%20From%202025-09-08%2022-53-34.png?raw=true)

### 5.2. Module Functionality

*   **Tab 1: Skin Cancer Imaging (FDTD):** The main simulator. Configure a multi-layered skin and tumor model, run a full-wave FDTD simulation, and reconstruct a 2D image.
*   **Tab 2: Algorithm Testbed (Analytical):** Rapidly test imaging algorithms using a fast mathematical model. Ideal for understanding the performance of DAS, MUSIC, and Capon without waiting for FDTD.
*   **Tab 3: Full-Wave EM Testbed (FDTD):** A generalized FDTD testbed for running simulations with more complex multi-target and multi-layer scenarios.
*   **Tab 4: TestbedV2 (Analytical vs FDTD):** A powerful validation tool to directly compare the results of the fast analytical model against the high-fidelity FDTD model using a single set of parameters.
*   **Tab 5: Depth Analysis Tool:** An analytical tool to calculate and visualize the maximum depth at which a target can be detected based on system parameters and a defined Signal-to-Noise Ratio (SNR) threshold.
*   **Tab 6: Parametric Analysis:** An advanced module to generate 2D heatmaps showing how system SNR changes as you sweep two parameters simultaneously (e.g., depth vs. frequency), available in both Analytical and FDTD modes.

---

## 6. Conclusion & Future Work

The **Definitive MIMO Imaging Suite** successfully meets its objective of providing a comprehensive, multi-modal simulation environment for biomedical radar imaging. By integrating fast analytical models with accurate FDTD simulations, it enables a complete workflow from initial system design to high-fidelity performance validation.

Potential directions for future work include:
*   **Advanced Algorithms:** Implementation of more sophisticated reconstruction algorithms like Kirchhoff Migration or Compressive Sensing.
*   **3D Simulation:** Extending the FDTD core to support full 3D models for enhanced realism, at the cost of significantly increased computation time.
*   **Material Library:** Incorporating a database of frequency-dependent dielectric properties for various biological tissues.
*   **Machine Learning Integration:** Adding a module to train and test convolutional neural networks (CNNs) for automated tumor detection and classification from the reconstructed images.

---

## 7. Installation

This application requires Python and several scientific libraries. The FDTD engine depends on **Meep**, which has specific installation requirements.

### 7.1. Prerequisites
*   Python 3.9 or newer.
*   We strongly recommend using a Conda environment, as it greatly simplifies the installation of Meep.

### 7.2. Setup Instructions

**Step A: Clone the Repository**
```bash
git clone https://github.com/your-username/your-repository-name.git
cd your-repository-name
