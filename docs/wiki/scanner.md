If the variable `pulseq_scanner` is not specified in your sequence creation script before `pulseq_init`, your default scanner will be selected. 

!!! info "Default scanner"
    The default scanner is set when you first run `install_OpenMRF.m`. It can be changed later by modifying the corresponding field in `user_specifications/user_definitions/pulseq_user_definitions.csv`.  

You can compile a sequence for a scanner that is not set as your default by explicitly setting the variable `pulseq_scanner` to the name of an alternative scanner. See the following example. Assume the default scanner is set to `'Siemens_Sola_1,5T_MIITT'`. 

=== "Use default scanner"
    ```matlab title="pulseq_mrf.m" linenums="1"

    %% init pulseq
    clear
    seq_name = 'mrf_';

    % optional flags
    flag_backup = 0; % 0: off,  1: only backup
    flag_report = 0; % 0: off,  1: only timings, 2: full report (slow)
    flag_pns    = 0; % 0: off,  1: simulate PNS stimulation
    flag_sound  = 0; % 0: off,  1: simulate gradient sound
    flag_mrf    = 0; % 0: off,  1: simulate sequence via MRF toolbox

    % init system, seq object and load pulseq user information
    pulseq_init();

    ```
    This will set the system specifications to the values defined in `user_specifications/system_definitions/Siemens_Sola_1,5T_MIITT.csv`.

=== "Use alternative scanner"
    ```matlab title="pulseq_mrf.m" linenums="1"

    %% init pulseq
    clear
    seq_name = 'mrf_';

    % optional flags
    flag_backup = 0; % 0: off,  1: only backup
    flag_report = 0; % 0: off,  1: only timings, 2: full report (slow)
    flag_pns    = 0; % 0: off,  1: simulate PNS stimulation
    flag_sound  = 0; % 0: off,  1: simulate gradient sound
    flag_mrf    = 0; % 0: off,  1: simulate sequence via MRF toolbox

    pulseq_scanner = 'Siemens_FreeMax_0,55T_MIITT';

    % init system, seq object and load pulseq user information
    pulseq_init();

    ```
    This will set the system specifications to the values defined in `user_specifications/system_definitions/Siemens_FreeMax_0,55T_MIITT.csv`.

!!! danger "Scanner selection"
    Never run a sequence on a scanner that it was not compiled for. While your scanner should usually detect if a sequence exceeds certain limits, this can't be guaranteed, and you risk damaging your system. 