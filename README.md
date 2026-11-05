# Introduction

This repository builds the Docker images for the MatML ecosystem ([MatGL], [MatCalc], [MatPES]) and
LAMMPS builds that can run MatGL foundation potentials. Images are published to the
[Materialyze Docker Repository]. The Dockerfiles are in [`docker/`](docker), and each one is built and
pushed by a GitHub Actions workflow in [`.github/workflows/`](.github/workflows).

# Available Docker Images

## matgl

[MatGL] alone, or MatGL plus a LAMMPS build that runs MatGL potentials natively through
`pair_style matgl` (and `matgl/kk` with Kokkos).

| Image | Dockerfile | Platforms | Contents |
| --- | --- | --- | --- |
| `materialyzeai/matgl:latest` | `Dockerfile.matgl` | amd64, arm64 | Python 3.12 and `matgl>=4.1.4`. No LAMMPS. |
| `materialyzeai/matgl:latest-kokkos-cpu` | `Dockerfile.matgl.kokkos-cpu` | amd64, arm64 | `matgl`, CPU-only torch, and LAMMPS (`stable_22Jul2025_update5`) with KOKKOS (OpenMP + Serial), ML-MATGL, ML-SNAP, MANYBODY, MOLECULE, EXTRA-COMPUTE/DUMP/FIX. LAMMPS is a shared library with the `lammps` Python module, linked against the torch wheel's libtorch. |
| `materialyzeai/matgl:latest-kokkos` | `Dockerfile.matgl.kokkos` | amd64 | CUDA 12.4 build: `matgl` plus LAMMPS with KOKKOS (CUDA + OpenMP) and ML-MATGL, linked against libtorch 2.5.1 (cu124). Default Kokkos arch is `AMPERE80` (A100). Single GPU only. Not built by CI; build it locally (see the header of the Dockerfile). |
| `materialyzeai/matgl:latest-dgl` | `Dockerfile.matgl.dgl` | amd64, arm64 | Legacy. `matgl:latest` with `dgl==2.1.0` and `torch==2.2.0`. See the note below. |

## matml

The full MatML stack: `matgl`, `matcalc` and `matpes`, together with LAMMPS.

| Image | Dockerfile | Platforms | Contents |
| --- | --- | --- | --- |
| `materialyzeai/matml:latest` | `Dockerfile.matml` | amd64, arm64 | `matgl>=4.1.4`, `matcalc`, `matpes` on top of `materialyzeai/matgl:latest-kokkos-cpu` (LAMMPS with KOKKOS OpenMP, `pair_style matgl`, ML-SNAP, usable from Python). torch is held at the base image's version so it still matches LAMMPS. Rebuilt automatically whenever the base image is republished. |
| `materialyzeai/matml:latest-kokkos-cpu` | `Dockerfile.matml.kokkos-cpu` | amd64, arm64 | Same build as `matml:latest`. |
| `materialyzeai/matml:latest-dgl` | `Dockerfile.matml.dgl` | amd64, arm64 | Legacy. `matml:latest` with `dgl==2.1.0` and `torch==2.2.0`. See the note below. |

**Choosing an image.** Use `matml:latest` for MD and property calculations with MatGL potentials, since
`pair_style matgl` runs the model inside LAMMPS. Use `matgl:latest` when you only need MatGL in Python.

**DGL images.** MatGL removed the DGL backend in v4.0.0, so `MATGL_BACKEND=DGL` has no effect in the
`-dgl` images and they do not provide the old DGL models. In `matml:latest-dgl`, replacing torch also
breaks LAMMPS. For DGL-based models, use an environment with `matgl<4`.

# Running code

Assuming you have your script `script.py` in the current directory, you can run it using:

```shell
docker run --name test -v .:/work -t materialyzeai/matml python /work/script.py
```

If you need to get a file out of your docker container, use:

```shell
docker cp test:<filename> .
```

