The dictionary used in the creation of parametric maps from acquired data is calculated based on the used .seq file. This ensures that the dictionary accurately reflects the events played out in the acquisition without requiring additional user input. 

### Simulation modes
The user can decide between two versions of dictionary creation: `'BLOCH'` and `'EPG'`. EPG is an efficient implementation without corrections for slice profile etc. and is recommended only for fast reconstructions/sanity checks, when accuracy is not the primary objective. In most cases, it is recommended to set `sim_mode` to `'BLOCH'`

!!! info 
    Once calculated, MRF dictionaries are stored in a compressed form under `include_pre_sim_library/compressed_dictionaries`. Whenever you are running an MRF reconstruction, the dictionary creation function will first check whether this exact same dictionary has been created before. If it has, it will be loaded, which substantially accelerates the reconstruction process. 

### Compression Energy
For more efficient computations, the dictionary will be compressed onto a temporal subspace[^1]. The parameter `comp_energy` defines the level of compression - the default value of 0.9999 retains 99.99% of the dictionary's energy. 

### B1+ Correction
Especially at field strengths of 3T and higher, we recommend including B1+ deviations as an additional parameter. This will increase the size of the dictionary and slow down computations but ensure more accurate results. The B1+ range can be fairly coarse, as seen in the example below (using the `'BLOCH'` simulator). 

!!! warning "Confounding effects of B1+ and T2"
    When using MRF sequences that do not feature T2 preparations, the effects of T2 relaxation and B1+ inhomogeneities can not reliably be separated - in this case, do not include B1+ as an additional dictionary parameter. Generally, we recommend using sequences that are interleaved with multiple inversions and T2 preparations.  

=== "Dictionary creation without B1+ correction"
    ```matlab
    sim_mode    = 'BLOCH'; % 'BLOCH' or 'EPG'
    comp_energy = 0.9999;  % 0 for uncompressed dictionary
    softDelay = []; % only for cMRF -> twix_obj.hdr.Meas.adFree(7...) * 1e-3

    P.T1.range = [0.01,  4]; P.T1.factor = 1.025;
    P.T2.range = [0.001, 3]; P.T2.factor = 1.025;
    P = MRF_get_param_dict(P, {'T2<T1'});
    look_up       = [P.T1, P.T2];
    look_up_names = {'T1', 'T2'};

    [SEQ, SIM] = MRF_read_seq_file( seq_path, ...                     % path of the original .seq file which was measured
                                twix_obj.hdr.Meas.lFrequency, ...     % f0; used for actual frequency offsets in fat suppression, CEST or WASABI modules
                                twix_obj.image.timestamp*0.0025, ...  % adc times stamps on a 2.5ms raster; used for correction of trigger delays
                                softDelay,...                         % soft delay input; used for correction of sequence timings
                                [], ...                               % kz partitions for 3D MRF; used to eliminate unnecessary repetitions
                                [], ...                               % echo mode; default: 'spiral_out'
                                1e-6, ...                             % raster time for the simulation 
                                0);                                   % flag_plot

    NIso = 1000; % number of isochromats
    sfac = 2;    % factor for out-of-slice simulation
    z    = linspace(-1/2, 1/2, NIso)' *sfac *PULSEQ.FOV.dz;
    SIM  = MRF_sim_pre(SIM, P, z, 'BLOCH', 1, 0); 
    DICT = MRF_sim_BLOCH(SIM, P, z, [], comp_energy);
    ```

=== "Dictionary creation with B1+ correction"
    ```matlab
    sim_mode    = 'BLOCH'; % 'BLOCH' or 'EPG'
    comp_energy = 0.9999;  % 0 for uncompressed dictionary
    softDelay = []; % only for cMRF -> twix_obj.hdr.Meas.adFree(7...) * 1e-3

    P.T1.range  = [0.01,  4]; P.T1.factor = 1.025;
    P.T2.range  = [0.001, 3]; P.T2.factor = 1.025;
    P.db1.range = [0.8, 1.2]; P.db1.step  = 0.025;
    P = MRF_get_param_dict(P, {'T2<T1'});
    look_up       = [P.T1, P.T2, P.db1];
    look_up_names = {'T1', 'T2', 'db1'};

    [SEQ, SIM] = MRF_read_seq_file( seq_path, ...                     % path of the original .seq file which was measured
                                twix_obj.hdr.Meas.lFrequency, ...     % f0; used for actual frequency offsets in fat suppression, CEST or WASABI modules
                                twix_obj.image.timestamp*0.0025, ...  % adc times stamps on a 2.5ms raster; used for correction of trigger delays
                                softDelay,...                         % soft delay input; used for correction of sequence timings
                                [], ...                               % kz partitions for 3D MRF; used to eliminate unnecessary repetitions
                                [], ...                               % echo mode; default: 'spiral_out'
                                1e-6, ...                             % raster time for the simulation 
                                0);                                   % flag_plot

    NIso = 1000; % number of isochromats
    sfac = 2;    % factor for out-of-slice simulation
    z    = linspace(-1/2, 1/2, NIso)' *sfac *PULSEQ.FOV.dz;
    SIM  = MRF_sim_pre(SIM, P, z, 'BLOCH', 1, 0); 
    DICT = MRF_sim_BLOCH(SIM, P, z, [], comp_energy);
    ```


[^1]: McGivney DF, Pierre E, Ma D, Jiang Y, Saybasili H, Gulani V, Griswold MA. SVD compression for magnetic resonance fingerprinting in the time domain. IEEE Trans Med Imaging. 2014 Dec;33(12):2311-22. doi: 10.1109/TMI.2014.2337321. 