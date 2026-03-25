# Noise Pre-Whitening

OpenMRF allows easy integration of noise pre-whitening scans into your sequence. In all example MRF sequences using the SPI readout module, this feature is enabled by default.

In the sequence creation script, add the noise prescans after initializing the SPI readout module and before adding any other objects to the sequence structure.

```matlab
SPI.Nnoise = 16;     % number of prescans
SPI_add_prescans();  % script for adding prescans
```

In the reconstruction script, the acquired raw data are automatically separated into `DATA` and `NOISE` arrays:

```matlab
% load MRF rawdata
[DATA, NOISE, PULSEQ, study_info] = pulseq_read_meas(path_raw_mrf, path_backup_mrf, vendor);
```

In the `mrf_reco()` function, noise pre-whitening is automatically performed if `NOISE` is not empty. By default, the whitening matrix is computed via [Cholesky factorization](http://hansenms.github.io/sunrise/sunrise2013/Hansen_ImageReconstruction_ParallelImaging.pdf):

```matlab
% noise pre-whitening
if ~isempty(NOISE)
    DATA = mg_noise_prewhitening(DATA, NOISE(:,:,20:end), 'cholesky', 1);
end
```
!!! note
    `NOISE(:,:,20:end)`  
    The first few samples are excluded from the whitening matrix calculation, as they may be affected by ADC filtering effects.  
    The value `20` is chosen empirically and typically provides sufficient padding for stable estimation.