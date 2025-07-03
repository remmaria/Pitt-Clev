# Docker pipe_dmri: Diffusion MRI Processing Pipeline

## Folder and Files Structure

The expected structure consists of a `main_folder` containing multiple session folders. Each session folder must include:

- The raw diffusion-weighted image (DWI)
- Its corresponding `.bval` and `.bvec` files (same root name)
- If applicable, a reverse phase-encoding volume

```
# Example:
main_folder/
├── session_01/
│   ├── dwis_PA.nii(.gz)           # Raw DWI image (.nii or .nii.gz)
│   ├── dwis_PA.bval               # b-values
│   ├── dwis_PA.bvec               # b-vectors
│   ├── dwis_AP.nii(.gz)           # Reverse phase-encoded volume (.nii or .nii.gz)
│
├── session_02/
│   ├── ...
```

An output folder path can be provided to store processed files.

```
output_path/
├── session_01/
│   ├── {processed_files}
│   └── stats_analysis.csv
├── session_02/
│   ├── ...
```
> **Note:** The `{session}` name is inferred from the input path.  
> Example: If `-w` is set to `main_folder/session_01/dwis_PA.nii`, the processed files will be saved to `{output_path}/session_01/`.
> If output_path is not specified, the processed files will be saved in the same folder as the input DWI: `main_folder/session_01/`.

## Temporary Folder

If desired, a temporary folder can be specified to store intermediate files (e.g., .mif files). If not provided, the default location is /tmp.
When running on a computing cluster (e.g., CRC), it is recommended to use the $SLURM_SCRATCH directory:

```bash
-u $SLURM_SCRATCH
```

## Example Script Usage
![Pipeline diagram](images/pipe_ex1_red.png)
```bash
docker run --rm -v $PWD:$PWD -w $PWD pipe_dmri pipeline \
    -t $N_CPUS \      # optional:Default 1
    -o $output_path \ # optional: Default same folder as input dwi
    -u $tmp_folder \  # optional: Default: /tmp
    -w main_folder/session_01/dwis_PA.nii.gz \
    -p 0 -g -e PA -i main_folder/session_01/dwis_AP.nii.gz \
    -d 1000 -m \
    -y csd_msmt -l CC_1,CC_2,CC_3,CC_4,CC_5,CC_6,CC_7,CG_left,CG_right,SLF_I_left,SLF_I_right,SLF_II_left,SLF_II_right,SLF_III_left,SLF_III_right \
    -a CC_1,CC_2,CC_3,CC_4,CC_5,CC_6,CC_7,CG_left,CG_right,SLF_I_left,SLF_I_right,SLF_II_left,SLF_II_right,SLF_III_left,SLF_III_right-dti \
    -x $csv_info      # optional
```
This script performs:

1. `-p 0`: MP-PCA denoising (not exporting noise_map and noise_residuals)
2. `-g`: Gibbs ringing removal
3. `-e PA -i main_folder/session_01/dwis_AP.nii.gz`: Eddy + motion correction (with inverse phase-encoding)
4. `-d 1000`: DTI fitting with shell of 1000 for AD, MD and RD computing. Adjust this value according to the diffusion sequence used. FA considers all shells.
5. `-m`: Skull stripping using SynthStrip
6. `-y csd_msmt -l CC_1...SLF_III_right`: WM bundle segmentation using [TractSeg](https://github.com/MIC-DKFZ/TractSeg) with FOD method 'csd_msmt' (multi-shell) for WM Tracts (CC_1–CC_7, CG (RL), SLF_I-III (RL)). Adjust to `csd` for single-shell sequence.
7. `-a CC_1...SLF_III_right-dti`: Statistical analysis using TractSeg bundle masks (CC_1–CC_7, CG (RL), SLF_I-III (RL)) applied to DTI metrics (AD, MD, RD, FA).
8. `-x $csv_info`: CSV info file – additional information about each subject/session that can be included in the statistical output. The `{session}` name must match the SessionID column in the CSV file exactly. Works fine without it.

> For best results using TractSeg:
> - Ensure **MNI-compatible orientation** (e.g., like HCP data)
> - Make sure **LEFT hemisphere** is properly aligned
> - Use **isotropic voxel spacing**.
>
> For multi-shell data, you can use other FOD methods: `csd_msmt`, `csd_msmt_5tt`, etc.
> 
> More info: https://github.com/MIC-DKFZ/TractSeg/tree/master

The run_pipeline script generates a `stats_analysis.csv` file for each session. To speed up processing, you can run sessions independently in parallel. Once all sessions have been processed, you can merge the results using:
```bash
docker run -v $PWD:$PWD -w $PWD pipe_dmri stats_merge -i {input_folder} -o path/to/output/results_merged.csv
```
Replace `{input_folder}` with the directory that contains all session folders — this is the same as the `main_folder` or `output_folder`, depending on how you configured your pipeline.