## LAMMPS with MatGL potentials (`matml` and Kokkos images)

These images run a single LAMMPS process. `pair_style matgl` does not support multi-rank domain
decomposition, so use threads (`-k on t N`) rather than MPI ranks. First export a MatGL model to
TorchScript in the same image, so that its torch matches the one LAMMPS links:

```shell
docker run --rm -v .:/work -t materialyzeai/matml \
    mgl create-lammps-model -m TensorNet-PES-MatPES-r2SCAN-2025.2 -o model.pt --dtype float32
```

In `in.lammps`, enable the atom map and list the species of each LAMMPS atom type after the model:

```text
atom_modify     map yes          # before create_box / read_data
pair_style      matgl
pair_coeff      * * model.pt Si
```

Then run LAMMPS with the Kokkos OpenMP backend, or with the plain (non-Kokkos) pair style:

```shell
docker run --rm -v .:/work -t materialyzeai/matml \
    lmp -k on t 8 -sf kk -pk kokkos newton on neigh half -in in.lammps
docker run --rm -v .:/work -t materialyzeai/matml lmp -in in.lammps
```

LAMMPS can also be driven in-process from Python, e.g. through MatCalc with `MATCALC_BACKEND=LAMMPS`,
or directly:

```python
from lammps import lammps

lmp = lammps(cmdargs=["-k", "on", "t", "8", "-sf", "kk"])
```

For the CUDA image, replace the Kokkos flags with `-k on g 1 -sf kk -pk kokkos newton on neigh half`.

## Extending the Kokkos images

Packages installed on top of `matgl:latest-kokkos-cpu` must not change the torch version, because LAMMPS
is built against that torch's libtorch. Install with the bundled constraint file:

```dockerfile
FROM materialyzeai/matgl:latest-kokkos-cpu
RUN UV_TORCH_BACKEND=cpu uv pip install --system -c /opt/lammps/torch-constraint.txt <packages>
```

# Running on HPC such as Expanse or NERSC

SDSC and NERSC uses singularity. You will first need to build a singularity image from the docker images.

## Building the SIF

SSH to the cluster and start an interactive session with enough memory, e.g.:

```shell
srun -N 1 -n 1 -c 4 --mem 50G -p condo -q condo -A <allocation> -t 4:00:00 --pty /bin/bash
```

or if you are using the `hotel` queue:

```shell
srun -N 1 -n 1 -c 4 --mem 50G -p hotel -q hotel -A <allocation> -t 4:00:00 --pty /bin/bash
```

Next pull the docker container.

```shell
module load singularitypro
singularity pull --disable-cache docker://materialyzeai/matml
```

You should see a `matml_latest.sif` file in your directory. Exit the interactive shell. Other images
and tags are pulled the same way, e.g. `docker://materialyzeai/matgl:latest-kokkos-cpu`.

## Running jobs using the SIF container

You can now run a job using the SIF container. An example script is given below:

```bash
#!/bin/bash
#SBATCH --job-name="singularity"
#SBATCH -t 00:05:00
#SBATCH --nodes=1
#SBATCH --tasks-per-node=1
#SBATCH -o sing.out
#SBATCH -e sing.err
##SBATCH -p hotel
#SBATCH --qos hotel
#SBATCH -A <allocation>
#SBATCH --export=ALL

export OMP_NUM_THREADS=1

module load singularitypro

IMAGE=/tscc/nfs/home/shyuep/test_singularity/matml_latest.sif

singularity exec $IMAGE python script.py
```

For LAMMPS runs, request the cores as threads for one task (e.g. `--cpus-per-task=8`) and set
`OMP_NUM_THREADS` to match the `-k on t N` value.

[Materialyze Docker Repository]: https://hub.docker.com/orgs/materialyzeai/repositories
[MatGL]: https://github.com/materialyzeai/matgl
[MatCalc]: https://github.com/materialyzeai/matcalc
[MatPES]: https://github.com/materialyzeai/matpes
