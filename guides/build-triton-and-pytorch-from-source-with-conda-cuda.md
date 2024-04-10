# Build Triton and Pytorch from source in a conda environment with a custom CUDA version

## Why?

This was written to use Triton with a CUDA version for which no prebuilt packages were available. It's also useful to develop on or use the latest Pytorch and Triton versions using just a conda environment (e.g. to test different cuda versions, or for use in a system that only supports installing dependencies using conda due to no root access, etc). The setup uses CUDA 11.8, but should work for other versions as well.

The guide provides two possible options:
  1. both Triton and Pytorch are built from source and
  2. only Triton is built from source.

The initial working directory is assumed to be `/scratch`.

## Install dependencies

First, a conda environment with the triton (and pytorch) dependencies needs to be created.

    conda create -f conda-envs/triton-cu118.yaml

Note: this installs the CUDA 11.8 libraries, `triton-cu118.yaml` needs to be changed if a different CUDA version is required.

Activate the conda environment:

    conda activate triton-cu118

Now, clone the Pytorch sources (and optionally checkout a desired revision):

    git clone --recursive https://github.com/pytorch/pytorch

Next, clone the Triton sources and checkout the revision compatible with the cloned Pytorch sources:

```
git clone https://github.com/openai/triton
cd triton
git checkout $(cat ../pytorch/.ci/docker/ci_commit_pins/triton.txt)
```

## Install Pytorch (Option 2)

If Pytorch should not be built from source, the prebuilt nightly package needs to be installed now. The package version should correspond to the cloned Pytorch sources.

    conda install pytorch torchvision torchaudio pytorch-cuda=11.8 -c pytorch-nightly -c nvidia -c conda-forge

Note: `torchvision` and `torchaudio` are optional. The `pytorch-cuda` version needs to be changed if a different CUDA version is required.

## Build LLVM

Before Triton can be built, LLVM has to be built. First checkout the LLVM sources compatible with the cloned Triton sources:

```
cd /scratch
git clone --recursive https://github.com/llvm/llvm-project
cd llvm-project
git checkout $(cat ../triton/cmake/llvm-hash.txt)
```

Next, the libraries installed in the conda environment are made visible to the build process:

```
export LD_LIBRARY_PATH=$CONDA_PREFIX/lib/stubs:$CONDA_PREFIX/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$LD_LIBRARY_PATH
```

Now, the build files can be generated using `cmake` in a new directory:

```
export LLVM_BUILD_DIR=/scratch/llvm-project/build
mkdir build
cd build
cmake -G Ninja ../llvm -DCMAKE_BUILD_TYPE=Release -DCMAKE_LIBRARY_PATH="$CONDA_PREFIX/lib" -DLLVM_ENABLE_PROJECTS="mlir;llvm" -DLLVM_BUILD_EXAMPLES=ON -DLLVM_BUILD_UTILS=ON -DLLVM_BUILD_TOOLS=ON -DLLVM_ENABLE_ASSERTIONS=ON -DLLVM_ENABLE_RTTI=ON -DLLVM_INSTALL_UTILS=ON -DLLVM_TARGETS_TO_BUILD="host;NVPTX;AMDGPU" -DMLIR_ENABLE_CUDA_RUNNER=ON -DMLIR_INCLUDE_INTEGRATION_TESTS=ON
```

Finally, the build is started:

    ninja

## Build Triton

With LLVM built, Triton can be built from the python package directory.

    cd /scratch/triton/python

Again, the binaries and libraries installed in the conda environment need to be made visible to the build process:

```
export TRITON_PTXAS_PATH=$CONDA_PREFIX/bin/ptxas
export TRITON_CUOBJDUMP_PATH=$CONDA_PREFIX/bin/cuobjdump
export TRITON_NVDISASM_PATH=$CONDA_PREFIX/bin/nvdisasm
```

Similarly, the location of the LLVM build must be declared:

```
export LLVM_BUILD_DIR=/scratch/llvm-project/build
export LLVM_INCLUDE_DIRS=$LLVM_BUILD_DIR/include
export LLVM_LIBRARY_DIR=$LLVM_BUILD_DIR/lib
export LLVM_SYSPATH=$LLVM_BUILD_DIR
export LD_LIBRARY_PATH=$LLVM_LIBRARY_DIR:$CONDA_PREFIX/lib/stubs:$CONDA_PREFIX/lib
export LIBRARY_PATH=$LD_LIBRARY_PATH
```

Now, Triton can be built:

    pip install -e .

Note: the `-e` is optional.

## Build Pytorch (Option 1)

With Triton built, it is now possible to build Pytorch from source. Can be skipped if the prebuilt packages have already been installed above.

    cd /scratch/pytorch

Install the requirements:

    pip install -r requirements.txt

Optionally build Pytorch using the new ABI:

    export _GLIBCXX_USE_CXX11_ABI=1

Again, link the conda environment to the build process:

```
export LD_LIBRARY_PATH=$LLVM_LIBRARY_DIR:$CONDA_PREFIX/lib/stubs:$CONDA_PREFIX/lib
export LIBRARY_PATH=$LD_LIBRARY_PATH
export CMAKE_PREFIX_PATH=$CONDA_PREFIX
```

Build Pytorch:

    python setup.py develop

## Test

To test the Triton installation:

    python /scratch/triton/python/tutorials/01-vector-add.py

To test Pytorch and Triton working together:

    python /scratch/pytorch/test/inductor/test_triton_wrapper.py

