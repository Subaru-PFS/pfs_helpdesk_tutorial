# Pipeline Installation

## Install Pipeline

---

The basic information of the PFS 2D DRP for this section includes:

1. LSST version: 26.0.2
3. pfs_pipe2d branch: gen3
4. python packages added: `numpy`, `scipy=1.10.1`(Due to the nan/inf issue in >1.11)

**Step 1**: We should fetch pfs_pipe2d Gen3:

```
$ cd $WORKDIR/(username)/
$ git clone http://github.com/Subaru-PFS/pfs_pipe2d --branch=gen3
```

!!! Skippable
    1. Comment out `install_pfs.sh` line 87 `setup pipe_drivers ${setup_args}`. This package seems discontinued and will not be installed by the script.
    2. Change `install_lsst.sh` line 58 from `LSST_VERSION=v26_0_0` to `LSST_VERSION=v26_0_2`
    3. Add `numpy scipy=1.10.1` to the end of `install_lsst.sh` line 43 (not necessary for now but for newer versions of LSST)

**Step 2**: We should check out to the lastest version:

```
$ cd pfs_pipe2d
$ git checkout $(git describe --tags `git rev-list --tags --max-count=1`)
```

**Step 3**: We should create the target folder and start the installation:
```
$ mkdir -p $WORKDIR/(username)/pfs/stack_26_0_2
$ cd $WORKDIR/(username)/pfs_pipe2d/bin
$ ./install_pfs.sh -t current -b gen3 $WORKDIR/(username)/pfs/stack_26_0_2
```

## Install Flux Model Data

---

**Step 1**: We should fetch the flux model data:

```
$ bash
$ mkdir -p $WORKDIR/(username)/source/
$ cd $WORKDIR/(username)/source/
$ wget https://hscdata.mtk.nao.ac.jp/hsc_bin_dist/pfs/fluxmodeldata-ambre-20230608.tar.gz
$ tar xzf fluxmodeldata-ambre-20230608.tar.gz -C .
```

**Step 2**: We should modify the file `$WORKDIR/(username)/pfs_pipe2d/fluxmodeldata-ambre-20230608/scripts/makespectra.py`, and add the following line to the `import` section.

```
from numpy.lib import recfunctions
```

**Step 3**: We can start the installation process

```
$ cd $WORKDIR/(username)/source/fluxmodeldata-ambre-20230608
$ ./install.py --prefix=/$WORKDIR/(username)/packages/
```

## (Optional) Individual Users: Install `drp_pfs_data` Package

---

If the PFS pipeline was installed for all users on a server in a public directory, e.g., `$WORKDIR/pfs/`, then for individual users, a local version of `drp_pfs_data` package -- other than the one included in the `pfs_pipe2d` installation above -- is needed.

We can install the local `drp_pfs_data` package as follows:
 
```
$ cd $WORKDIR/(username)/packages/
$ git clone https://github.com/Subaru-PFS/drp_pfs_data.git --single-branch
$ cd drp_pfs_data
$ git fetch --tags
$ git checkout $(git describe --tags `git rev-list --tags --max-count=1`)
```