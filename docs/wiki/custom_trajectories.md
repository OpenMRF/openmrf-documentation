The SPI readout module offers three options for the design of k-space trajectories, which can be selected by setting the variable `SPI.geo.traj_mode` to one of the following three options before running the `SPI_init()` function:

* `'standard'`: calls `SPI_traj_standard.m` to create a spiral trajectory based on geometric parameters, following the formulation by Kai Tobias Block[^1]. 
* `'vds'`: calls `SPI_traj_vds.m` to create a variable-density spiral based on geometric parameters and system specifications, using Brian Hargreaves' VDS toolbox [^2].  
* `'import'`: calls `SPI_traj_import.m` to allow the import of custom trajectories. In order to use this feature, the parameter `SPI.geo.path` must be set to the path of a `.mat` file containing two variables: `dur` (the desired duration of the readout in seconds), and either `gxy` (the gradient waveform as a complex array) or `kxy` (the k-space trajectory as a complex array). 
!!! warning 
    Importing custom trajectories is an experimental feature - use with care, desired performance cannot be guaranteed. Will likely be updated in future versions. 

[^1]: KT Block. Spiralförmige Abtastung des k-Raumes bei der Magnetresonanz-Tomographie. Diploma thesis, Georg-August-Universität zu Göttingen. 2004. [https://www.mpinat.mpg.de/597698/tblock_diploma.pdf](https://www.mpinat.mpg.de/597698/tblock_diploma.pdf) (accessed November 30, 2025). 
[^2]: The original toolbox used to be available under [http://mrsrl.stanford.edu/~brian/vdspiral/](http://mrsrl.stanford.edu/~brian/vdspiral/), but this link seems to not be active anymore. Parts of it can be found in [https://github.com/mribri999/MRSignalsSeqs/tree/master/Matlab](https://github.com/mribri999/MRSignalsSeqs/tree/master/Matlab). 