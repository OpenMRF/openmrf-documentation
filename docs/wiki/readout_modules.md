Originally, the OpenMRF project did not start with the goal of unifying MRF, but rather with the simple aim of creating a modular framework for MRI. In the directory `include_pulseq_toolbox/src_readouts`, you can find all kinds of readouts, which can be easily combined with [preparations](https://openmrf.org/wiki/preparation_modules/). All readouts are initialized with a respective `..._init()` function and can be used with a corresponding `..._add()` function. For MRF, the central readout package is SPI. Historically, this package was designed solely for spiral readouts. Today, SPI can be used to implement different trajectories, including radial, rosette, and cones. In addition to the readout trajectory, SPI can be used for FLASH-like, FISP-like, or bSSFP-like acquisitions, and the package can be used for 2D, stacked 3D, or 3D k-space sampling. In the following, the SPI package is explained step by step. At the end of the page, we also describe the other readouts, which can be used for non-MRF purposes.

### Step 1: Define the Field of View
The field-of-view parameters are stored in the `FOV` struct. It is important to distinguish between 2D, stacked 3D, and 3D acquisitions.

=== "2D"
    For a 2D readout, only the matrix size, FOV geometry, and slice thickness need to be defined. Note that the SPI package only supports quadratic and isotropic FOVs.
    ```matlab
    FOV.Nxy    = 256;        % [ ] matrix size
    FOV.fov_xy = 256 *1e-3;  % [m] FOV geometry
    FOV.dz     = 5   *1e-3;  % [m] slice thickness
    FOV_init();
    ```

=== "stacked 3D"
    A stacked 3D readout additionally needs the number of kz partitions and the FOV dimension along z. The FOV still needs to be isotropic along x and y. Note that `FOV.fov_z` is used to define the z-encoding steps. `FOV.dz` is the slice (or rather slab) thickness and is used to calculate the selective excitation pulse. It is reasonable to make the slab a little smaller than the encoded z dimension to avoid aliasing.
    ```matlab
    FOV.Nxy    = 256;        % [ ] matrix size
    FOV.Nz     = 48;         % [ ] number of "stack-of-spirals"
    FOV.fov_xy = 256 *1e-3;  % [m] FOV geometry
    FOV.fov_z  = 96  *1e-3;  % [m] FOV geometry
    FOV.dz     = FOV.fov_z * 0.8;  % [m] slab thickness, reduce to prevent aliasing along z
    FOV_init();
    ```

=== "3D"
    A 3D readout needs to be isotropic and samples a cube. It is reasonable to make the slab a little smaller than the encoded z dimension to avoid aliasing.
    ```matlab
    FOV.Nxy    = 256;             % [ ] matrix size
    FOV.fov_xy = 256  *1e-3;      % [m] FOV geometry
    FOV.fov_z  = FOV.fov_xy;      % [m] FOV geometry; 3D readout is isotropic
    FOV.dz     = FOV.fov_z * 0.8; % [m] slab thickness (reduce to prevent aliasing)
    FOV_init();
    ```

### Step 2: Define flip angles and repetition times
For MRF, we store the flip angles and repetition times in the `MRF` struct. There are a few pre-implemented options to set these parameters. However, you can also freely design them or load parameters from a `.mat` file.

=== "Published Patterns"
    You can find multiple TR and flip-angle patterns within OpenMRF that were previously published. For example, we stored the pattern of the original IR-FISP MRF paper (['yun'](https://doi.org/10.1002/mrm.25559)) and another pattern (['cao'](https://doi.org/10.1002/mrm.29194)), which is frequently used for 3D brain MRF.
    ```matlab
    MRF.pattern         = 'yun'; % select pattern: e.g. 'yun' or 'cao'
    [MRF.FAs, MRF.TRs]  = MRF_get_FAs_TRs(MRF.pattern, 1);
    MRF.FAs(MRF.FAs==0) = 1e-6;
    seq_name            = [seq_name MRF.pattern];
    MRF.NR              = numel(MRF.FAs);
    ```

=== "Custom Sinusoidals"
    You can also calculate a custom sinusoidal flip-angle pattern using the `MRF_calc_FAs_sin()` function. The input needs to be `nx3`. In one line, the first value is the starting flip angle, the second value is the peak flip angle, and the third value is the number of flip angles for the half-sinusoid. The total number `NR` will be the sum of the third column.
    ```matlab
    MRF.pattern = 'sin_70';
    MRF.FAs     = MRF_calc_FAs_sin([5, 30, 200; 1, 70, 200; 10, 10, 100]) *pi/180;
    MRF.NR      = numel(MRF.FAs);
    MRF.TRs     = 12 *1e-3 *ones(MRF.NR,1);
    seq_name    = [seq_name MRF.pattern];
    ```

=== "Random Sinusoidals"
    For cardiac MRF, we usually use random sinusoidal flip-angle patterns. Random means that the peak flip angle for one half-sinusoid is randomly selected between a defined minimum and maximum value. For cardiac MRF, `MRF.nr` defines the number of TRs for one segment (or heartbeat). The total number of TRs is then `MRF.nr * MRF.n_segm`. Note that you can set `MRF.TRs` to 0, which will minimize the TR.
    ```matlab
    MRF.nr     = 48;                         % numer of readouts per hear beat
    MRF.NR     = MRF.n_segm * MRF.nr;        % total number of readouts
    MRF.TRs    = 0.0 *1e-3 *ones(MRF.NR,1);  % minimize TRs
    MRF.FA_min = 4 *pi/180;                  % [rad] minimum flip angle
    MRF.FA_max = 15 *pi/180;                 % [rad] minimum flip angle
    MRF.FAs    = MRF_calc_FAs_sin_rand(MRF.FA_min, MRF.FA_max, MRF.nr, MRF.n_segm);
    ```

### Step 3: Define basic readout parameters
All parameters of the readout are stored in the `SPI` struct. The setting of `2D`, `3D_stacked`, or `3D` must match the settings in `FOV`. The receiver bandwidth can be chosen freely; however, the final bandwidth might be slightly different, since the dwell time must fit the `adcRasterTime` (see terminal output). TRs and FAs are defined in Step 2 and passed to the `SPI` struct.

```matlab
% basic/import params for MRF
SPI.mode_2D_3D     = '2D';      % '2D', '3D' or '3D_stacked'
SPI.adcBW          = 400 *1e3;  % [Hz] desired receiver bandwidth
SPI.NR             = MRF.NR;    % [ ] number of repetitions
SPI.mrf_import.TRs = MRF.TRs;   % [s] repetition times
SPI.mrf_import.FAs = MRF.FAs;   % [rad] flip angles
```

The slice excitation can use the default Pulseq sinc pulse based on `mr.makeSincPulse()`, or you can use an optimized SLR (Shinnar–Le Roux) pulse calculated with sigpy. Note that we implemented a Matlab clone of the sigpy SLR package, which is located in `include_pulseq_toolbox/src_misc/SIGPY`, so you do not need Python communication with Matlab. For MRF, the `'import'` mode is correct. `'ramped'` and `'equal'` modes are used for conventional imaging. If you observe stimulation issues, the duration of the slice rephaser can be modified.

```matlab
% slice excitation
SPI.exc_mode      = 'sinc';      % 'sinc' or 'sigpy_SLR'
SPI.exc_shape     = 'ex';        % only for sigpy: 'st' or 'ex'
SPI.exc_time      = 2.0 *1e-3;   % [s] excitation time
SPI.exc_tbw       = 6;           % [ ] time bandwidth product
SPI.exc_fa_mode   = 'import';    % 'equal',  'ramped',  'import'
SPI.reph_duration = 1.0 *1e-3;   % [s] slice rephaser duration
```

The SPI package supports all main spoiling modes. You can choose between gradient and/or RF spoiling. The number of 2pi twists within the selected slice should be an integer value greater than 1. Note that for stacked 3D, the number of twists needs to be greater than 1 in each subslice, which is ensured by multiplication with `FOV.Nz`. For a 3D readout, replace `FOV.Nz` with `FOV.Nxy`. RF spoiling can be done either with linear or quadratic phase increments. bSSFP works with linear spoiling, while FLASH needs quadratic spoiling. For bSSFP, the RF increment should be 180°, while FLASH uses 117°. For FISP, the RF increment is 0°, and linear or quadratic spoiling makes no difference.

```matlab
% params: gradient spoiling & rf spoiling
SPI.spoil_nTwist   = 4 * FOV.Nz; % [ ] number of 2pi twist in slice
SPI.spoil_rf_mode  = 'lin';      % 'lin' or 'quad'
SPI.spoil_rf_inc   = 0 *pi/180;  % [rad] rf phase increment
SPI.spoil_duration = 1.0 *1e-3;  % [s] slice spoiler duration
```

### Step 4: Define k-space trajectory
The SPI package supports more than just spirals. You only need to choose a few parameters in the geometry struct `SPI.geo` to switch between spiral, radial, rosette, cones, or Seiffert spiral trajectories. Tip: you can test the different geometries in `include_pulseq_toolbox/src_readouts/SPI/src_trajectories`.

Note: OpenMRF uses a toolbox for calculating time-optimized gradient waveforms for arbitrary k-space curves. This toolbox was originally implemented by [Miki Lustig](https://people.eecs.berkeley.edu/~mlustig/software) and presented [here](https://people.eecs.berkeley.edu/~mlustig/tOptGrad_ISMRM12.pdf).

=== "Spiral"
    The most established k-space trajectory for MRF is variable-density sampling. For the basic geometry, OpenMRF uses the [vds toolbox](https://github.com/mribri999/MRSignalsSeqs/tree/master/Matlab) proposed by [Brian Hargreaves](https://doi.org/10.1002/mrm.20489). `N_interleaves` defines the number of interleaves that are necessary to sample the k-space center. `FOV_coeff` defines the decreasing density. In the current example, `[1 -0.5]` means that 24 interleaves are required for the center and 48 for the k-space periphery. `Ns` and `ds` are parameters for time optimization of the gradient waveform. `Ns` is the number of samples for the initial curve parametrization, and `ds` is the step size for ODE integration. Tip: use `1e5` and `1e-3` for your final optimization. You can use `ds = 1e-2` to speed up the calculation while searching for a good waveform. If `flag_rv` is set to 0, the ODE will find a rotationally invariant solution, which means the gradient and slew-rate limits will not be violated even for double-oblique slice orientations (recommended).
    ```matlab
    SPI.geo.design        = 'spiral';
    SPI.geo.design_fun    = 'hargreaves';
    SPI.geo.N_interleaves = 24;
    SPI.geo.FOV_coeff     = [1 -0.5];
    SPI.geo.Ns            = 1e5;
    SPI.geo.ds            = 1e-3;
    SPI.geo.flag_rv       = 0;
    ```

=== "Radial"
    **Option I:** full projection  
    For radial readouts, only the prephaser and rephaser gradients are time-optimized. The duration of the full-projection readout is defined with `t_adc`.
    ```matlab
    SPI.geo.design = 'radial';
    SPI.geo.t_adc  = 3.2 *1e-3;
    ```

    **Option II:** half projection  
    For radial half-projection readouts, the duration of the trapezoid can be fixed with `t_adc`.
    ```matlab
    SPI.geo.design     = 'radialHalf';
    SPI.geo.design_fun = 'fixedTimeTrap';
    SPI.geo.t_adc      = 3.2 *1e-3;
    ```

    **Option III:** half projection (fast)  
    For fast radial half-projection readouts, use the `'minTimeTrap'` mode.
    ```matlab
    SPI.geo.design     = 'radialHalf';
    SPI.geo.design_fun = 'minTimeTrap';
    ```

=== "Rosette"
    Rosette trajectories only need the number of rosette loops as input. However, OpenMRF will find the time-minimized waveform depending on the current gradient and slew-rate limits. This means that the time points of k-space revisits might not be sufficient for fat suppression. Calculating a rosette trajectory will also visualize a simulation result for fat suppression. If the fat frequency is not suppressed, increase or decrease the slew rate to shift the fat frequency to a minimum of the frequency response.
    ```matlab
    SPI.geo.design  = 'rosette';
    SPI.geo.N_lobes = 17;
    SPI.geo.Ns      = 1e5;
    SPI.geo.ds      = 1e-3;
    SPI.geo.flag_rv = 0;
    ```

=== "Cones"
    OpenMRF also provides a 3D readout based on cones. This feature has not yet been tested for MRF. Let us know if you have any suggestions.
    ```matlab
    SPI.geo.design    = 'cones';
    SPI.geo.N_loops   = 16;
    SPI.geo.theta     = pi/8;
    SPI.geo.FOV_coeff = [1 -0.5];
    SPI.geo.Ns        = 1e5;
    SPI.geo.ds        = 1e-3;
    SPI.geo.flag_rv   = 0;
    ```

=== "Seifferts"
    OpenMRF also provides a 3D readout based on Seiffert spirals. This feature has not yet been tested for MRF. Let us know if you have any suggestions.
    ```matlab
    SPI.geo.design   = 'seiffert';
    SPI.geo.ell_mod  = 0.2;
    SPI.geo.s_range  = 10 *2*pi;
    SPI.geo.weight   = 1.0;
    SPI.geo.Ns       = 1e5;
    SPI.geo.ds       = 1e-3;
    SPI.geo.flag_rv  = 0;
    ```

=== "Import"
    You don't like our implementations or you're super picky?! No problem — simply use the import option and load your own trajectory from a `.mat` file. The file must meet the following requirements: it must contain a variable `dur`, which defines the total readout duration in [s], and either the variable `kxy` or `gxy`, representing a k-space or gradient trajectory in complex-valued `Nx1` format. The exact dimension is not critical, as the trajectory will be automatically rescaled to match the properties defined in the `FOV`.

    ```matlab
    SPI.geo.design = 'import';
    SPI.geo.path   = '.../path/to/your/file.mat';
    ```
    
### Step 5: Gradient & slew rate limits
The last step in defining the readout is to set the gradient and slew-rate limitation factors. Note that `lim_read_grad` and `lim_read_slew` directly impact the k-space sampling. However, this section can also be used to change the slice excitation, slice rephasing, and z-spoiling gradients. If you observe stimulation issues, you can use this section to correct them.

```matlab
% gradient & slew rate limitation factors
SPI.lim_exc_slew   = 0.9; % slice excitation
SPI.lim_reph_grad  = 0.9; % slice rephasing
SPI.lim_reph_slew  = 0.6; % slice rephasing
SPI.lim_read_grad  = 0.9; % readout
SPI.lim_read_slew  = 0.5; % readout
SPI.lim_spoil_grad = 0.9; % slice spoiling
SPI.lim_spoil_slew = 0.6; % slice spoiling
```

### Step 6: Projection mode
After finalizing all geometry parameters, a projection mode needs to be selected. The default option for MRF is an approximated golden-angle pattern, `'RoundGoldenAngle'`. Here, the user selects a number of unique projections (e.g. 48). If the MRF sequence has 1000 TRs, we first calculate a projection angle based on the golden ratio for each readout and finally round each angle to the nearest of 48 unique angles, which are uniformly distributed around `2pi`. This results in a golden-angle-like pattern, which has the basic features of random aliasing, but with the advantage that fewer unique trajectories are needed. This reduces the computational burden of the reconstruction. As an alternative, you can use `'Equal2Pi'`, `'Random2Pi'`, `'GoldenAngle'`, `'RandomGoldenAngle'`, or `'FibonacciSphere'` for 3D.

```matlab
% k-space projection params
SPI.proj.mode = 'RoundGoldenAngle';
SPI.proj.Nid  = 48; % number of unique projections
```

### Step 7: Initialize the SPI objects
Finally, the `SPI_init()` function is used to calculate all Pulseq objects related to the readout.

```matlab
[SPI, ktraj_ref] = SPI_init(SPI, FOV, system, 1);
```

### Step 8: Add SPI readouts
The `SPI_add()` function is used to add readouts to the `.seq` object. The `loop_NR` counter defines which projection is used. The `loop_kz` counter specifies the z-partition for stacked 3D mode (default: 1). The `loop_rf_inc` counter controls RF spoiling and is automatically incremented.

```matlab
for loop_NR = 1:SPI.NR
    SPI_add();
end
```

### Overview: All Readout Modules
The source code for the following modules can be found in `include_pulseq_toolbox/src_readouts`. Please note that only the SPI package is designed for MRF.

| Module Name | Long Name | Notes |
|-|-|-|
| BSSFP | <b>B</b>alanced <b>S</b>teady <b>S</b>tate <b>F</b>ree <b>P</b>recession | A 3D-ready BSSFP readout. |
| EPI | <b>E</b>cho <b>P</b>lanar <b>I</b>maging | Cloned and packaged from [here](https://pulseq.github.io/writeEpiRS.html). |
| GRE | <b>GR</b>adient <b>E</b>cho | Cloned and packaged from [here](https://pulseq.github.io/writeGradientEcho.html). |
| PRESS | <b>P</b>oint <b>RES</b>olved <b>S</b>pectroscopy | Can be used for NMR spectroscopy. |
| RAD | <b>RAD</b>ial | Cloned and packaged from [here](https://pulseq.github.io/writeRadialGradientEcho.html). |
| SPI | <b>SPI</b>ral | Our main readout for MRF. Not only spiral. |
| SPITSE | <b>Spi</b>ral <b>T</b>urbo <b>S</b>pin <b>E</b>cho | Cloned and packaged from [here](https://github.com/HennigJue/single-shot-spiral-TSE). |
| TSE | <b>T</b>urbo <b>S</b>pin <b>E</b>cho | Cloned and packaged from [here](https://pulseq.github.io/writeTSE.html). |
| UTE | <b>U</b>ltra-short <b>TE</b> | Cloned and packaged from [here](https://pulseq.github.io/writeUTE_rs.html). |
