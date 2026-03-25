---
hide:
    - navigation
---

# :rocket: Quickstart Guide

The following steps briefly outline how to set up OpenMRF, compile a basic sequence, and reconstruct parametric maps from the acquired data.

## 0. Recommended directory setup

Before cloning the main OpenMRF repository, we recommend creating a top-level folder called `OpenMRF` at your preferred location and keeping the main framework code separate from your individual projects. 

## 1. Clone the repository. 
Go to <https://github.com/OpenMRF/openmrf-core-matlab>. We recommend you fork the repository - this will allow you to make and keep track of changes while also staying up to date with our updates. Then, open a terminal, navigate to the location where you want your code to live (if you follow the above recommendation, this would be the `OpenMRF` folder), and run 
=== "Original repo"
    ```bash
    git clone <git@github.com:OpenMRF/openmrf-core-matlab.git>
    ```
=== "Forked repo"
    ```bash
    git clone <link-to-your-forked-repo>
    ```

This will create the following file tree:
```text
- OpenMRF
    `-- openmrf-core-matlab
```

## 2. Add/confirm system specifications. 
In the `openmrf-core-matlab` folder, navigate to `user_specifications/system_definitions` and check whether there
exists a `.csv` file with your scanner’s system specifications. If it does, still open it and double check that the specifications listed are accurate. If it doesn’t, create a file for your scanner using one of the existing files as a template, and save it in the same location using the same naming convention.

!!! warning "Warning"
    Please be sure to double check your system specifications. Running Pulseq sequences that are compiled for a different gradient system can potentially damage your scanner. 

<a id='asc-info'></a>
If you have a `.asc` file for your scanner, add it under `user_specifications/system_definitions/asc_files`. Make sure the name matches the `ascfile` field in the corresponding scanner's `.csv` file. This file is needed for PNS simulations. 

!!! info "`.asc` files"
    You don't need an `.asc` file to compile sequences. However, if your sequence happens to result in too high PNS, your sequence might not run on the scanner. 

## 3. Set user definitions. 
Open Matlab, navigate to the `openmrf-core-matlab` folder. Run `install_OpenMRF.m`. 

!!! info "install_OpenMRF.m"
    Every time you reopen Matlab, you will need to run this script first in order to be able to run any other scripts. However, after the first time, it doesn't require any user input. 
    
The first time you run this script, this will create multiple pop-up windows prompting you to enter your user information:

- Username: Used to sign sequences you create. Can be a combination of your first and last name, for example. Will show up in the filename of all sequences you create. Please avoid special characters and white spaces. 
- Lab: Will be stored in the backup information for all sequences you create. If your lab isn’t listed, select “Other” and enter your affiliation manually. 
- Path to backup data: This is where all your backup data will be stored. Every time you write a sequence, a subfolder will be created containing the .seq file needed to run the sequence on the scanner, as well as backup information necessary for reconstruction. If you want all OpenMRF related code and files to live in the same location, follow the recommendation from step 0 and choose the top-level `OpenMRF` directory. 
- Default MRI scanner: the hardware limitations of this scanner will be used per default, unless you specify a different scanner in your code. 
!!! info "Scanner selection"
    Every time you compile a sequence, the currently selected scanner specifications will be printed to the terminal. If the variable `pulseq_scanner` is not specified before `pulseq_init` in your sequence creation script, your default scanner will be automatically selected. More information [here](wiki/scanner.md).

## 4. Take a look around! 
OpenMRF is a comprehensive framework with many different parts to it. For getting started, we strongly recommend taking a closer look at all the different examples in the `main_sequences` folder, and especially at the example MRF sequences in `main_sequences/fingerprinting`. In every subfolder, you will find code to compile the respective sequence (usually starts with `pulseq_....m`) as well as the corresponding code for reconstruction of the acquired raw data (usually starts with `reco_....m`). 

## 5. Compile a test sequence. 
Navigate to `main_sequences/fingerprinting` and open `pulseq_mrf.m`. At the beginning of every sequence creation script, you can define five flags (setting the flags is optional, if not set explicitly, they default to 0). 

- `flag_backup`: When set to 0, nothing is saved. When set to 1, a `.seq` file and all corresponding backup files are saved in the `Pulseq_Workspace` subfolder. When set to 2, the .seq file is also copied to the `Seq_Export` folder (facilitates exporting of a larger number of .seq files). If either the `Pulseq_Workspace` or the `Seq_Export` folder do not yet exist, they will automatically be created in the backup location specified during the first execution of `install_OpenMRF.m`. Here's an overview of the resulting file tree:

```text
OpenMRF/
|-- openmrf-core-matlab/
|-- Pulseq_Workspace/
    `-- username/                                           # set during the first run of install_OpenMRF.m
        `-- yymmdd/                                         # today's date
            `-- yymmdd_hhmm/                                # timestamp created from current date and time
                |-- yymmdd_hhmm_username_seqname.seq        # .seq file to put on the scanner
                |-- backup_yymmdd_hhmm_username_seqname.m   # a copy of the matlab script used to create the .seq file
                |-- backup_yymmdd_hhmm_workspace.mat        # a copy of the matlab workspace
                |-- include_pulseq_toolbox.tar              # a copy of the include_pulseq_toolbox folder
                `-- md5_???.txt                             # md5 hash and other information in human-readable format
