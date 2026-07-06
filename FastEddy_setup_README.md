0. Remove Kitware by comment the line in:
sudo nano /etc/apt/sources.list

1. Install openmpi:
```
sudo apt update
sudo apt install build-essential openmpi-bin libopenmpi-dev libnetcdf-dev
```

2. Install Cuda toolkit 12.8:

```
sudo apt-key del 7fa2af80
wget https://developer.download.nvidia.com/compute/cuda/repos/wsl-ubuntu/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
sudo apt install cuda-toolkit-12-8
```

3. Install NVidia HPC-SDK, e.g. version 26.5:

```
wget https://developer.download.nvidia.com/hpc-sdk/26.5/nvhpc_2026_265_Linux_x86_64_cuda_13.2.tar.gz

tar -xpf nvhpc_2026_265_Linux_x86_64_cuda_13.2.tar.gz

cd nvhpc_2026_265_Linux_x86_64_cuda_13.2/

sudo ./install
```

After done, one can remove
```
rm -rf nvhpc_2026_265_Linux_x86_64_cuda_13.2

rm nvhpc_2026_265_Linux_x86_64_cuda_13.2.tar.gz
```

Then choose `1` as in `1  Single system install`.

4. Change env before building:

Modify .bashrc then `source .bashrc`:
```
export CUDA_HOME=/usr/local/cuda
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH

# NVIDIA HPC SDK 26.5
export NVHPC=/opt/nvidia/hpc_sdk/Linux_x86_64/26.5
export MPI_HOME=$NVHPC/comm_libs/mpi

# Required by NVIDIA MPI wrappers: provides nvc / nvc++.
export PATH=$NVHPC/compilers/bin:$PATH

# Select NVIDIA's CUDA-aware MPI wrapper.
export PATH=$MPI_HOME/bin:$PATH

# Runtime libraries.
export LD_LIBRARY_PATH=$NVHPC/compilers/lib:$LD_LIBRARY_PATH
export OPAL_PREFIX=$MPI_HOME

# Disable UCX CUDA memory-type cache
export UCX_MEMTYPE_CACHE=n
```

5. In Makefile, Replace

```
OTHER_INCLUDES = -I${NCAR_ROOT_MPI}/include
```

by

```
TEST_CC = mpicc
TEST_CU_CC = nvcc

CUDA_HOME ?= /usr/local/cuda
CUDA_INC  = $(CUDA_HOME)/targets/x86_64-linux/include
CUDA_LIB  = $(CUDA_HOME)/targets/x86_64-linux/lib

MPI_INC = $(shell mpicc --showme:incdirs | sed 's|^|-I|; s| | -I|g')
MPI_LIB = $(shell mpicc --showme:libdirs | sed 's|^|-L|; s| | -L|g')
...

ifeq ($(strip $(NCAR_ROOT_MPI)),)
OTHER_INCLUDES = \
	$(MPI_INC) \
	-I$(CUDA_INC)
else
OTHER_INCLUDES = \
	-I${NCAR_ROOT_MPI}/include \
	-I$(CUDA_INC)
endif

TEST_LDFLAGS = -L. $(MPI_LIB) -L$(CUDA_LIB)
TEST_CU_LDFLAGS = -L. $(MPI_LIB) -L$(CUDA_LIB)

TEST_LIBS = -lm -lmpi -lstdc++ -lcurand -lcudart -lnetcdf
TEST_CU_LIBS = -lm -lmpi -lcudart
```

6. Build FastEddy inside `FastEddy/SRC/FEMAIN`:

```
make clean
make WITH_GAD=1 WITH_URBAN=1 V=1
```

7. Verify:

```
ls -l FastEddy
ldd FastEddy | egrep 'cuda|curand|mpi|netcdf'
```

and run an example:

```
mpirun -n 4 ~/Projects/FastEddy/SRC/FEMAIN/FastEddy ~/Projects/FastEddy/tutorials/examples/Example01_NBL.in
```
