# Preparation

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
It installs applications conveniently without the root privilege in CentOS7.
```
$ cd $WORKDIR/(username)/bin
$ git clone https://gitlab.com/caroff/user-yum.sh.git
```
You will find it handy if you change `Makefile` line 40 from
`INSTALL_FLAG_PREFIX := +` to `INSTALL_FLAG_PREFIX :=`

!!! note user-yum
    If you are the administrator of your local machine, you may skip this step and directly install the below dependencies with `sudo yum install ***`.

**Step 2**: Install dependencies using `user-yum.sh`

```
$ cd user-yum.sh/
$ make blas bzip2-devel cmake freetype-devel gcc-c++ gcc-gfortran glib2-devel libuuid-devel libXt-devel ncurses-devel openssl-devel readline-devel zlib-dev
$ make install
```

**Step 3**: Install OpenBLAS

OpenBLAS is needed for pfs_pipe2d; however it cannot be found in the default repositories of CentOS7.
(From [this page](https://gist.github.com/bmmalone/1b5f9ff72754c7d4b313c0b044c42684))
```
$ cd $WORKDIR/(username)/bin
$ git clone https://github.com/OpenMathLib/OpenBLAS.git --single-branch --branch=v0.3.28
$ cd OpenBLAS && make FC=gfortran
```

!!! note
    If compilation fails due to missing dependencies, proceed with the LSST framework installation and retry this step afterward. You will probably manage to install the LSST platform and fail at the installation of PFS packages. Use command `source $WORKDIR/(username)/packages/stack_26_0_2/loadLSST.bash` and then come back to repeat the last line listed above. The last step is setting up environment variables in `~/.bash_profile`

Then, to set up the default environment variable, we can add the following lines to `~/.bash_profile`
```
# for the OpenBLAS library
export LD_LIBRARY_PATH=$WORKDIR/(username)/bin/OpenBLAS:$LD_LIBRARY_PATH
export BLAS=$WORKDIR/(username)/bin/libopenblas.a
export ATLAS=$WORKDIR/(username)/bin/libopenblas.a
```

**Step 4**: Install git LFS

Git LFS must be installed to download large files from Git.

```
$ cd /$WORKDIR/(username)/bin
$ wget https://github.com/git-lfs/git-lfs/releases/download/v3.5.1/git-lfs-linux-amd64-v3.5.1.tar.gz
$ tar xzf git-lfs-linux-amd64-v3.5.1.tar.gz
$ cd git-lfs-3.5.1
$ PREFIX=$(dirname $(command -v git)) ./install.sh
```