`-- Seq_Export/
    `-- username/
        `-- yymmdd_hhmm_username_seqname.seq                # same .seq file as in Pulseq_Workspace
```

!!! warning "Timestamps"
    If you're trying to compile more than one sequence per minute, you will receive the following warning: `Warning: pulseq duplicate detected! .seq writing stopped!` and nothing will be saved (otherwise the timestamps used in the naming convention would be ambiguous). 

- `flag_report`: when set to 1, a timing check report will be printed to the command window. When set to 2, additional hardware checks will be performed (takes longer). 
- `flag_pns`: when set to 1, a PNS simulation will be performed. Only works if you have a correctly named `.asc` file in the correct file location (see [here](#asc-info)). Per default, the PNS will be simulated for an axial slice. To change this default behavior, add a line 
```matlab
pns_orientation = 'coronal';
```
for PNS in coronal slice orientation or 
```matlab
pns_orientation = 'all';
```
to show PNS in all major slice orientations. 
!!! info "PNS Simulation"
    Even when the simulated PNS is under 100%, your sequence might not run on the scanner. Based on our experience we recommend staying under 85%. PNS can be reduced by adjusting the gradient slew rates. More information [here](troubleshoot.md#stimulation-limit-exceeded). 
- `flag_sound`: when set to 1, the sound resulting from gradient vibrations when running your sequence will be simulated and played by your default speaker. 
- `flag_mrf`: when set to 1, an MRF dictionary is created based on your sequence, allowing you to confirm that your sequence creates your intended signal evolutions. 

## 6. Acquire data.
For more information on how to run `.seq` files on your scanner, see <https://pulseq.github.io/index.html>. Export the raw data as a `.dat` file. 

!!! danger 
    Make sure you know what you're doing before you start running Pulseq sequences on your scanner. 

<!-- !!! tip "`Timing and Flip Angles` on Siemens scanners"
    If you are using a Siemens system, make sure to set `Timing and Flip Angles` in the `Sequence/Special` tab to `strict`. Otherwise, all RF events might be rescaled if the peak RF voltage exceeds your scanner's limit.  -->

## 7. Reconstruct the data and create parametric maps.
In Matlab, navigate to `main_sequences/fingerprinting` and open `reco_mrf.m`. Set `vendor` to the make of the MRI machine you used for data acquisition (capitalized), and let `path_raw_mrf` point to the rawdata file. For now, we are not correcting for trajectory imperfections, so delete or comment the line defining `path_raw_traj` - in this case, the nominal trajectory will be used for reconstruction (more information on trajectory calibration [here](wiki/trajectory.md)). After having updated the paths, run the file. Once the reconstruction is complete, a figure showing the final parametric maps should appear, as well as a pop-up window asking where to save the results. 

## 8. Your feedback matters! 
Please help us improve OpenMRF. If you encounter any bugs, issues, or limitations - send us an [email](mailto:maximilian.gram@uni-wuerzburg.de,tomgr@med.umich.edu) or create an issue on the [github page](https://github.com/OpenMRF/openmrf-core-matlab/issues)! 