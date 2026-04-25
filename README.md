# Introduction

This repository provides tools for managing the MatML ecosystem, including building of Docker images.

# Available Docker Images

All docker images are at the [Materials Virtual Lab Docker Repository].

# Running code

Assuming you have your script `script.py` in the current directory, you can run it using:

```docker
docker run --name test -v .:/work -t materialyzeai/matml python /work/script.py
```

If you need to get a file out of your docker container, use:

```docker
docker cp test:<filename> .
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

You should see a `matml_latest.sif` file in your directory. Exit the interactive shell.

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

[Materials Virtual Lab Docker Repository]: https://hub.docker.com/orgs/materialyzeai/repositories
