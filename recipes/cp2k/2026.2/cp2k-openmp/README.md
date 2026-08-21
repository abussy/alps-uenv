# CP2K uenv with OpenMP GPU offloading for EuroHack
This uenv contains all the necessary compilers and libraries for CP2K with standard options, with 
the addition of OpenMP offloding. It was tested with array additions in arbitrarily places in the 
CP2K code, i.e.:

```
    !$omp target teams distribute parallel do
    do i = 1, n
        c(i) = a(i) + b(i)
    end do
    !$omp end target teams distribute parallel do
```

compiles and runs on the GPU.

## Getting the uenv
After ssh-ing to the machine run

```sh
uenv image pull service::cp2k-openmp/2026.2:2776459744
```

## Loading the uenv for development
To load the uenv, run

```sh
uenv start cp2k-openmp/2026.2:2776459744 --view=develop
```

All relevant libraries will be available in $PATH/$LD_LIBRARY_PATH, such that CMake can find them. In 
case one needs to examine them, they can be found in `/user-environment/linux-neoverse_v2`.

## Compiling CP2K with OpenMP offloading
To compile CP2K, cd into the source code, and run the following:

```sh
mdkri build && cd build

gcc_nvptx_path=$(dirname $(which aarch64-unknown-linux-gnu-accel-nvptx-none-gcc))

CC=mpicc CXX=mpic++ FC=mpifort cmake \
    -GNinja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCP2K_USE_MPI=ON \
    -DCP2K_USE_ELPA=ON \
    -DCP2K_USE_SPLA=ON \
    -DCP2K_USE_COSMA=ON \
    -DCP2K_USE_FFTW3=ON \
    -DCP2K_USE_LIBXC=ON \
    -DCP2K_USE_LIBINT2=ON \
    -DCP2K_USE_ACCEL=CUDA \
    -DCMAKE_CUDA_ARCHITECTURES=90 \
    -DCMAKE_INSTALL_PREFIX=$(pwd) \
    -DCMAKE_C_FLAGS="${CMAKE_C_FLAGS} -foffload=nvptx-none -B${gcc_nvptx_path}" \
    -DCMAKE_CXX_FLAGS="${CMAKE_CXX_FLAGS} -foffload=nvptx-none -B${gcc_nvptx_path}" \
    -DCMAKE_Fortran_FLAGS="${CMAKE_Fortran_FLAGS} -foffload=nvptx-none -B${gcc_nvptx_path}" \
    -DCMAKE_EXE_LINKER_FLAGS="${CMAKE_EXE_LINKER_FLAGS} -fopenmp -foffload=nvptx-none" \
    -DCMAKE_SHARED_LINKER_FLAGS="${CMAKE_SHARED_LINKER_FLAGS} -fopenmp -foffload=nvptx-none" \
    ..

ninja -j 32
```
After modification of the source code, it is sufficient to run `ninja -32` to recompile the changes. Note 
that it is necessary to pass all these compiler and linker flags to CMake.

## Launching a calculation with Slurm
To launch a calculation, modify the following script accordingly:

```sh
#!/bin/bash -l

#SBATCH --job-name=test
#SBATCH --time=00:30:00
#SBATCH --nodes=1
#SBATCH --ntasks-per-core=1
#SBATCH --ntasks-per-node=32
#SBATCH --cpus-per-task=8
#SBATCH --account=<account>
#SBATCH --hint=nomultithread
#SBATCH --hint=exclusive
#SBATCH --uenv=cp2k-openmp/2026.2:2776459744
#SBATCH --view=develop

export MPICH_GPU_SUPPORT_ENABLED=1
export MPICH_MALLOC_FALLBACK=1
export OMP_NUM_THREADS=$((SLURM_CPUS_PER_TASK - 1))
export CUDA_CACHE_PATH="/dev/shm/$USER/cuda_cache"

export CP2K_DATA_DIR="path/to/cp2k/data"
export OMP_TARGET_OFFLOAD=MANDATORY

EXE="path/to/cp2k.psmp"
srun --cpu-bind=socket ./mps-wrapper.sh $EXE -i input.inp -o output.out
```

The `mps-wrapper.sh` can be found [here](https://docs.cscs.ch/running/slurm/#multiple-ranks-per-gpu). It 
launches MPS servers in an optimal way to enable multiple MPI ranks per GPU. It also ensures each MPI
rank only sees a single GPU, avoiding all OpenMP oflloading to be done in the same place. The script 
must be made an executable with `chmod +x mps-wrapper.sh`.

## Profiling with nsys
To get a profile with NVIDIA Nsight system, use the following line in your Slurm script:

```sh
srun --cpu-bind=socket ./mps-wrapper.sh nsys profile --sample=cpu --backtrace=fp $EXE -i input.inp -o output.out
```

To visualize the report, the easiest is to have a local nsys-ui installation, and use sshfs to access the files.
