# Cardiac Triggering

Cardiac triggering is used for cMRF sequences. You can find the trigger parameters and timing settings in the following Matlab sections.

---

### General Timing Adjustments
Since MRF sequences typically use multiple preparations with different durations, OpenMRF automatically calculates filling delays for each segment using the `MRF_adjust_segment_delays()` function. These delays are stored in `MRF.delay_dynamic`.

```matlab
% adjust dynamic segment delays
MRF_adjust_segment_delays();
```

---

### Trigger and Soft Delay

If `MRF.mode_trig` is enabled, the trigger object `TRIG_IN` is calculated and added to the beginning of every MRF segment. Note that the trigger must also be activated on the scanner. For Siemens systems, this can be done in the physio menu.

Another important concept is the soft delay. For cardiac scans, we typically want to shift the acquisition window to diastole. `MRF.delay_soft` defines a variable delay object within the Pulseq framework, which can be modified via the scanner GUI.

To ensure that the acquisition is completed before the next R-wave, enter the end of your desired acquisition window in the GUI, including a safety buffer. OpenMRF then automatically calculates `MRF.delay_soft` to match the desired timing.

```matlab
% Trigger Mode
% on:  Cardiac
% off: Abdominal or Phantom
MRF.mode_trig = 'on';

% calc fixed segment timings
MRF.acq_duration      = sum(SPI.TR(1:MRF.nr));
MRF.prep_acq_duration = MRF.prep_max + MRF.acq_duration;

if strcmp(MRF.mode_trig, 'on')
    MRF.delay_soft = mr.makeSoftDelay(0, 'acq_end', 'offset', -MRF.prep_acq_duration, 'factor', 1); % block_duration [s] = offset [s] + input [s] / factor
    TRIG_IN = mr.makeTrigger('physio1', 'system', system, 'delay', 10e-6, 'duration', 10e-3); % Input Trigger
else
    MRF.seg_duration = 1000 *1e-3; % [s] adjust segment duration
    MRF.delay_soft   = mr.makeDelay( round((MRF.seg_duration-MRF.prep_acq_duration)/system.gradRasterTime)*system.gradRasterTime ); % fixed delay
end
```

!!! example "Example"
    **Goal:** diastolic acquisition window:

    - Average RR interval: 1000 ms  
    - Acquisition duration: 250 ms (`MRF.acq_duration`)  
    - Longest preparation: 350 ms (`MRF.prep_max`)  
    - Total duration: 600 ms (`MRF.prep_acq_duration`)  

    To account for RR variations, you enter 900 ms in the GUI.  
    This results in a final delay of:  
    **900 ms − 600 ms = 300 ms**

!!! warning "Short RRs"
    If a patient’s RR interval is too short, the soft delay will be clipped to zero. This may lead to acquisition in the wrong cardiac phase and can cause missed triggers. Always verify sequence timing before in vivo scans and include a sufficient safety buffer.

!!! tip "Phantom tests or abdominal MRF"
    If `MRF.mode_trig` is disabled, the sequence can be used for phantom validation or abdominal imaging. In this case, the user defines a fixed segment duration (`MRF.seg_duration`, e.g. 1 s), and `MRF.delay_soft` becomes a conventional fixed delay.

!!! tip "Disable filling delays"
    For some MRF sequences, it may be useful to have zero delays between all segments. This can be achieved by disabling the trigger and overwriting all dynamic and soft delays:
    ```matlab
    MRF.mode_trig = 'off';
    for ii=1:MRF.n_segm
        MRF.delay_dynamic(ii, 1) = mr.makeDelay(1e-5);
    end
    MRF.delay_soft = mr.makeDelay(1e-5); % minimize delays
    ```

---

### Patient-Specific Dictionaries

OpenMRF calculates the dictionary directly from the `.seq` file. However, the `.seq` file does not contain:

- the user-defined soft delay from the scanner GUI  
- patient-specific trigger waiting times  

Therefore, a patient-specific dictionary must be calculated. The soft delay and trigger information must be passed to the dictionary simulation.

- The **soft delay must be recorded manually** by the user  
- The **trigger waiting times** can be derived from ADC timestamps  

This is handled in the `mrf_reco()` pipeline via `MRF_read_seq_file()`:

```matlab
[~, SIM] = MRF_read_seq_file( params_dict.seq_path, ...
                              params_dict.f0, ...
                              params_dict.time_stamps, ...  % adc time stamps; used for correction of trigger delays
                              params_dict.soft_delays,...   % soft delay input; used for correction of sequence timings
                              params_dict.flag_kz, ...
                              params_dict.echo_mode, ...
                              PULSEQ.SPI.adcNPad,...
                              PULSEQ.SPI.adc.dwell,...
                              1e-6, ... 
                              0);
```

Please refer to the example reconstruction script  
`main_sequences/fingerprinting/reco_mrf.m`  
and enter the soft delay in the following section:

```matlab
%% parameters: dictionary simulation
params_dict.seq_path    = [];
params_dict.sim_mode    = 'BLOCH';
params_dict.comp_energy = 0.9999;
params_dict.N_iso       = 1000;
params_dict.s_fac       = 2;
params_dict.f0          = [];
params_dict.time_stamps = study_info.time_stamps;  % adc time stamps: required for correction of trigger delays in cMRF
params_dict.soft_delays = ???;                     % soft delay user input: required for correction of the acquisition window in cMRF
params_dict.flag_kz     = [];
params_dict.echo_mode   = [];
```
