!!! warning
    Site under construction

OpenMRF offers the option to calibrate your trajectory. This feature is especially relevant if you're using demanding gradient operations at higher field strengths. In the following, we'll outline the process in five easy steps. 

### 1. MRF sequence creation. 
Compile any MRF sequence that you want to use for data acquisition. Make sure `flag_backup` is set to 1. 

### 2. Trajectory calibration sequence creation. 
Open `main_sequences/calibrate_trajectories/pulseq_traj_meas.m`. In the second cell, change the path of the SPI object to be loaded to the backup `.mat` file corresponding to the MRF sequence in question. 

### 3. Measurement

### 4. Reconstruction - standalone (optional)

### 5. Use in MRF reconstruction