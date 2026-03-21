OpenMRF is a modular framework. All magnetization preparation and readout modules are implemented as Matlab structs and follow a similar structure. 
!!! tip "Module vs. instance"
    The Matlab structure defines the module. If your sequence features a specific module more than once, you do not need to initialize them individually. See code examples below.

## Overview of module names
### Magnetization preparations
The source code for the following modules can be found in `include_pulseq_toolbox/src_preparations`. 

| Module Name | Long Name | Notes
|-|-|-|
| ADIASL | <u>A</u>diabatic <u>S</u>pin <u>L</u>ock | Defines an adiabatic spin lock module for T1\(\rho\) preparation |
| CEST | <u>C</u>hemical <u>E</u>xchange <u>S</u>aturation <u>T</u>ransfer ||
| CRUSH | <u>Crush</u>er ||
| FAT | <u>Fat</u> suppression | |
| INV | <u>Inv</u>ersion | |
| MLEV | <u>M</u>alcolm <u>Lev</u>itt | Defines a T2\(\rho\) preparation featuring Malcolm Levitt phase cycling. Can also be used as an alternative to the T2 preparation module for robust T2 encoding - more information [here](mlev_params.md).|
| SAT | <u>Sat</u>uration | |
| SL | <u>S</u>pin <u>L</u>ock | Defines a continuous wave spin lock module for T1\(\rho\) preparation. |
| T2 | <u>T2</u> preparation | |

### Readouts
The source code for the following modules can be found in `include_pulseq_toolbox/src_readouts`. 

| Module Name | Long Name | Notes
|-|-|-|
| EPI | <u>E</u>cho <u>P</u>lanar <u>I</u>maging ||
| PRESS | <u>P</u>oint <u>Res</u>olved <u>S</u>pectroscopy ||
| RAD | <u>Rad</u>ial ||
| SPI | <u>Spi</u>ral | Used for spiral and custom non-Cartesian trajectories. More information [here](custom_trajectories.md). |
| SPITSE | <u>Spi</u>ral <u>T</u>urbo <u>S</u>pin <u>E</u>cho ||
| TSE | <u>T</u>urbo <u>S</u>pin <u>E</u>cho | |
| UTE | <u>U</u>ltra short <u>TE</u>||

### Other
The source code for the following modules can be found in `include_pulseq_toolbox/src_misc`.

| Module Name | Long Name | Notes
|-|-|-|
| FOV | <u>F</u>ield <u>o</u>f <u>v</u>iew | Defines the field of view geometry. 
| TRAJ | <u>Traj</u>ectory | Used for [trajectory calibration](trajectory.md). |
| WASABI | <u>Was</u>erstein based <u>a</u>natomical <u>brain</u> <u>I</u>ndex | Used for simultaneous B0 and B1 mapping. |
<!-- | MRF | <u>M</u>agnetic <u>R</u>esonance <u>F</u>ingerprinting | Defines MRF specific parameters, e.g., flip angles and repetition times. | -->

## The `..._init()` function
Before they can be added to a sequence, all magnetization preparations and readouts have to be initialized. 
The first step is to create the structure. Second, the respective `..._init()` function is called, with the respective structure as the first input argument. You don't have to define all parameters of the module yourself - optional parameters have a default value that will be assigned by the `..._init()` function unless specified explicitly. When in doubt, just look at the source code of the module in question.

!!! info "Multiple instances of the same preparation module"
    If you are planning to add multiple instances of the same magnetization preparation module with different preparation times (as often done in cardiac & abdominal MRF), you can initialize them all at once by making the respective field an array instead of a single value. See examples below. 

The following two examples show how advanced magnetization preparations can be easily initialized in OpenMRF:
=== "Hyperbolic Secant Inversion"
    ```matlab
    INV.rf_type      = 'HYPSEC_inversion';
    INV.tExc         = 10 *1e-3;  % [s]  hypsec pulse duration
    INV.beta         = 700;       % [Hz] maximum rf peak amplitude
    INV.mu           = 4.9;       % [ ]  determines amplitude of frequency sweep
    INV.inv_rec_time = [15 75 150 250] *1e-3;
    INV = INV_init(INV, FOV, system);
    ```
