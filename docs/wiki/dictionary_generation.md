# Dictionary Calculation

OpenMRF calculates the MRF dictionary directly from the used `.seq` file. This is one of the central design concepts of the framework: the simulator does not rely on a manually reimplemented sequence model, but instead parses the actual Pulseq sequence file and simulates exactly the events that are played out during the acquisition.

This ensures a highly general and flexible approach to dictionary creation. New MRF sequences can therefore be implemented and tested without the need to develop a dedicated simulator for each sequence variant.

---

## 1. Global Parameters for Dictionary Simulation

The main settings for dictionary generation are defined in the example reconstruction script:

```text
main_sequences/fingerprinting/reco_mrf.m
```

The following section controls the global dictionary simulation settings:

```matlab
%% parameters: dictionary simulation
params_dict.seq_path    = [];       % full path of .seq file; [] for automatic search in user backups
params_dict.sim_mode    = 'BLOCH';  % 'BLOCH' or 'EPG'
params_dict.comp_energy = 0.9999;   % svd compression energy: 0 for uncompressed dictionary
params_dict.N_iso       = 1000;     % number of isochromats for bloch simulation
params_dict.s_fac       = 2;        % factor for out-of-slice simulation
params_dict.f0          = [];       % larmor frequency f0: required to simulate rf pulses with ppm freq offsets
params_dict.time_stamps = study_info.time_stamps; % adc times stamps: required for correction of trigger delays in cMRF
params_dict.soft_delays = [];       % soft delay user input: required for correction of the acq window in cMRF
params_dict.flag_kz     = [];       % find kz partitions for stacked 3D MRF -> eliminate unnecessary partitions
params_dict.echo_mode   = [];       % echo mode; default: 'spiral_out'
```

### `params_dict.seq_path`
Defines the full path to the `.seq` file. If set to `[]`, OpenMRF automatically searches for the sequence in the user backup folders.

### `params_dict.sim_mode`
Defines the simulation mode:
- `'BLOCH'`: full Bloch simulation (default)
- `'EPG'`: faster EPG-based simulation

The `'BLOCH'` mode is the recommended default, as it accurately models slice profile effects and supports additional physical processes such as T2ρ relaxation, which are not supported by the EPG framework.  

The `'EPG'` mode can be useful for simple sequences or quick sanity checks, but it does not account for several important effects. For more complex sequences—especially cardiac MRF—we strongly recommend using the `'BLOCH'` simulation.

### `params_dict.comp_energy`
Defines the SVD compression energy of the dictionary. The default value `0.9999` retains 99.99% of the temporal signal energy. If set to `0`, the dictionary is kept uncompressed.

!!! info
    OpenMRF stores precomputed dictionaries in compressed form under `include_pre_sim_library/compressed_dictionaries`. Whenever a reconstruction is started, OpenMRF first checks whether an identical dictionary has already been calculated. If so, the dictionary is loaded directly, which can substantially accelerate the reconstruction.

### `params_dict.N_iso`
Defines the number of isochromats used for Bloch simulation. This parameter is relevant only for `'BLOCH'` mode.

### `params_dict.s_fac`
Defines the factor for out-of-slice simulation. This is used to model the effect of magnetization outside the nominal slice profile.

### `params_dict.f0`
Defines the Larmor frequency `f0`. This parameter is required if the simulation should include RF pulses with ppm frequency offsets, for example in CEST-related applications.

### `params_dict.time_stamps`
ADC time stamps from the measurement. These are required for correction of trigger delays in cardiac MRF.

### `params_dict.soft_delays`
Stores the user-entered soft delay values. These are required for accurate modeling of the acquisition window in cardiac MRF.

### `params_dict.flag_kz`
Optional flag used for stacked 3D MRF. It allows OpenMRF to identify kz partitions and eliminate unnecessary partitions during dictionary generation.

### `params_dict.echo_mode`
Defines the echo mode. If left empty, the default mode is `'spiral_out'`.

---

## 2. Define the Look-Up Table

In a second step, the parameter ranges of the dictionary are defined. This is done in the same reconstruction script by creating a look-up table.

The following examples illustrate common configurations.

=== "T1 + T2"
    ```matlab
    %% parameters: dictionary and look-up table

    % T1, T2
    P.T1.range = [0.01,  4]; P.T1.factor = 1.05;
    P.T2.range = [0.001, 3]; P.T2.factor = 1.05;
    P = MRF_get_param_dict(P, {'T2<T1'});
    look_up       = [P.T1, P.T2];
    look_up_names = {'T1', 'T2'};

    params_dict.P             = P;
    params_dict.look_up       = look_up;
    params_dict.look_up_names = look_up_names;
    clear P look_up look_up_names;
    ```

=== "T1 + T2 + T1ρ"
    ```matlab
    %% parameters: dictionary and look-up table

    % T1, T2, T1p
    P.T1.range  = [0.01,  4]; P.T1.factor  = 1.05;
    P.T2.range  = [0.001, 3]; P.T2.factor  = 1.05;
    P.T1p.range = [0.001, 3]; P.T1p.factor = 1.05;
    P = MRF_get_param_dict(P, {'T2<T1', 'T2<T1p', 'T1p<T1'});
    look_up       = [P.T1, P.T2, P.T1p];
    look_up_names = {'T1', 'T2', 'T1p'};

    params_dict.P             = P;
    params_dict.look_up       = look_up;
    params_dict.look_up_names = look_up_names;
    clear P look_up look_up_names;
    ```

