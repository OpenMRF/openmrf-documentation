# Single Parameter Mapping

For validation of the OpenMRF framework, it is important to compare MRF results with gold-standard reference measurements obtained with dedicated single-parameter mapping sequences.

In this context, it is useful to distinguish between two categories:

- **T1 and T2 mapping**, which can be measured using vendor-provided protocols based on spin-echo acquisitions
- **Specialized relaxation parameters** such as **T1ρ**, **T2ρ**, or **adiabatic T1ρ**, which are usually not available as standard vendor sequences and are therefore provided by dedicated OpenMRF implementations

---

## T1 and T2 Mapping for OpenMRF Validation

For OpenMRF validation, T1 and T2 reference measurements were acquired using single-parameter spin-echo-based mapping protocols.

### T1 Mapping
The T1 mapping protocol for the acquisition of T1-weighted images uses:

- adiabatic inversion pulse
- identical spin-echo readouts
- minimal TE (e.g. 6.4 ms)
- TR = 5 s
- inversion delays: 35, 60, 95, 160, 260, 430, 710, 1180, 1940, 3200 ms

### T2 Mapping
The T2 mapping protocol for the acquisition of T2-weighted images uses:

- identical spin-echo sequence without inversion
- TR = 5 s
- echo times: 10, 15, 20, 30, 40, 60, 85, 120, 175, 250 ms

### Common Imaging Parameters

- FOV: 250 × 250 mm
- Matrix: 128 × 128
- Slice thickness: 6 mm
- Bandwidth: 281 Hz/pixel

The corresponding reconstruction and mapping code is available in:

```text
include_misc/src_nist_values/reco_spin_echo_T1_T2.m
```

!!! note
    We are currently working on a Pulseq-based implementation of a standardized spin-echo sequence for T1 and T2 mapping. This will enable vendor-independent, reproducible gold-standard reference measurements within the OpenMRF framework.

---

## Reference Mapping for T1ρ, T2ρ, and Adiabatic T1ρ

Reference measurements for **T1ρ**, **T2ρ**, and **adiabatic T1ρ** are not typically available as vendor-provided product sequences. For these parameters, OpenMRF provides dedicated implementations for both sequence compilation and reconstruction.

The corresponding source code can be found in:

```text
main_sequence/relax_mapping
```

These reference mapping sequences are based on an **interleaved spiral readout**.

### Default Acquisition Scheme

By default:

- each preparation time is acquired with an **8× interleaved spiral readout**
- a **5 s recovery delay** is applied between different readouts

### Rationale for Spiral-Based Reference Mapping

The main advantage of the spiral mapping approach is that the readout is primarily **proton-density weighted**, while the image contrast is dominated mainly by the preceding **T1ρ**, **T2ρ**, or **adiabatic T1ρ** preparation.

This is important because it supports the use of a **monoexponential signal model** for fitting, which is typically assumed for these relaxation parameters.
