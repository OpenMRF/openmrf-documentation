---
hide: 
    - navigation
    # - toc
---

A central feature of MRF is the great flexibility in sequence design. While we provide a few [example sequences](https://github.com/OpenMRF/openmrf-publications/tree/main/2026_MRM_OpenMRF) for you to use and reproduce the results presented in the original [OpenMRF publication](https://arxiv.org/abs/2604.22713), we encourage you to modify the sequence creation code and adapt it according to your needs. Importantly, since the dictionary creation is fully automated and based on the .seq file, you do not have to make any changes to the reconstruction pipeline! Most changes to the sequence will automatically be detected by the automated dictionary creation code and produce a dictionary that accurately reflects the experiment run on the scanner.  

We currently provide several different templates for MRF sequences (all to be found in the `main_sequences/fingerprinting` folder of the [`openmrf-core-matlab`](https://github.com/OpenMRF/openmrf-core-matlab) repository). In this tutorial, we will focus on two of the scripts found there (`pulseq_mrf.m` and `pulseq_cmrf_template.m`), explain the code in detail (cell by cell) and lay out potential modifications. 

!!! tip
    This tutorial will focus on how to make basic changes to MRF sequence parameters. However, if you need more in-depth customization, we invite you to explore the codebase by yourself - it's all publicly available and you're invited to create a fork of the repository and make changes wherever you need. 

## IR-FISP MRF type sequences: `pulseq_mrf.m`
Most MRF sequences for brain imaging applications are based on the IR-FISP MRF approach originally introduced by Jiang Y, MRM 2015[^1]. These sequences feature an inversion pulse at the beginning, followed by several hundred RF excitations and fast spiral readouts. `pulseq_mrf.m` allows to recreate and modify this type of sequence. 

### Cell 1: set definitions
This first cell initializes pulseq, defines export/report flags, and selects the scanner profile.

``` matlab
%% init pulseq
clear
seq_name = 'mrf_';

% main flags
flag_backup = 0; % 0: off,  1: only backup,  2: backup and send .seq
flag_report = 0; % 0: off,  1: only timings, 2: full report (slow)
flag_pns    = 0; % 0: off,  1: simulate PNS stimulation

% optional: select scanner
pulseq_scanner = 'Siemens_Sola_1,5T_MIITT';

% optional: select pns sim orientation
% pns_orientation = 'all';

% init system, seq object and load pulseq user information
pulseq_init();
```

The main thing you may want to edit in this cell is the flags (see [here](quickstart.md/#5-compile-a-test-sequence) for more detail) and the scanner selection. 

### Cell 2: FOV geometry
Here, you can set matrix size, FOV, etc. 
``` matlab
%% FOV geometry
FOV.Nxy    = 256;        % [ ] matrix size
FOV.Nz     = 1;          % [ ] numer of "stack-of-spirals", 1 -> 2D
FOV.fov_xy = 256 *1e-3;  % [m] FOV geometry
FOV.dz     = 5   *1e-3;  % [m] slice thickness
FOV_init();
```

### Cell 3: set MRF flip angles and repetition times
This cell defines the MRF signal encoding by setting flip angles and repetition times.

``` matlab
%% params: MRF flipangles and repetition times
MRF.pattern         = 'yun'; % select pattern: e.g. 'yun' or 'cao'
[MRF.FAs, MRF.TRs]  = MRF_get_FAs_TRs(MRF.pattern, 1);
% MRF.FAs(MRF.FAs==0) = 1e-6;
seq_name            = [seq_name MRF.pattern];
MRF.NR              = numel(MRF.FAs);

% or define a custom pattern
% MRF.pattern = 'sin_70';
% MRF.FAs     = MRF_calc_FAs_sin([5, 30, 200; 1, 70, 200; 10, 10, 100]) *pi/180;
% MRF.NR      = numel(MRF.FAs);
% MRF.TRs     = 12 *1e-3 *ones(MRF.NR,1);
% seq_name    = [seq_name MRF.pattern];
```

Here, you have three choices. You can either choose to reproduce flip angles and repetition times as used in previous publications (`MRF.pattern = 'yun';`[^1] or `'MRF.pattern = 'cao';` [^2]), or build your own flip angle and repetition time pattern (using the `MRF_calc_FAs_sin.m` or `MRF_calc_FAs_sin_rand.m` function). Finally, you can also just set `MRF.FAs` and `MRF.TRs` to any array you define yourself - just make sure their sizes match. 

!!! info "Units"
    Throughout this repository, we define variables in standard units where possible. That means that flip angles are defined in radians, repetition times (and all other timings) in seconds. 

!!! warning "TR limitations"
    Depending on hardware limitations, your chosen TR might not be feasible. In that case, you will see a warning in the command window, and the TRs will be set to the smallest feasible value. If you want to minimize the TRs, just initialize them as an array of zeros. 

!!! warning "Zero degree flip angle" 
    In the current implementation (v1.0, July 2026), it is not possible to set any flip angle to zero, this will cause an error. If your desired flip angle train does include a zero degree flip angle, replace it by a near zero value (e.g., 1e-6) to avoid this issue. 

### Cell 4: RF excitation and trajectory parameters
This is one of the most important cells for practical sequence tuning, because it controls readout timing, excitation shape, trajectory geometry and hardware safety margins.

``` matlab
%% params: Spiral Readouts

% basic/import params for MRF
SPI.mode_2D_3D     = '2D';      % '2D', '3D' or '3D_stacked'
SPI.adcBW          = 400 *1e3;  % [Hz] desired receiver bandwidth
SPI.NR             = MRF.NR;    % [ ] number of repetitions
SPI.mrf_import.TRs = MRF.TRs;   % [s] repetition times
SPI.mrf_import.FAs = MRF.FAs;   % [rad] flip angles

% slice excitation
SPI.exc_mode      = 'sinc';      % 'sinc' or 'sigpy_SLR'
SPI.exc_shape     = 'ex';        % only for sigpy: 'st' or 'ex' 
SPI.exc_time      = 2.0 *1e-3;   % [s] excitation time
SPI.exc_tbw       = 6;           % [ ] time bandwidth product
SPI.exc_fa_mode   = 'import';    % 'equal',  'ramped',  'import'  
SPI.reph_duration = 1.0 *1e-3;   % [s] slice rephaser duration

% params: gradient spoiling & rf spoiling
SPI.spoil_nTwist   = 4 * FOV.Nz; % [ ] number of 2pi twist in slice
SPI.spoil_rf_mode  = 'lin';      % 'lin' or 'quad'
SPI.spoil_rf_inc   = 0 *pi/180;  % [rad] rf phase increment
SPI.spoil_duration = 1.0 *1e-3;  % [s] slice spoiler duration

% k-space geometry params
SPI.geo.design        = 'spiral';
SPI.geo.design_fun    = 'hargreaves';
SPI.geo.N_interleaves = 24;
SPI.geo.FOV_coeff     = [1 -0.5];
SPI.geo.Ns            = 1e5;
SPI.geo.ds            = 1e-3;
SPI.geo.flag_rv       = 0;

% k-space projection params
SPI.proj.mode = 'RoundGoldenAngle'; % 'Equal2Pi' 'RoundGoldenAngle'
SPI.proj.Nid  = 48; % number of unique projections

% gradient & slew rate limitation factors
SPI.lim_exc_slew   = 0.9; % slice excitation
SPI.lim_reph_grad  = 0.9; % slice rephasing
SPI.lim_reph_slew  = 0.6; % slice rephasing
SPI.lim_read_grad  = 0.9; % readout
SPI.lim_read_slew  = 0.5; % readout
SPI.lim_spoil_grad = 0.9; % slice spoiling
SPI.lim_spoil_slew = 0.6; % slice spoiling

[SPI, ktraj_ref] = SPI_init(SPI, FOV, system, 1);
MRF.TRs = SPI.TR;
```

In short, this block does two things: (1) it sets all design targets, and (2) it runs `SPI_init` to compute an implementation that actually satisfies your scanner limits. The final implemented TR values are written back into `MRF.TRs`.

There are a number of parameters that you might need to adapt, based on your scanner limitations. For example, you may not be able to run a very high time bandwidth product RF pulse that is also short. 

Another important thing to consider are the gradient and slew rate limits and the end of this cell. The values you set here will be multiplied with the limits defined in your scanner specification .csv file. Depending on your specific scanner setup, they might require some tinkering. Reducing the limits may slow down your sequence, while setting them too high might result in PNS violations that will cause your sequence to not execute. A good way to check this is to set `flag_pns = 1` at the beginning of the script and make sure you stay below ~80-85% in all directions. 

### Cell 5: Inversion preparation
This cell defines the inversion pulse before the MRF train. 

``` matlab
%% params: Inversion
INV.rf_type      = 'HYPSEC_inversion';
INV.tExc         = 10 *1e-3;  % [s]  hypsech pulse duration
INV.beta         = 700;       % [Hz] maximum rf peak amplitude
INV.mu           = 4.9;       % [ ]  determines amplitude of frequency sweep
INV.inv_rec_time = 0.01;      % [s]  inversion recovery time
INV = INV_init(INV, FOV, system);
```

The default inversion pulse is a hyperbolic secant. 

Typical edits:

- Increase or decrease `INV.inv_rec_time` (often referred to as the inversion time). 
- Adjust `INV.tExc`, `INV.beta`, `INV.mu` only if you know your desired adiabatic pulse behavior and hardware constraints.

### Cell 6: Noise pre-scans
Pre-scans are appended before imaging readouts. They are an easy way to improve your SNR at almost no cost. The evaluation will be handled automatically in the reconstruction script, so you don't have to worry about anything here. When in doubt, just leave this block in place. 

``` matlab
%% noise pre-scans
SPI.Nnoise = 16;
SPI_add_prescans();
```

Set `SPI.Nnoise = 0` if you explicitly want to disable them.

### Cell 7: build sequence events
This is where the events are actually added to the sequence object: first inversion, then all spiral readouts.

``` matlab
%% create sequence

% inversion
seq.addTRID('inversion');
INV_add();

% spiral imaging
for loop_NR = 1:SPI.NR
    SPI_add();
end
```

For most use cases, you will not need to modify this logic at all, only the parameter cells above.

### Cells 8 & 9: plotting and export
Finally, the script plots the sequence and calls the exit routine for checks/export:

``` matlab
%% plot sequence diagram
seq.plot()

%% set definitions, check timings/gradients and export/backup files
filepath = [mfilename('fullpath') '.m'];
pulseq_exit();
```

`pulseq_exit()` handles timing/gradient checks and file export according to your flags in cell 1.

Here again, you should usually not need to make edits, with one exception: depending on the length of the MRF sequence, Pulseq's sequence plotting functionality can be pretty demanding and slow. You can address this by specifying the range of the sequence you want plotted. Instead of calling `seq_plot()` with no arguments, use `seq.plot('TimeRange', [0 1])` to plot the first second of the sequence (or adapt the time range to the part that matters to you). 

---


## cMRF type sequences: `pulseq_cmrf_template.m`
`pulseq_cmrf_template.m` is a modular example for cardiac MRF. In contrast to `pulseq_mrf.m`, it organizes the acquisition into **segments**, each segment starting with a selected preparation (or no preparation), followed by readouts. While this kind of sequence is typically used in combination with ECG triggering to ensure the acquisition windows fall int o the same cardiac phase, triggering can also be disabled and replaced with fixed or flexible delays between the acquisition windows - more information below.

!!! warning
    As noted in the template itself, this script is meant as a demonstration of module combination. For realistic cardiac protocols, start from the dedicated examples (`pulseq_cmrf_t1_t2.m` and `pulseq_cmrf_t1_t2_t1p.m`) and adapt those.

### Cell 1: set definitions
This first cell initializes pulseq, defines export/report flags, and selects the scanner profile.

``` matlab
%% init pulseq
clear
seq_name = 'cardiac_mrf';

% main flags
flag_backup = 0; % 0: off,  1: only backup,  2: backup and send .seq
flag_report = 0; % 0: off,  1: only timings, 2: full report (slow)
flag_pns    = 0; % 0: off,  1: simulate PNS stimulation

% optional: select scanner
pulseq_scanner = 'Siemens_Sola_1,5T_MIITT';

% optional: select pns sim orientation
% pns_orientation = 'all';

% init system, seq object and load pulseq user information
pulseq_init();
```

The main thing you may want to edit in this cell is the flags (see [here](quickstart.md/#5-compile-a-test-sequence) for more detail) and the scanner selection. 

### Cell 2: FOV geometry
Cardiac defaults differ from the brain MRF example and use a larger in-plane FOV. Feel free to adapt to your specific needs, but be aware that changing the FOV geometry might influence timings. 

``` matlab
%% FOV geometry
FOV.Nxy    = 192;         % [ ] matrix size
FOV.fov_xy = 320  *1e-3;  % [m] FOV geometry
FOV.dz     = 8   *1e-3;   % [m] slice thickness
FOV_init();
```

### Cell 3: define the segment encoding list
This is one of the most important cMRF design choices. `MRF.enc_list` determines which preparation is used for each segment.

``` matlab
%% params: MRF contrast encoding

% encoding list with contrast preparations for different segments
% 'No_Prep'    ->  use for recovery of longitudinal magnetization
% 'Saturation' ->  use for T1 encoding
% 'Inversion'  ->  use for T1 encoding
% 'T2'         ->  use for T2 encoding
% 'SL'         ->  use for T1p encoding
% 'MLEV'       ->  use for T2 or T2p encoding

MRF.enc_list = {
'Inversion';
'No_Prep';
'T2';
'T2';
'Inversion';
'Saturation';
'SL';
'SL';
'Inversion';
'Saturation';
'No_Prep';
'No_Prep';
'Inversion';
'No_Prep';
'MLEV';
'MLEV'
};

MRF.n_segm = numel(MRF.enc_list);
```

Typical edit: Change order and frequency of preparation types to rebalance sensitivity (for example stronger T1-encoding vs T2/T1p encoding).

### Cell 4: set flip angles and TRs
Within each segment, cMRF uses `MRF.nr` readouts. Total readouts are `MRF.NR = MRF.n_segm * MRF.nr`.

``` matlab
%% params: MRF flipangles and repetition times
MRF.nr     = 48;                         % numer of readouts per hear beat
MRF.NR     = MRF.n_segm * MRF.nr;        % total number of readouts
MRF.TRs    = 0.0 *1e-3 *ones(MRF.NR,1);  % minimize TRs
MRF.FA_min = 4 *pi/180;                  % [rad] minimum flip angle
MRF.FA_max = 15 *pi/180;                 % [rad] minimum flip angle
MRF.FAs    = MRF_calc_FAs_sin_rand(MRF.FA_min, MRF.FA_max, MRF.nr, MRF.n_segm);

% or use constant FAs
% MRF.FAs = 15*pi/180 * ones(MRF.NR,1);

% or import from .mat file
% load('FAs.mat');
% MRF.FAs = FAs;
```

### Cell 5: segment timing mode and trigger behavior
This cell controls how segment timing is handled, especially relevant for cardiac synchronization.

``` matlab
%% params: MRF segment timings and trigger options

MRF.mode_seg = 'trigger'; % 'trigger', 'constant', 'variable'

switch MRF.mode_seg
    case 'trigger'
        % cardiac trigger based mode

    case 'constant'
        MRF.seg_duration = 1000 *1e-3; % [s]

    case 'variable'
        MRF.rec_times = rand(MRF.n_segm, 1); % [s]
end
```

Use cases:

- `trigger`: preferred for cardiac scans synchronized to ECG trigger.
- `constant`: use for controlled phantom tests, ensures fixed delays between segments.
- `variable`: use when you want to prescribe variable delays between segments explicitly.

### Cell 6: spiral readout parameters
This is one of the most important cells for practical sequence tuning, because it controls readout timing, excitation shape, trajectory geometry and hardware safety margins.

``` matlab
%% params: Spiral Readouts

% basic/import params for MRF
SPI.mode_2D_3D     = '2D';      % '2D', '3D' or '3D_stacked'
SPI.adcBW          = 400 *1e3;  % [Hz] desired receiver bandwidth
SPI.NR             = MRF.NR;    % [ ] number of repetitions
SPI.nr             = MRF.nr;    % [ ] numer of readouts per hear beat
SPI.mrf_import.TRs = MRF.TRs;   % [s] repetition times
SPI.mrf_import.FAs = MRF.FAs;   % [rad] flip angles

% slice excitation
SPI.exc_mode      = 'sinc';      % 'sinc' or 'sigpy_SLR'
SPI.exc_shape     = 'ex';        % only for sigpy: 'st' or 'ex' 
SPI.exc_time      = 0.5 *1e-3;   % [s] excitation time
SPI.exc_tbw       = 2;           % [ ] time bandwidth product
SPI.exc_fa_mode   = 'import';    % 'equal',  'ramped',  'import'  
SPI.reph_duration = 0.3 *1e-3;   % [s] slice rephaser duration

% params: gradient spoiling & rf spoiling
SPI.spoil_nTwist   = 4 * FOV.Nz;  % [ ] number of 2pi twist in slice
SPI.spoil_rf_mode  = 'lin';       % 'lin' or 'quad'
SPI.spoil_rf_inc   = 0 *pi/180;   % [rad] rf phase increment
SPI.spoil_duration = 0.5 *1e-3;   % [s] slice spoiler duration

% k-space geometry params
SPI.geo.design        = 'spiral';
SPI.geo.design_fun    = 'hargreaves';
SPI.geo.N_interleaves = 24;
SPI.geo.FOV_coeff     = [1 -0.5];
SPI.geo.Ns            = 1e5;
SPI.geo.ds            = 1e-3;
SPI.geo.flag_rv       = 0;

% k-space projection params
SPI.proj.mode = 'RoundGoldenAngle'; % 'Equal2Pi' 'RoundGoldenAngle'
SPI.proj.Nid  = 48; % number of unique projections

% gradient & slew rate limitation factors
SPI.lim_exc_slew   = 0.95; % slice excitation
SPI.lim_reph_grad  = 0.95; % slice rephasing
SPI.lim_reph_slew  = 0.95; % slice rephasing
SPI.lim_read_grad  = 0.90; % readout
SPI.lim_read_slew  = 0.55; % readout
SPI.lim_spoil_grad = 0.95; % slice spoiling
SPI.lim_spoil_slew = 0.95; % slice spoiling

[SPI, ktraj_ref] = SPI_init(SPI, FOV, system, 1);
MRF.TRs = SPI.TR;
```

In short, this block does two things: (1) it sets all design targets, and (2) it runs `SPI_init` to compute an implementation that actually satisfies your scanner limits. The final implemented TR values are written back into `MRF.TRs`.

There are a number of parameters that you might need to adapt, based on your scanner limitations. For example, you may not be able to run a very high time bandwidth product RF pulse that is also short. 

Another important thing to consider are the gradient and slew rate limits and the end of this cell. The values you set here will be multiplied with the limits defined in your scanner specification .csv file. Depending on your specific scanner setup, they might require some tinkering. Reducing the limits may slow down your sequence, while setting them too high might result in PNS violations that will cause your sequence to not execute. A good way to check this is to set `flag_pns = 1` at the beginning of the script and make sure you stay below ~80-85% in all directions. 

Compared to `pulseq_mrf.m`, this block additionally uses `SPI.nr = MRF.nr` and has different default excitation/spoiling settings tailored to the segmented cardiac setup.

### Cell 7: inversion preparation module
This cell defines the inversion pulse before the MRF train. 

``` matlab
%% params: Inversion
INV.rf_type      = 'HYPSEC_inversion';
INV.tExc         = 10 *1e-3;  % [s]  hypsech pulse duration
INV.beta         = 700;       % [Hz] maximum rf peak amplitude
INV.mu           = 4.9;       % [ ]  determines amplitude of frequency sweep
INV.inv_rec_time = [15 75 150 250] *1e-3;
INV = INV_init(INV, FOV, system);
```

The default inversion pulse is a hyperbolic secant. 

In cMRF, this module is only used in segments where `MRF.enc_list` contains `'Inversion'`.

Typical edits:

- Adapt `INV.inv_rec_time` (often referred to as the inversion times). In cMRF, the length of this array has to correspond to the number of inversions in the MRF encoding list.
- Adjust `INV.tExc`, `INV.beta`, `INV.mu` only if you know your desired adiabatic pulse behavior and hardware constraints.

### Cell 8: saturation preparation module

``` matlab
%% params: Saturation
SAT.mode         = 'on';
SAT.rf_type      = 'adiabatic_BIR4';
SAT.bir4_tau     = 10 *1e-3;  % [s]  bir4 pulse duration
SAT.bir4_f1      = 640;       % [Hz] maximum rf peak amplitude
SAT.bir4_beta    = 10;        % [ ]  am waveform parameter
SAT.bir4_kappa   = atan(10);  % [ ]  fm waveform parameter
SAT.bir4_dw0     = 30000;     % [rad/s] fm waveform scaling
SAT.sat_rec_time = [100 200] *1e-3; % [s] saturation times
SAT = SAT_init(SAT, FOV, system);
```

This module is used in `'Saturation'` segments.

### Cell 9: T2 preparation module

``` matlab
%% params: T2 preparation
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

`T2.prep_times` is a key parameter to tune the T2 encoding and the length of this array has to correspond to the number of T2 preps in the MRF encoding list.

### Cell 10: spin-lock preparation module
This cell shows how to add continuous wave spin locking preparations to the MRF sequence. 

``` matlab
%% params: Spin-Lock

% spin-lock pulses
SL.relax_type = {'T1p'};         % T1p or T2p or T2
SL.seq_type   = {'BSL'};         % BSL or CSL or RESL
SL.tSL        = [40 80] *1e-3;   % [s]  SL time
SL.fSL        = [200 200];       % [Hz] SL amplitude

% excitation pulses
SL.exc_mode  = 'adiabatic_AHP';  % 'adiabatic_AHP', 'sinc', 'sigpy_SLR' or 'bp'
SL.exc_time  = 3.0 *1e-3;        % [s] excitation time
SL.adia_wmax = 600 * 2*pi;       % [rad/s] amplitude of adiabatic pulse

% refocusing pulses
SL.rfc_mode = 'bp';              % 'bp', 'sinc', 'sigpy_SLR' or 'comp'
SL.rfc_time = 1.0 *1e-3;         % [s] refocusing time

SL = SL_init(SL, FOV, system);
```

For T1rho mapping experiments, the most common edits are `SL.tSL` and `SL.fSL`. The length of these arrays has to correspond to the number of SL T1rho preps in the MRF encoding list

### Cell 11: MLEV preparation module

``` matlab
%% params: MLEV T2p preparation
MLEV.n_mlev   = [1 2];           % number of MLEV4 preps
MLEV.fSL      = 250;             % [Hz] eff spin-lock field strength
MLEV.t_inter  = 1 *1e-5;         % [s]  inter pulse delay for T2 preparation
MLEV.exc_mode = 'adiabatic_AHP'; % 'adiabatic_BIR4' or 'adiabatic_AHP'
MLEV = MLEV_init(MLEV, FOV, system);
```

MLEV settings matter only for segments labeled `'MLEV'` in `MRF.enc_list`. The prep times for each specific occurence result from the respecive entry of MLEV_.n_mlev times MLEV.t_inter times 4. In the above example, the two MLEV preparations will have total preparation times of 40 and 80 milliseconds. 

### Cell 12: fat saturation module

``` matlab
%% params: Fat Saturation
FAT.mode = 'on';
FAT = FAT_init(FAT, FOV, system);
```

If fat suppression is not desired, set `FAT.mode = 'off';`. If active, each acquisition window will be preceded by a fat saturation. 

### Cell 13: validate encoding and adjust segment delays
Before creating events, the script checks consistency and computes required timing adjustments:

``` matlab
%% check MRF encoding params and adjust segment delays
MRF_check_enc_list();
MRF_adjust_segment_delays();
```

This cell does not need to be modified. 

### Cell 14: noise pre-scans
Pre-scans are appended before imaging readouts. They are an easy way to improve your SNR at almost no cost. The evaluation will be handled automatically in the reconstruction script, so you don't have to worry about anything here. When in doubt, just leave this block in place. 

``` matlab
%% noise pre-scans
SPI.Nnoise = 16;
SPI_add_prescans();
```

Set `SPI.Nnoise = 0` if you explicitly want to disable them.

### Cell 15: generate the sequence

``` matlab
%% create sequence
MRF_add_segments();
```

`MRF_add_segments()` loops over `MRF.enc_list`, inserts the selected preparation module per segment, and appends the corresponding readout block.

### Cells 16 & 17: plotting and export
Finally, the script plots the sequence and calls the exit routine for checks/export:

``` matlab
%% plot sequence diagram
seq.plot();

%% set definitions, check timings/gradients and export/backup files
filepath = [mfilename('fullpath') '.m'];
pulseq_exit();
```

`pulseq_exit()` handles timing/gradient checks and file export according to your flags in cell 1.

Here again, you should usually not need to make edits, with one exception: depending on the length of the MRF sequence, Pulseq's sequence plotting functionality can be pretty demanding and slow. You can address this by specifying the range of the sequence you want plotted. Instead of calling `seq_plot()` with no arguments, use `seq.plot('TimeRange', [0 1])` to plot the first second of the sequence (or adapt the time range to the part that matters to you). 

---

!!! tip "Safe execution of Pulseq sequences"
    Before running Pulseq sequences on your scanner, we recommend compiling the sequence with `flag_report = 2;` and `flag_pns = 1;`. This will allow you to double check PNS and gradient/slew rate limits. Another recommended step is to simulate the sequence in your scanners vendor specific environment. 

[^1]: Jiang Y, Ma D, Seiberlich N, Gulani V, Griswold MA. MR fingerprinting using fast imaging with steady state precession (FISP) with spiral readout. Magn Reson Med. 2015 Dec;74(6):1621-31. doi: 10.1002/mrm.25559. 
[^2]: Cao X, Liao C, Iyer SS, Wang Z, Zhou Z, Dai E, Liberman G, Dong Z, Gong T, He H, Zhong J, Bilgic B, Setsompop K. Optimized multi-axis spiral projection MR fingerprinting with subspace reconstruction for rapid whole-brain high-isotropic-resolution quantitative imaging. Magn Reson Med. 2022 Jul;88(1):133-150. doi: 10.1002/mrm.29194.