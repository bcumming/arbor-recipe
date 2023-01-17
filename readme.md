# Arbor Development Env Recipe

A spack stack with everything needed to develop Arbor on Hohgant, to illustrate how to specify, build and use a spack stack squashfs image.


## Building a Software Stack

Download and configure sstool:

```bash
git clone git@github.com:bcumming/sstool.git
export PATH=$(pwd)/sstool:$PATH
```

Spack stacks start with a declarative recipe, written in yaml.
This repository is one such recipe, which we can and configure.

> **Note**
> This step is equivalent to running configure or cmake - we are converting a description of
> what to build into a set of Makefiles and sources that are run to the software stack.

```bash
git clone git@github.com:bcumming/arbor-recipe.git

# -r: the source path for the recipe
# -b: the path for the out of tree build
sstool -rarbor-recipe -b/dev/shm/bcumming/arbor
```

The recipe is configured in the build path, and is literally equivalent to
the `make` step in building software

> **Note**
> Always call make with `env --ignore-environment` to ensure a clean environment
> when running the build. This increases reproducability, and isolates the build process
> from arbitrary changes to the environment (which are very rare, but you can't
> be too careful.)

```bash
cd /dev/shm/bcumming/arbor
env --ignore-environment PATH=/usr/bin:/bin:`pwd`/spack/bin make modules -j64
env --ignore-environment PATH=/usr/bin:/bin:`pwd`/spack/bin make store.squashfs

mv store.squashfs $SCRATCH/arbor.squashfs
```

don't forget to delete your temporary working path if you used `/dev/shm` - it uses memory on the node, and won't be cleaned up automatically when you log out or your allocation finishes.

## Build the Software


```bash
# Start a new process with the squashfs image mounted at /user-environment
# with a shell. The 
squashfs-mount $SCRATCH/arbor.squashfs /user-environment bash
squashfs-run $SCRATCH/arbor.squashfs bash
```

Set up my module environment
```bash
module use /user-environment/modules/
module avail
module load cmake gcc cray-mpich fmt ninja nlohmann-json random123 python cuda pugixml py-pybind11
```

```
git clone git@github.com:arbor-sim/arbor.git
cd arbor/
mkdir build
cd build/
CC=mpicc CXX=mpic++ cmake .. -G Ninja -DARB_WITH_MPI=on -DARB_WITH_PYTHON=on -DARB_GPU=cuda
```

I will build with the modules.

## Run the application

```bash
# first attempt...
srun -n1 --partition=cpu ./bin/ring

# ... and we get an error like the following:
/scratch/e1000/bcumming/arbor/build/./bin/ring: error while loading shared libraries: libpugixml.so.1: cannot open shared object file: No such file or directory
```

When `squashfs-mount` or `squashfs-run` was used to mount the image to build the application, it was only mounted for the one process on the login node. Other users on the login node don't see the mounted image, likewise it is not mounted on the compute node.

A SLURM plugin installed on Hohgant (developed by @simonpintarelli and @jpcoles) accepts a flag to mount an image for rank on the compute nodes.

```bash
srun -n1 --partition=cpu --uenv-mount-file=$SCRATCH/arbor.squashfs ./bin/ring
```

> **Note**
>
> The command line flags for the plugin are going to change to `--uenv-file` and `--uenv-mount`,
> and the plugin will also automatically mount images that have been mounted by the caller of
> srun/salloc/sbatch if no flags are explicitly set.

## The Packages

- `cmake`
- `fmt`
- `pugixml`
- `nlohmann-json`
- `random123`
- `cuda@11.8`
- `cray-mpich-binary`
- `py-mpi4py`
- `python@3.10`
- `py-numpy`
- `py-pybind11`
- `py-sphinx`
- `py-svgwrite`
