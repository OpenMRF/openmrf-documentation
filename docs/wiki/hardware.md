### Create a `.csv` File

In OpenMRF, hardware specifications are stored in `.csv` files. These files are located in `user_specifications/system_definitions`. The content of a `.csv` file looks like this:

```text
B0;1.5
gamma;4.257747851783256e+07
maxGrad;43
gradUnit;mT/m
maxSlew;180
slewUnit;T/m/s
maxB1;20
rfDeadTime;1e-4
rfRingdownTime;1e-4
adcDeadTime;1e-5
adcRasterTime;1e-7
rfRasterTime;1e-6
gradRasterTime;1e-5
blockDurationRaster;1e-5
ascfile;Sola_MP_GPA_K2368_2250V_950A_GC04XQ.asc
```

Please make sure that the hardware specifications are set correctly. If your lab has `.asc` files that can be used for PNS simulation, copy the corresponding file to `user_specifications/system_definitions/asc_files` and enter its name in the last line of the `.csv` file.

### Select a Scanner

All OpenMRF sequences start with the same initialization section, which calls the `pulseq_init()` function. You can specify the scanner using the variable `pulseq_scanner`. If `pulseq_scanner` is not set, OpenMRF uses the default scanner stored in the user definitions.

=== "Use custom scanner"
    ```matlab

    % main flags
    flag_backup = 0; % 0: off,  1: only backup,  2: backup and send .seq
    flag_report = 0; % 0: off,  1: only timings, 2: full report (slow)
    flag_pns    = 0; % 0: off,  1: simulate PNS stimulation

    % optional: select scanner
    pulseq_scanner = 'Siemens_Sola_1,5T_MIITT';

    % init system, seq object and load pulseq user information
    pulseq_init();

    ```

    This sets the system specifications to the values defined in `user_specifications/system_definitions/Siemens_Sola_1,5T_MIITT.csv`.

=== "Use default scanner"
    ```matlab

    % main flags
    flag_backup = 0; % 0: off,  1: only backup,  2: backup and send .seq
    flag_report = 0; % 0: off,  1: only timings, 2: full report (slow)
    flag_pns    = 0; % 0: off,  1: simulate PNS stimulation

    % init system, seq object and load pulseq user information
    pulseq_init();

    ```

    This sets the system specifications to the values defined in `user_specifications/user_definitions/pulseq_user_definitions.csv`.

!!! info "Default scanner"
    The default scanner is set when you first run `install_OpenMRF.m`. It can be changed later by modifying the corresponding field in `user_specifications/user_definitions/pulseq_user_definitions.csv`.

!!! danger "Scanner selection"
    Never run a sequence on a scanner for which it was not compiled. Although the scanner will usually detect whether a sequence exceeds certain limits, this cannot be guaranteed, and you risk damaging the system.
