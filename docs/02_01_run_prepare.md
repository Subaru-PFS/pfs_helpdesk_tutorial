# Preparation of Running PFS 2D DRP

## Software Setup

---

First, we need to set up the 2D DRP environment:

```
$ source $WORKDIR/(username)/pfs/stack_28/loadLSST.bash
$ setup pfs_pipe2d -t current
$ setup -jr $WORKDIR/(username)/packages/fluxmodeldata-ambre-20230608-full
```

If you are using a public pipeline on a server, you will need to set up a local `drp_pfs_data` as follows:
```
$ setup -jr $WORKDIR/(username)/packages/drp_pfs_data
```

## Data Setup

---

!!! note
    This step is not necessary for processing **Science Data**. 

    Skip over this unless you’re running the **Integration Test** for yourself.

The integration test uses simulated data from the [drp_stella_data package](https://github.com/Subaru-PFS/drp_stella_data), which must be available locally.
Some of the headers in the simulated data are inaccurate (and have not been fixed upstream), so we need to check and fix them before running the integration test:

```
$ checkPfsRawHeaders.py --fix drp_stella_data/raw/PFSA*.fits
$ checkPfsConfigHeaders.py --fix drp_stella_data/raw/pfsConfig-*.fits
```

These scripts check the headers of raw data files (`checkPfsRawHeaders.py`) and pfsConfig files
(`checkPfsConfigHeaders.py`). They can be run without the `--fix` option to test for errors without fixing them.

## Repository Setup

---

The data repository is a directory containing data products and configuration files for the Gen3 middleware.
For this tutorial, we will store the data repository in the `$DATASTORE` directory on a local disk. However, the Gen3 middleware supports storing data products on remote storage and various cloud services.

The first step is to copy the default `butler.yaml`:

```
$ cp $OBS_PFS_DIR/gen3/butler.yaml $WORKDIR/(username)/data/butler.yaml
```

<!-- and we need to modify the `registry` section:

```
registry:
  db: postgresql+psycopg2://localhost:5435/drp
```

!!! note 
    Remember to update this file every time when there is an update of pipeline. -->

Next, create this directory and set up the Gen3 configuration files:

```
$ butler create $DATASTORE --seed-config $OBS_PFS_DIR/gen3/butler.yaml --dimension-config $OBS_PFS_DIR/gen3/dimensions.yaml --override
```

This specifies two configuration files that are important for the Gen3 middleware. `butler.yaml` contains the configuration
for the data `butler`, specifying the registry and how the pipeline will read and write the various data products.
Most of this can be ignored by the pipeline user, except for the `registry` section, which specifies the location of the
registry database. The default is to use a SQLite database in the repository directory (suitable for small-scale testing
without extensive parallelization), but this can be changed to use a PostgreSQL database for better performance. For
example, the following snippet configures the use of the `drp` database:

```
registry:
  db: postgresql+psycopg2://pfsa-db:5435/drp
```

`dimensions.yaml` contains the configuration for the dimensions of the data products. Dimensions are a very important
concept in the Gen3 middleware, as they are used to specify data products and control the parallelization levels of
different operations. Some important dimensions are:

- `instrument`: the instrument that produced the data. The datastore can contain data from multiple instruments,
but we probably only care about the PFS instrument. Usually this will be PFS for data from Subaru, but in the
case of the simulated data in the integration test it is `PFS-F` (the `F` stands for fake).
- `exposure`: the exposure number. This is the unique identifier for an exposure, what we would call a `visit` in
the Gen2 pipeline. In the Gen3 pipeline, visit is a separate dimension that groups multiple exposures together.
We may make use of this in the future, but for now you should use `exposure` (almost) everywhere you used to
use `visit`.
- `arm`: the spectrograph arm: `b`, `r`, `n`, or `m`.
- `spectrograph`: the spectrograph module: 1, 2, 3, or 4.
- `detector`: the detector number in the camera configuration. You shouldn’t need to worry about this dimension,
since it is specified by the combination of arm and `spectrograph`.
- `pfs_design_id`: the unique identifier for the PFS design. Note the spelling (not `pfsDesignId`), as this is a database table name.
- `dither`: the slit dither. This is a new dimension that was not present in the Gen2 pipeline, which is used in
creating fiber flats.
- `cat_id`: the unique identifier for the catalog. Again, notice the spelling (not `catId`). There is no `obj_id`
dimension, as the large number of objects would overwhelm the database.
- `skymap`, `tract`, `patch`: these specify the sky tessellation. They are currently unused in the pipeline, but might
be used in the future.

Next, register the instrument with `butler`. This also copies the camera configuration into the repository:

```
$ butler register-instrument $DATASTORE lsst.obs.pfs.PrimeFocusSpectrograph
```

The following commands import the defects files (and any other “curated” calibration files) from the `drp_pfs_data`
package (you’ll need to be using your own copy of `drp_pfs_data`, not the shared version):

```
$ makePfsDefects --mko
$ butler write-curated-calibrations $DATASTORE PFS
```
