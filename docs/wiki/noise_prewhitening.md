!!! info
    Currently only implemented for use with the SPI readout module. 

OpenMRF allows the easy inclusion of noise pre-whitening into your sequence. In all example sequences using the SPI readout module, this will be active per default. It requires two steps:

1. In the sequence creation script, add the noise pre-scans after initializing the SPI readout module and before adding any other objects to the sequence structure. 
```matlab
SPI.Nnoise = 16;
SPI_add_prescans();
```
2. In the reconstruction script, check for pre-scans during raw data read in and, if applicable, perform the pre-whitening operation. 
=== "Without noise pre-whitening"
    ```matlab
    DATA = SPI_get_rawdata(twix_obj, Nnoise);
    DATA = permute(DATA, [3,1,2]);
    DATA(:,:,1:adcNPad)  = [];     % adc padding
    ```
=== "With noise pre-whitening"
    ```matlab
    if isfield(PULSEQ.SPI, 'Nnoise')
        Nnoise = PULSEQ.SPI.Nnoise;
    else
        Nnoise = 0;
    end
    [DATA, ~, ~, ~, NOISE] = SPI_get_rawdata(twix_obj, Nnoise);
    DATA = permute(DATA, [3,1,2]); % mrf rawdata
    DATA(:,:,1:adcNPad)  = [];     % adc padding

    % noise pre-whitening
    if ~isempty(NOISE)
        NOISE = permute(NOISE, [3,1,2]);  % permute noise prescans
        NOISE(:,:,1:adcNPad) = [];        % adc padding
        DATA = mg_noise_prewhitIening(DATA, NOISE, 'cholesky', 1);
    end
    clear NOISE;
    ```