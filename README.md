# Multi-tasking version of gau_xtb: a Gaussian-xTB Interface Script

## Overview
This directory hosts a toolchain that couples the Gaussian `external` interface with the [xTB](https://github.com/grimme-lab/xtb) program. During a Gaussian calculation, xTB can be invoked on the fly to evaluate energies, gradients, and Hessians. Compared with the [original gau_xtb release](http://sobereva.com/soft/gau_xtb) proposed by **Dr. Tian Lu**, this version supports running multiple Gaussian+xTB jobs in parallel within the same folder and automatically cleans up the corresponding temporary artifacts, making it suitable for HPC environments and large-scale data generation.

## Directory Layout
| File | Description |
| --- | --- |
| `xtb.sh` | Core external script called by Gaussian, responsible for job orchestration and file management |
| `genxyz` / `genxyz.f90` | Utility that converts Gaussian’s `mol.tmp` into the `mol.xyz` format required by xTB |
| `extderi` / `extderi.f90` | Extracts energy, gradients, and Hessian data from xTB outputs and writes them back to Gaussian |
| `*.gjf` / `*.log` | Sample input and log files |

> The executables and the Fortran sources for `genxyz` and `extderi` are both provided and can be recompiled on the target platform if necessary.

## Key Features
1. **Automatic job prefix detection**
   - Prefer the `GAUSS_JOBNAME` supplied by Gaussian.
   - Fall back to the output file name or the input file name.
   - Sanitize the prefix to avoid naming conflicts within the same directory.
2. **Isolated working directories**: Each invocation creates its own temporary directory (`<prefix>_xtb_XXXXXX`) so that all xTB artifacts stay separated.
3. **Result extraction and hand-off**: `extderi` converts xTB outputs into the format expected by Gaussian, while the script optionally preserves intermediate files.
4. **Parallel job compatibility**: Unique prefixes and dedicated working folders allow multiple Gaussian+xTB jobs to run in parallel under the same path.
5. **Automatic cleanup**: Temporary directories and prefixed intermediates are removed by default to keep the workspace tidy.

## Workflow
1. Gaussian invokes `xtb.sh` via the `external` keyword.
2. The script parses the temporary input file (`$2`) and extracts atom count, derivative order, charge, and spin multiplicity.
3. A unique prefix is generated and a temporary working directory is created.
4. `genxyz` produces `mol.xyz`; xTB is then executed with the proper `--grad` / `--hess` options according to the derivative level.
5. `extderi` transforms xTB outputs into the format required by Gaussian and writes them to the file specified by `$3`.
6. Intermediate artifacts are either copied back or discarded based on configuration, and the temporary directory is deleted.

The following diagram summarizes the data flow:
```
Gaussian (external) -> xtb.sh -> genxyz -> xTB -> extderi -> Gaussian
```

## Prerequisites
- **Gaussian**: Must support the `external` keyword (Gaussian 09 or later).
- **xTB**: Install the official binaries or build from source; ensure the `xtb` executable is on the `PATH`.
- **bc**: Used to compute `uhf = spin - 1`.
- **mktemp**: Creates temporary directories (the script falls back to `$$` and `$RANDOM` if unavailable).
- **Fortran compiler (optional)**: Required only when recompiling `genxyz.f90` or `extderi.f90`.

## Quick Start
1. **Prepare the scripts**: Place `xtb.sh`, `genxyz`, and `extderi` in the same folder and grant execute permission.
2. **Author the Gaussian input**: Add `external="./xtb.sh"` to the route section, e.g.
   ```
   %chk=myjob.chk
   %mem=8GB
   %nprocshared=1
   # opt=(nomicro) external='./xtb.sh'

   My job title

   0 1
   ... molecular structure ...
   ```
3. **Submit the calculation**: Run Gaussian in the directory; the script handles the xTB calls automatically.
4. **Inspect the results**: Gaussian logs contain energy and gradient data from xTB. Refer to the environment variables section if you need additional intermediate outputs.

## Environment Variables
| Variable | Default | Description |
| --- | --- | --- |
| `GAUSS_JOBNAME` | Set by Gaussian | Automatically inferred by the script when absent |
| `OMP_NUM_THREADS` / `MKL_NUM_THREADS` | 1 | Controls the thread count for xTB and linked BLAS/LAPACK libraries |
| `XTB_KEEP_INTERMEDIATE` | 0 | Preserve prefixed files (e.g., `<prefix>_xtbout`, `<prefix>_mol.xyz`) when set to a non-zero value |

> For debugging, temporarily set `XTB_KEEP_INTERMEDIATE=1` to check xTB outputs, then revert to save disk space.

## Output and Cleanup Policy
- **Default behavior**: Remove the temporary directory and all `<prefix>_*` intermediates under the working path once the job finishes to keep parallel runs tidy.
- **Retention mode**: When `XTB_KEEP_INTERMEDIATE` is non-zero, the script copies back the following files if present: `mol.xyz`, `xtbout`, `gradient`, `hessian`, `charges`, `energy`, `xtbrestart`, `wbo`, `xtbtopo.mol`, `xtbhess.xyz`, `g98.out`, `g98_canmode.out`, `xtb_normalmodes`, `vibspectrum`. The copied files are all prefixed with `<prefix>_`.

## Troubleshooting
1. **xTB did not terminate normally**: The script logs “Warning: XTB may not have terminated normally.” Enable `XTB_KEEP_INTERMEDIATE=1` and review the preserved `xtbout` for details.
2. **`genxyz` or `extderi` lacks execute permission**: Run `chmod +x genxyz extderi`.
3. **Missing `bc` or `mktemp`**: Install them via the system package manager, e.g., `sudo apt-get install bc`.
4. **Prefix conflicts or invalid characters**: The script restricts prefixes to alphanumerics, underscores, hyphens, and dots to prevent conflicts.

## Changelog
- **2025-12-04**
  - Introduced prefix-based temporary directories to allow parallel jobs under the same folder.
  - Added automatic cleanup and an optional intermediate file retention mechanism.
  - Improved logging to monitor the xTB execution status.

## License
The original scripts were authored by Dr. Tian Lu (credit retained in the comments). If you plan to publish this work, comply with the original licensing terms as well as those of xTB and Gaussian, and acknowledge the sources in your repository.

## References
- Tian Lu, gau_xtb: A Gaussian interface for xtb code, http://sobereva.com/soft/gau_xtb
- [Official xTB repository](https://github.com/grimme-lab/xtb)
- [Community discussions and examples on Gaussian/xTB coupling](http://sobereva.com/421)

Feel free to open a GitHub issue for feature requests or suggestions.
