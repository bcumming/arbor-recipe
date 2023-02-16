# Arbor Development Env Recipe

A spack stack with everything needed to develop Arbor on for the A100 nodes on Hohgant.

This guide walks us through the process of configuring a spack stack, building and using it.

Arbor is a C++ library, with optional support for CUDA, MPI and Python. An Arbor developer would ideally have an environment that provides everything needed to build Arbor with these options enabled.

The full list of all of the Spack packages needed to build a full-featured CUDA version is:
- MPI: `cray-mpich-binary`
- compiler: `gcc@11`
- Python: `python@3.10`
- CUDA: `cuda@11.8`
- `cmake`
- `fmt`
- `pugixml`
- `nlohmann-json`
- `random123`
- `py-mpi4py`
- `py-numpy`
- `py-pybind11`
- `py-sphinx`
- `py-svgwrite`

For the compiler, we choose `gcc@11`, which is compatible with cuda@11.8.

## Write the Recipe

Spack stacks start with a declarative recipe, written in yaml.

- GCC 11 is specified in [`compilers.yaml`](compilers.yaml).
- Hohgant is specified in [`config.yaml`](config.yaml).
- The rules for generating modules in [`modules.yaml`](modules.yaml).
- The environments in [`environments.yaml`](environments.yaml).

## Get the tool

### Method 1: GitHub

```bash
git clone git@github.com:eth-cscs/stackinator.git
(cd stackinator && ./bootstrap.sh)
export PATH=$(pwd)/stackinator/bin:$PATH
stack-config --help
```

### Method 2: Pip

```bash
pip install stackinator
stack-config --help
```

## Building a Software Stack

Parallel builds in memory (`/dev/shm`)  are great!
Stepping on the toes of other users on the login nodes is not great!
Work on a compute node.

```bash
salloc -t180 -N1 --partition=cpu
ssh nid003193
```

> **Note**
> Build on the compute node architecture that you are targetting.
> At least the same CPU type, however if targetting CUDA you will also need a
> node that has the NVIDIA drivers installed.

Spack stacks start with a declarative recipe, written in yaml.
This repository is one such recipe.

```bash
git clone git@github.com:bcumming/arbor-recipe.git
```

Use `stack-config` to configure the recipe.

> **Note**
> This step is equivalent to running configure or cmake - the input is a
> description of what to build, the output is a set of Makefiles and sources
> that are run to build the software stack.

```bash
# -r: the source path for the recipe
# -b: the path for the out of tree build
stack-config -rarbor-recipe -b/dev/shm/bcumming/arbor
```

Next perform the `make` step in building software, where the final target is a squashfs file with the development environment.

```bash
# build the image
cd /dev/shm/bcumming/arbor
env --ignore-environment PATH=/usr/bin:/bin:`pwd`/spack/bin make store.squashfs -j64

# save the image
mv store.squashfs $SCRATCH/arbor.squashfs
```

> **Note**
> Always call make with `env --ignore-environment` to ensure a clean environment
> when running the build. This increases reproducability, and isolates the build process
> from arbitrary changes to the environment (which are very rare, but you can't
> be too careful.)

Don't forget to delete your temporary working path if you used `/dev/shm` - it uses memory on the node, and won't be cleaned up automatically when you log out or your allocation finishes.

## Using the stack to build

To start a new process with the image mounted on a node that you are logged into (login or compute), use the `squashfs-{mount,run}` utilities.
The image was configured to be mounted at `/user-environment`, which is the location that `squashfs-run` will always mount images.

```bash
squashfs-mount $SCRATCH/arbor.squashfs /user-environment bash
squashfs-run $SCRATCH/arbor.squashfs bash
```

The squashfs images can be used as a Spack upstream, and optionally can provide a module environment.

The upstream Spack configuration is in `$mount/config`:

```bash
ls /user-environment/config
```

I am old-fashioned, so I will use modules to build Arbor.

```bash
module use /user-environment/modules/
module avail
module load cmake gcc cray-mpich fmt ninja nlohmann-json random123 python cuda pugixml py-pybind11
```

Through the magic of CMake we build our software:

```
git clone git@github.com:arbor-sim/arbor.git
cd arbor/
mkdir build
cd build/
CC=mpicc CXX=mpic++ cmake .. -G Ninja -DARB_WITH_MPI=on -DARB_WITH_PYTHON=on -DARB_GPU=cuda
ninja examples pyarb
```

## Run the application

```bash
# first attempt...
srun -n1 --partition=cpu ./bin/ring

# ... and we get an error like the following:
/scratch/e1000/bcumming/arbor/build/./bin/ring: error while loading shared libraries: libpugixml.so.1: cannot open shared object file: No such file or directory
```

When `squashfs-mount` or `squashfs-run` was used to mount the image to build the application, it was only mounted for the one process on that node. Other users/processes on the node don't see the mounted image, likewise it won't be mounted on the compute nodes `srun` will launch the MPI ranks on.

A SLURM plugin installed on Hohgant (developed by @simonpintarelli and @jpcoles) accepts a flag to mount an image for each rank on the compute nodes.

```bash
srun -n1 --partition=cpu --uenv-file=$SCRATCH/arbor.squashfs ./bin/ring
```