=== "T1 + T2 + B1+ correction"
    ```matlab
    %% parameters: dictionary and look-up table

    % T1, T2, B1+ correction
    P.T1.range  = [0.01,  4]; P.T1.factor = 1.025;
    P.T2.range  = [0.001, 3]; P.T2.factor = 1.025;
    P.db1.range = [0.8, 1.2]; P.db1.step  = 0.025;
    P = MRF_get_param_dict(P, {'T2<T1'});
    look_up       = [P.T1, P.T2, P.db1];
    look_up_names = {'T1', 'T2', 'db1'};

    params_dict.P             = P;
    params_dict.look_up       = look_up;
    params_dict.look_up_names = look_up_names;
    clear P look_up look_up_names;
    ```

### Range Definitions
The look-up table is generated from the fields in `P`. For each parameter, you can either define:
- a `range` together with a multiplicative `factor`, or
- a `range` together with a fixed `step`

This allows flexible sampling of the parameter space depending on the application.

### Constraints
The function `MRF_get_param_dict()` also supports constraints between parameters. For example:
- `'T2<T1'`
- `'T2<T1p'`
- `'T1p<T1'`

These constraints help exclude physically implausible combinations and reduce the total dictionary size.

---

## 3. B1+ Correction

Especially at field strengths of 3T and above, it can be beneficial to include B1+ deviations as an additional parameter in the dictionary.

This increases the dictionary size and computational cost, but may improve the quantitative accuracy if the sequence contains enough information to separate B1+ effects from relaxation effects.

!!! warning "Confounding effects of B1+ and T2"
    If the MRF sequence does not contain dedicated T2 preparation modules, the effects of T2 relaxation and B1+ inhomogeneity may not be separable in a robust way. In such cases, including B1+ as an additional parameter may lead to unstable or ambiguous matching results.

In general, B1+ correction is most meaningful for sequences that contain sufficient contrast encoding, for example through multiple inversions and dedicated T2 preparations.

---

## 4. A Look Under the Hood

The sections above describe the parameters that users typically need to choose. Internally, however, OpenMRF performs several additional processing steps before the actual dictionary simulation starts.

The key idea is that the `.seq` file is not simulated directly line by line during matching. Instead, it is first translated into a compact list of simulation instructions stored in the `SIM` struct. This provides a general intermediate representation that can be interpreted efficiently by the Bloch or EPG simulator.

### From `.seq` File to `SIM` Struct

The conversion from Pulseq sequence to simulation instructions is performed when the sequence is read and analyzed. During this step, OpenMRF automatically detects different sequence scenarios and classifies the events accordingly.

Examples of automatically detected conditions include:

- conventional RF excitations
- slice-selective excitation pulses
- slice-selective refocusing pulses
- adiabatic pulses
- continuous-wave spin-lock modules
- adiabatic spin-lock modules
- timing blocks, gradient moments, and frequency offsets

These events are converted into a compact instruction set stored in fields such as:

- `SIM.ID`
- `SIM.RF`
- `SIM.GX`, `SIM.GY`, `SIM.GZ`
- `SIM.DB`
- `SIM.DT`
- `SIM.PHI`

The purpose of this conversion is to replace the raw sequence description by a simulation-friendly representation that preserves the relevant physics while avoiding repeated high-level parsing during dictionary generation.

### Pre-Simulations Before Dictionary Generation

Before the `SIM` struct is used for the actual dictionary simulation, OpenMRF performs several pre-simulation steps. These steps generate reusable building blocks that make the final simulation much faster.

#### Slice Profile Pre-Simulation

For slice-selective pulses, OpenMRF precomputes slice profiles in 1° steps. These profiles are generated once and then stored for later reuse.

This means that during the final dictionary simulation, the simulator does not need to recompute the full slice profile response for every dictionary entry. Instead, it can directly access the precomputed slice-profile operators from the `SIM` struct.

The corresponding precomputed data are stored in:

- `SIM.SPROF`

This is one of the key reasons why OpenMRF can model slice-profile effects while still keeping the final dictionary calculation practical.

#### Transition Matrices for Adiabatic Pulses

Adiabatic RF pulses are long and cannot simply be treated as ideal instantaneous rotations. Relaxation occurs during the pulse itself, and this effect depends on the tissue parameters.

To handle this efficiently, OpenMRF precomputes a **transition matrix** for each adiabatic pulse. This matrix describes how the magnetization is transformed from the state before the pulse to the state after the pulse while accounting for relaxation during the pulse.

Importantly, these transition matrices are **T1- and T2-dependent**. They are therefore calculated in advance for the parameter combinations represented in the look-up table and stored for later use.

The corresponding data are stored in:

- `SIM.TM`

During the final dictionary simulation, the simulator can then apply these precomputed matrices directly instead of re-simulating the full adiabatic waveform every time. This greatly accelerates the simulation while preserving the relevant physics.

#### Adiabatic Spin-Lock and Additional Sequence Logic

OpenMRF also detects specific conditions such as adiabatic spin-locking and stores the necessary reduced information in the `SIM` struct. In this way, the final simulator operates on a compact and sequence-aware instruction list rather than on the original waveform objects.

This makes the framework highly flexible: new MRF concepts can often be supported automatically as long as their sequence logic can be recognized during `.seq` parsing and translated into the `SIM` representation.

### Why This Matters

This internal workflow is one of the main strengths of OpenMRF:

1. The `.seq` file is interpreted directly  
2. The sequence is translated into a compact `SIM` instruction list  
3. Pre-simulations generate reusable operators such as slice profiles and adiabatic transition matrices  
4. The final dictionary simulation becomes both accurate and efficient

In short, OpenMRF combines a very general sequence description with targeted precomputation. This makes it possible to support complex MRF concepts—such as slice-profile-aware Bloch simulation, spin-locking, adiabatic spin-locking, and relaxation during long RF pulses—without requiring a custom simulator for every new sequence design.
