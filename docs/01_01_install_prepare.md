# Preparation

## Supported Machines and Environments

The current Gen3 2D DRP is supported on `CentOS 7` and `AlmaLinux 9` (recommended).
The PFS 2D DRP pipeline is built upon the LSST pipeline, and requires that be installed. These pipelines use Python 3.11 features; lower python versions (python &ge; `3.9`) may be acceptable for using the `datamodel` package without the pipeline.

The examples below assume use of `bash`, although the pipeline supports other shells as well.

Support for additional systems may be added in the future.

## Setup Directory

---

We assume that the working directory is `$WORKDIR/(username)/`.

First, we will need to create a folder for necessary dependencies.
```
$ mkdir $WORKDIR/(username)/bin
```
Then, to set up the default environment variable, we should add the following line to `~/.bashrc`
```
$ export PATH=$WORKDIR/(username)/bin:$PATH
```

## Install Dependencies

---
    
**Step 1**: Fetch a tool of `yum`

There is a useful tool for installing dependencies in local environments: [user-yum](https://gitlab.com/caroff/user-yum.sh).
It installs applications conveniently without the root privilege in `CentOS 7`.
```
$ cd $WORKDIR/(username)/bin
$ git clone https://gitlab.com/caroff/user-yum.sh.git
```
You will find it handy if you change `Makefile` line 40 from
`INSTALL_FLAG_PREFIX := +` to `INSTALL_FLAG_PREFIX :=`

Then insert the following lines to `~/.bash_profile`:

``` bash
# Setting environment for $WORKDIR/(username)/bin/user-yum.sh/root
ROOT_D="$WORKDIR/(username)/bin/user-yum.sh/root"
export PATH=$ROOT_D/usr/sbin:$ROOT_D/usr/bin:$ROOT_D/bin:$PATH
L="/lib:/lib64:/usr/lib:/usr/lib64"
export LD_LIBRARY_PATH=$L:$ROOT_D/usr/lib:$ROOT_D/usr/lib64:$LD_LIBRARY_PATH
```

!!! note user-yum
    If you are the administrator of your local machine, you may skip this step and directly install the below dependencies with `sudo yum install ***`.

**(Skippable for the latest LSST v28) Step 2**: Install dependencies using `user-yum.sh`

**NOTE**: IMO, it's not worth including anything that's exclusively relevant to LSST versions earlier than 28.

```
$ cd user-yum.sh/
$ make blas bzip2-devel cmake freetype-devel gcc-c++ gcc-gfortran glib2-devel libuuid-devel libXt-devel ncurses-devel openssl-devel readline-devel zlib-dev
$ make install
```

**(Skippable for the latest LSST v28) Step 3**: Install OpenBLAS

OpenBLAS is needed for pfs_pipe2d; however it cannot be found in the default repositories of `CentOS 7`.
(From [this page](https://gist.github.com/bmmalone/1b5f9ff72754c7d4b313c0b044c42684))
```
$ cd $WORKDIR/(username)/bin
$ git clone https://github.com/OpenMathLib/OpenBLAS.git --single-branch --branch=v0.3.28
$ cd OpenBLAS && make FC=gfortran
```

!!! note
    If compilation fails due to missing dependencies, proceed with the LSST framework installation and retry this step afterward. You will probably manage to install the LSST platform and fail at the installation of PFS packages. Use command `source $WORKDIR/(username)/packages/stack_(LSST version)/loadLSST.bash` and then come back to repeat the last line listed above. The last step is setting up environment variables in `~/.bash_profile`

Then, to set up the default environment variable, we can add the following lines to `~/.bash_profile`
```
# for the OpenBLAS library
export LD_LIBRARY_PATH=$WORKDIR/(username)/bin/OpenBLAS:$LD_LIBRARY_PATH
export BLAS=$WORKDIR/(username)/bin/libopenblas.a
export ATLAS=$WORKDIR/(username)/bin/libopenblas.a
```

**Step 4**: Install git LFS

Git LFS must be installed to download large files from Git.
**QUESTION**: Isn't this installed in the conda environment installed with the LSST stack?

```
$ cd $WORKDIR/(username)/bin
$ wget https://github.com/git-lfs/git-lfs/releases/download/v3.5.1/git-lfs-linux-amd64-v3.5.1.tar.gz
$ tar xzf git-lfs-linux-amd64-v3.5.1.tar.gz
$ cd git-lfs-3.5.1
$ PREFIX=$WORKDIR/(username) ./install.sh
```