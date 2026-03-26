OpenMRF provides a dedicated workflow for accurate k-space trajectory measurement. This is particularly important when using demanding gradient waveforms (e.g., spiral or rosette), where system imperfections can lead to trajectory deviations.

Below, the full workflow is described step-by-step.

---

### 1. Generate MRF Sequence

Compile the MRF sequence that you plan to use for data acquisition.

Make sure that:
```matlab
flag_backup = 1;
```

This ensures that a Pulseq workspace backup file (`.mat`) is generated. This file contains the reference k-space trajectory required for calibration.

---

### 2. Generate Trajectory Calibration Sequence

Open:
```
main_sequences/calibrate_trajectories/pulseq_traj_meas.m
```

In the second section, update the path to the Pulseq workspace backup file of your MRF sequence:

```matlab
%% import SPI object for trajectory measurement
% load any backup file containing an SPI object
load('OpenMRF/Pulseq_Workspace/user/YYMMDD/YYMMDD_HHMM/backup_YYMMDD_HHMM_workspace.mat')
```

---

### Calibration Method

OpenMRF supports two established approaches for trajectory measurement:

- **Duyn et al. (1998)**  
  Uses a single thin slice with an offset and measures phase evolution per gradient axis.  
  [doi: 10.1006/jmre.1998.1396](https://doi.org/10.1006/jmre.1998.1396)  

- **Robison et al. (2019)** *(default)*  
  Uses two slices with positive and negative offsets and measures both gradient polarities.  
  This allows correction of both B0 offsets and eddy current effects.  
  [doi: 10.1002/mrm.27583](https://doi.org/10.1002/mrm.27583)  

By default, OpenMRF uses the **Robison method**.

---

### Sequence Parameters

You can define the parameters of the calibration sequence as follows:

```matlab
%% sequence objects for trajectory measurement

% select method
% duyn:    10.1006/jmre.1998.1396
% robison: 10.1002/mrm.27583
TRAJ.method = 'robison';

% number of repetitions for averaging
TRAJ.Nav = 5;

% number of dummy scans
TRAJ.Ndummy = 50;

% recovery time between repetitions
TRAJ.Trec = 150 *1e-3; % [s]

% analyze phase evolution in a thin slice far from the iso-center
TRAJ.slice_thickness = 2 *1e-3;  % [m] slice thickness
TRAJ.slice_offset    = 50 *1e-3; % [m] slice offset

% flip angle
TRAJ.exc_fa = 20 *pi/180;

% reduce stimulation
TRAJ.lim_slew = 0.5;

% init seq objects for trajectory scans
[TRAJ, FOV] = TRAJ_init(TRAJ, PULSEQ.PULSEQ_SPI.SPI, system);
```

!!! note
    A typical spiral calibration with 48 projections takes approximately **7 minutes**.  
    More complex trajectories (e.g., rosette) may require significantly longer scan times due to:

    - a higher number of unique projections  
    - longer waveform durations  

    If scan time or system storage becomes limiting, consider splitting the calibration into multiple scans.

---

### 3. Trajectory Measurement

We recommend performing trajectory calibration immediately after the MRF scan using a **spherical calibration phantom**.

- Place the phantom exactly at the **isocenter**
- Copy the **slice orientation** from the MRF protocol  
- **Do NOT copy the slice offset** (must remain at isocenter)
- Ensure at least **50 mm distance** in all directions for reliable phase evolution measurements

!!! important
    Only the slice rotation must match the MRF sequence.  
    The calibration sequence uses internally defined slice offsets.

---

### 4. Use Measured Trajectories for Reconstruction

Use the example reconstruction script:

```
main_sequences/fingerprinting/reco_mrf.m
```

In the first section of the script, define the path to the calibration raw data:

```matlab
path_raw_traj = '...';
```

The trajectory processing is performed automatically via:

```matlab
TRAJ_reco();
```

The reconstructed trajectories are then used for MRF reconstruction.