=== "Adiabatic BIR4 T2 preparation"
    ```matlab
    T2.exc_mode   = 'adiabatic_BIR4';
    T2.rfc_dur    = 2 *1e-3;   % [s]  duration of composite refocusing pulses
    T2.bir4_tau   = 10 *1e-3;  % [s]  bir4 pulse duration
    T2.bir4_f1    = 640;       % [Hz] maximum rf peak amplitude
    T2.bir4_beta  = 10;        % [ ]  am waveform parameter
    T2.bir4_kappa = atan(10);  % [ ]  fm waveform parameter
    T2.bir4_dw0   = 30000;     % [rad/s] fm waveform scaling
    T2.prep_times = [40 80] * 1e-3;  % [s] inversion times
    T2            = T2_init(T2, FOV, system);
    ```

This example shows how to define and initialize a spiral readout with golden angle interleaves for MRF: 
```matlab
% import params from MRF struct
SPI.nr             = MRF.nr;    % [ ] number of readouts per heart beat
SPI.NR             = MRF.NR;    % [ ] number of repetitions
SPI.mrf_import.TRs = MRF.TRs;   % [s] repetition times
SPI.mrf_import.FAs = MRF.FAs;   % [rad] flip angles

% slice excitation
SPI.exc_mode      = 'sinc';      % 'sinc' or 'sigpy_SLR'
SPI.exc_time      = 0.8 *1e-3;   % [s] excitation time
SPI.exc_tbw       = 2;           % [ ] time bandwidth product
SPI.exc_fa_mode   = 'import';    % 'equal',  'ramped',  'import' 
SPI.lim_gz_slew   = 0.9;         % [ ] reduce stimulation during slice excitation
SPI.lim_reph_slew = 0.9;         % [ ] reduce stimulation during slice rephaser

% gradient spoiling
SPI.spoil_nTwist   = 4;          % [ ] number of 2pi twists in z-direction, 0 for balanced
SPI.spoil_duration = 0.8 *1e-3;  % [s] time for spoiler and rewinder gradients
SPI.lim_spoil_slew = 0.9;        % [ ] reduce stimulation during gradient spoiling

% rf spoiling
SPI.spoil_rf_mode = 'lin';      % rf spoiling mode: 'lin' or 'quad'
SPI.spoil_rf_inc  = 0 *pi/180;  % rf spoiling increment [rad]

% spiral geometry mode
SPI.geo.interleave_mode = 'RoundGoldenAngle';
SPI.geo.traj_mode       = 'vds';

% vds parameters
SPI.Nunique       = 48;            % number of unique projection angles
SPI.deltak        = 1/FOV.fov_xy;  % [1/m] kspace sampling
SPI.kmax          = SPI.deltak * FOV.Nxy/2;
SPI.geo.Nvds      = 24;            % number of vds-spirals for sampling the kspce center
SPI.geo.BW        = 500 *1e3;      % [Hz] bandwidth of spiral acquisition
SPI.geo.Fcoeff    = [1  -0.5];     % [1 0] for archimedean (equal density), [1 -0.5] for logarithmic (variable density)
SPI.geo.grad_lim  = 1/sqrt(3);     % limit of gradient field strength
SPI.geo.slew_lim  = 1/sqrt(3);     % limit of slew rate
SPI.geo.kmax      = SPI.kmax;      % determines resolution
SPI.geo.t_dwell   = 1/SPI.geo.BW;  % [s] dwell time for spiral acquisition

[SPI, ktraj_adc, ktraj_full, ktraj_reco] = SPI_init(SPI, FOV, system, 1);
```

## The `..._add()` function 
When building a sequence, instances of a module can be added using the respective `..._add()` function. If your sequence features multiple instances of the same magnetization preparation module, a counter is increased automatically to ensure that each instance is added with the correct preparation time. 

!!! tip
    Take a closer look at `main_sequences/fingerprinting/pulseq_mrf.m` and `main_sequences/fingerprinting/pulseq_cmrf.m` to understand the use of magnetization preparation and readout modules. 