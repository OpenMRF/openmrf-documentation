---
hide: 
    - navigation
---

On this page, you'll find quick fixes for the most common problems you may encounter. 

!!! tip "Your feedback matters!" 
    Ran into an issue that's not listed here? [Let us know!](mailto:maximilian.gram@uni-wuerzburg.de,tomgr@med.umich.edu)


## "Error using mr.makeArbitraryGrad"
This error will be raised in Matlab while trying to compile a sequence if you are defining gradients that are not compatible with the selected system specifications. There are two variants that can appear: "Slew rate violation" or "Requested area is too large for this gradient". In both cases, either reduce the gradient moment (for example, fewer twists for a spoiler) or increase the gradient duration.  


## Stimulation limit exceeded
This may be one of the most common issues you'll run into. Even if you double-checked all parameters, the sequence compiled without errors, and the PNS simulation predicted moderate values, your scanner might not execute your sequence (or abort halfway through) and display a warning that the stimulation limit was exceeded. Luckily for you, in most cases this is also an easy fix. You'll need to reduce the slew rate. The first step would be to analyze the PNS simulation figure in detail. By comparing the peaks to the sequence diagram, you should be able to identify which events cause the highest PNS. For the specific module, you can then reduce the slew rate. Per default, most modules already reduce the system's nominal slew rate maximum by a factor of `sqrt(3)`. 

!!! info "PNS calculation" 
    You will only be able to create the estimated PNS diagram for your sequence if you have access to the .asc file for your scanner. If not, you'll have to find the problematic gradient by trial and error.

If you're unsure how to reduce the gradient or slew rate limits for a specific event, just open the corresponding `..._init` function. In there, you will find a passage that defines the default value. Here's an example for the T2 preparation crushers:
```matlab title="T2_init.m"
function T2 = T2_init(T2, FOV, system)
...
    if ~isfield(T2, 'crush_lim_grad')
        T2.crush_lim_grad = 1/sqrt(3); % reduce crusher gradient amplitude from nominal limit
    end    
    if ~isfield(T2, 'crush_lim_slew')
        T2.crush_lim_slew = 1/sqrt(3); % reduce crusher gradient slew rate from nominal limit to avoid stimulation
    end
...
```

Here's an example showing how to initialize the T2 preparation using the default limits versus using explicitly reduced slew rate: 

=== "Use default values"
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
=== "Manually reduce slew rate for crushers"
    ```matlab
    T2.exc_mode   = 'adiabatic_BIR4';
    T2.rfc_dur    = 2 *1e-3;   % [s]  duration of composite refocusing pulses
    T2.bir4_tau   = 10 *1e-3;  % [s]  bir4 pulse duration
    T2.bir4_f1    = 640;       % [Hz] maximum rf peak amplitude
    T2.bir4_beta  = 10;        % [ ]  am waveform parameter
    T2.bir4_kappa = atan(10);  % [ ]  fm waveform parameter
    T2.bir4_dw0   = 30000;     % [rad/s] fm waveform scaling
    T2.prep_times = [40 80] * 1e-3;  % [s] inversion times
    T2.crush_lim_slew = 0.5;
    T2            = T2_init(T2, FOV, system);
    ```

## 