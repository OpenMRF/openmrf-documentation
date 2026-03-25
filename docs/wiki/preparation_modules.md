OpenMRF is a modular framework. All magnetization preparation and readout modules are implemented as Matlab structs and follow a similar structure. The following information provides an overview of all magnetization preparations that are currently available in OpenMRF. The source code can be found in `include_pulseq_toolbox/src_preparations`.

### Overview
| Module Name | Long Name | Notes |
|-|-|-|
| ADIASL | <b>ADIA</b>batic <b>S</b>pin <b>L</b>ock | Defines an adiabatic spin-lock module for T1\(\rho\) preparation |
| CEST | <b>C</b>hemical <b>E</b>xchange <b>S</b>aturation <b>T</b>ransfer | Work in progress. Currently, you can encode CEST contrast; however, the dictionary calculation needs an update for Bloch-McConnell simulation. |
| CRUSH | <b>CRUSH</b>er | Crusher gradients are necessary for all magnetization preparations. |
| FAT | <b>FAT</b> Suppression | Fat suppression is used for abdominal and cardiac MRF sequences. |
| INV | <b>INV</b>ersion | Inversion recovery preparation module. |
| MLEV | <b>M</b>alcolm <b>LEV</b>itt | Preparation module based on MLEV phase cycling; can be used as an alternative for robust T2 encoding or for encoding T2\(\rho\) contrast. |
| SAT | <b>SAT</b>uration | Saturation recovery preparation module. |
| SL | <b>S</b>pin <b>L</b>ock | Defines a continuous-wave spin-lock module for T1\(\rho\) preparation. |
| T2 | <b>T2</b> preparation | Conventional T2 preparation module. |

## The `..._init()` function
Before magnetization preparations can be added to a sequence, all preparation modules have to be initialized. The first step is to create the struct (e.g. INV) and choose your parameters (e.g. `INV.tExc = 0.01`). Second, the respective `..._init()` function is called, with the respective struct as the first input argument. You do not have to define all parameters of the module yourself - optional parameters have default values that will be assigned by the `..._init()` function unless specified explicitly. When in doubt, just look at the source code of the module in question.

!!! info "Multiple instances of the same preparation module"
    If you are planning to add multiple instances of the same magnetization preparation module with different preparation times (as often done in cardiac and abdominal MRF), you can initialize them all at once by making the respective field (e.g. inversion times) an array instead of a single value. See the examples below.

### Examples
=== "Inversion Recovery"
    The inversion recovery module uses, by default, a hyperbolic sechans inversion pulse calculated via sigpy. Optionally, the inversion recovery times can be an array.
    ```matlab
    INV.rf_type      = 'HYPSEC_inversion';
    INV.tExc         = 10 *1e-3;  % [s]  hypsech pulse duration
    INV.beta         = 700;       % [Hz] maximum rf peak amplitude
    INV.mu           = 4.9;       % [ ]  determines amplitude of frequency sweep
    INV.inv_rec_time = [15 75 150 250] *1e-3;
    INV = INV_init(INV, FOV, system);
    ```
=== "Saturation Recovery"
    The saturation recovery preparation uses, by default, a B1-Insensitive-Rotation pulse calculated via sigpy. Optionally, the saturation recovery times can be an array.
    ```matlab
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
=== "T2 Preparation"
    The T2 preparation uses, by default, two B1-Insensitive-Rotation pulses calculated via sigpy and a two-composite-refocusing-block pulse scheme. Optionally, the T2 preparation times can be an array.
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
=== "Spin-Lock Preparation"
    The spin-lock preparation can be used for T1\(\rho\), T2\(\rho\), or T2. By default, the [Balanced Spin-Lock preparation](https://doi.org/10.1002/mrm.28585) is used. The default excitation and refocusing pulses are adiabatic half-passage pulses. Optionally, spin-lock times and amplitudes can be arrays.
    ```matlab
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
=== "MLEV Preparation"
    The MLEV preparation can be used for T2 or T2\(\rho\) preparation. The original idea of MLEV phase cycling was already published in [1981](https://doi.org/10.1016/0022‐2364(81)90082‐2). The concept of this implementation is explained [here](https://doi.org/10.1002/nbm.5199). The preparation uses, by default, adiabatic half passages for excitation. Refocusing is done using multiple composite block pulses, which have a symmetric phase-cycling scheme (e.g. + - - +). The number of refocusing pulses is always a multiple of 4. The interpulse delay is the delay between consecutive block pulses and defines the T2 weighting. However, MLEV preparations use many refocusing pulses, and during these pulses the magnetization will experience additional T2\(\rho\) weighting. If the interpulse delay is set to 0, the preparation will encode pure T2\(\rho\) contrast. The final preparation times are stored in `MLEV.t2_prep_times` and `MLEV.t2p_prep_times`. These are calculated in the init function and cannot be set directly. Choosing `MLEV.n_mlev` and `MLEV.t_inter` defines the final contrast weighting. `MLEV.fSL` defines the amplitude of the composite refocusing pulses and is equivalent to a spin-lock amplitude (e.g. 250 Hz leads to a 4 ms composite pulse).
    ```matlab
    MLEV.n_mlev   = [1 2 4 8];       % number of MLEV4 preps, -> MLEV4 MLEV8 MLEV16 MLEV32
    MLEV.fSL      = 250;             % [Hz] eff spin-lock field strength
    MLEV.t_inter  = 1 *1e-5;         % [s]  inter pulse delay for T2 preparation
    MLEV.exc_mode = 'adiabatic_AHP'; % 'adiabatic_BIR4' or 'adiabatic_AHP'
    MLEV = MLEV_init(MLEV, FOV, system);
    ```
=== "Adiabatic Spin-Lock"
    The adiabatic spin-lock preparation can be used to encode a contrast that is similar to T1\(\rho\); however, the value of the relaxation time will be considerably higher. Adiabatic spin-locking is currently under active research and might be an alternative to the conventional continuous-wave approach, since it improves robustness to field inhomogeneities. We recommend using a multiple of 4, since the preparation uses an MLEV-like phase-cycling pattern. Even numbers will generate magnetization along +z, and odd numbers will generate an inversion to -z. In the dictionary calculation, a constant adiabatic relaxation time is assumed.
    ```matlab
    ADIASL.N_HS  = [4 8 12 16]'; % number of inversion pulses
    ADIASL.tau   = 0.015;        % [s] duration of inversion pulse
    ADIASL.f1max = 500;          % [Hz] peak amplitude of inversion pulse
    ADIASL.beta  = 5;            % [ ] shape parameter
    ADIASL.dfmax = 200;          % [ ] shape parameter
    ADIASL       = ADIASL_init(ADIASL, FOV, system);
    ```

## The `..._add()` function
When building a sequence, instances of a module can be added using the respective `..._add()` function. If your sequence features multiple instances of the same magnetization preparation module, a counter is increased automatically to ensure that each instance is added with the correct preparation time.

!!! tip
    Take a closer look at `main_sequences/fingerprinting/pulseq_mrf.m` and `main_sequences/fingerprinting/pulseq_cmrf.m` to understand the use of magnetization preparation and readout modules.