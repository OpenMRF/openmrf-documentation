---
hide:
    - navigation
---

# :rocket: Quickstart Guide

The following steps briefly outline how to set up OpenMRF, compile a basic sequence, and reconstruct parametric maps from the acquired data.

## 1. Fork and clone the repository. 
Go to <https://github.com/HarmonizedMRI/OpenMRF> and fork the repository. Then, open a terminal, navigate to the location where you want your code to live, and run 
```bash
git clone <link-to-your-forked-repo>
```

## 2. Add/confirm system specifications. 
Navigate to `user_specifications/system_definitions` and check whether there
exists a `.csv` file with your scanner’s system specifications. If it does, still open it and double check that the specifications listed are accurate. If it doesn’t, create a file for your scanner using one of the existing files as a template, and save it in the same location using the same naming convention.

!!! warning "Warning"
    Please be sure to double check your system specifications. Running Pulseq sequences that are compiled for a different gradient system can damage your scanner. 

<a id='asc-info'></a>
If you have a `.asc` file for your scanner, add it under `user_specifications/system_definitions/asc_files`. Make sure the name matches the `ascfile` field in the corresponding scanner's `.csv` file. 

!!! info "`.asc` files"
    You don't need an `.asc` file to compile sequences. However, if your sequence happens to result in too high PNS, your sequence might not run on the scanner. 

## 3. Set user definitions. 
Open Matlab, navigate to the main OpenMRF folder. Run `install_OpenMRF.m`. 

!!! info "install_OpenMRF.m"
    Every time you reopen Matlab, you will need to run this script first in order to be able to run any other scripts. However, after the first time, it doesn't require any user input. 
    
The first time you run this script, this will create multiple pop-up windows prompting you to enter your user information:

- Username: Used to sign sequences you create. Can be a combination of your first and last name, for example. Will show up in the filename of all sequences you create. 
- Lab: Will be stored in the backup information for all sequences you create. If your lab isn’t listed, select “Other” and enter your affiliation manually. 
- Path to backup data: This is where all your backup data will be stored. Every time you write a sequence, a subfolder will be created here containing the .seq file needed to run the sequence on the scanner, as well as backup information necessary for reconstruction. 
- Default MRI scanner: the hardware limitations of this scanner will be used per default, unless you specify a different scanner in your code. 
!!! info "Scanner selection"
    Every time you compile a sequence, the currently selected scanner specifications will be printed to the terminal. If the variable `pulseq_scanner` is not specified before `pulseq_init` in your sequence creation script, your default scanner will be automatically selected. More information [here](wiki/scanner.md).

## 4. Take a look around! 
OpenMRF is a comprehensive framework with many different parts to it. For getting started, we strongly recommend taking a closer look at all the different examples in the `main_sequences` folder, and especially at the example MRF sequences in `main_sequences/fingerprinting`. In every subfolder, you will find code to compile the respective sequence (usually starts with `pulseq_....m`) as well as the corresponding code for reconstruction of the acquired raw data (usually starts with `reco_....m`). 

## 5. Compile a test sequence. 
Navigate to `main_sequences/fingerprinting` and open `pulseq_mrf.m`. At the beginning of every sequence creation script, you will define five flags:

- `flag_backup`: when set to 1, a `.seq` file and all corresponding backup files are saved in a subfolder in your specified backup path. When set to 0, nothing is saved. 
!!! info "`flag_backup=2`;"
    You may encounter comments indicating that setting `flag_backup=2` results in backing up and sending the `.seq` file. This functionality is specific to the Experimental Physics 5 group at the University of Würzburg and won't work outside of their network - please just ignore. 
!!! warning "Timestamps"
    You can only create one sequence per minute with this flag active, otherwise the timestamps used in the naming convention would be ambiguous (more information [here](wiki/timestamps.md)).

- `flag_report`: when set to 1, a timing check report will be printed to the command window. When set to 2, additional hardware checks will be performed (takes longer). 
- `flag_pns`: when set to 1, a PNS simulation will be performed. Only works if you have a correctly named `.asc` file in the correct file location (see [here](#asc-info)). Per default, the PNS will be simulated for an axial slice. To change this default behavior, add a line 
```matlab
pns_orientation = 'coronal';

```
!!! info "PNS Simulation"
    Even when the simulated PNS is under 100%, your sequence might not run on the scanner. Based on our experience we recommend staying under 85%. PNS can be reduced by adjusting the gradient slew rate. More information [here](troubleshoot.md#stimulation-limit-exceeded). 
- `flag_sound`: when set to 1, the sound resulting from gradient vibrations when running your sequence will be simulated and played by your default speaker. 
- `flag_mrf`: when set to 1, an MRF dictionary is created based on your sequence, allowing you to confirm that your sequence creates your intended signal evolutions. 

## 6. Acquire data.
For more information on how to run `.seq` files on your scanner, see <https://pulseq.github.io/index.html>. Export the raw data as a `.dat` file. 

!!! danger 
    Make sure you know what you're doing before you start running Pulseq sequences on your scanner. 

!!! tip "`Timing and Flip Angles` on Siemens scanners"
    If you are using a Siemens system, make sure to set `Timing and Flip Angles` in the `Sequence/Special` tab to `strict`. Otherwise, all RF events might be rescaled if the peak RF voltage exceeds your scanner's limit. 

## 7. Reconstruct the data and create parametric maps.
In Matlab, navigate to `main_sequences/fingerprinting` and open `reco_mrf.m`. Change the variables `study_path` and `study_name_mrf` to point to your acquired data. For now, we are not correcting for trajectory imperfections, so delete or comment the line defining `study_name_traj` - in this case, the nominal trajectory will be used for reconstruction. After having updated the paths, run the file. Once the reconstruction is complete, a figure showing the final parametric maps should appear. 

## 8. Your feedback matters! 
Please help us improve OpenMRF. If you encounter any bugs, issues, or limitations - send us an [email](mailto:maximilian.gram@uni-wuerzburg.de,tomgr@umich.edu) or create an issue on the [github page](https://github.com/HarmonizedMRI/OpenMRF/issues)! 