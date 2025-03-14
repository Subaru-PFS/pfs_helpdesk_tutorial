# Build Calibration Frames

After the preparation, we will need to build calibration data before running the processing of science data.

First, let's assume we have default variables as in the following example, where a user works in the public directory `$WORKDIR/pfs/` and using a publicly install pipeline: 

```
DATASTORE="$WORKDIR/pfs/data/datastore"
DATADIR="$WORKDIR/pfs/data"
CORES=16
INSTRUMENT="lsst.obs.pfs.PrimeFocusSpectrograph"
RERUN="u/(username)/"
```

In this case, you may want to set up the rerun directory specified by your username so that multiple users won't mix things up.

## Build Bias

---

Next, we’ll build the calibration products, starting from the bias frames:

```
pipetask run \
--register-dataset-types \                                # register the dataset types from the pipeline
-j $CORES \                                               # number of cores to use in parallel
-b $DATASTORE \                                           # datastre directory to use
--instrument lsst.obs.pfs.PrimeFocusSpectrograph \        # the instrument PFS
-i PFS/raw/sps,PFS/calib \                                # input collection (comma-separated)
-o "$RERUN"/bias \                                        # output CHAINED collection
-p $DRP_STELLA_DIR/pipelines/bias.yaml \                  # pipeline configuration file to use
-d "instrument='PFS' AND exposure.target_name = 'BIAS'" \ # or, for example: -d "visit IN (123456..123466)" \
--fail-fast \                                             # immediately stop the ingestion process if error
-c isr:doCrosstalk=True                                   # (optional) turn on the crosstalk correction 
```

The `pipetask run` command is used to run a pipeline. 
A task is an operation within the pipeline, characterized by a set of dimensions that define the level at which it parallelizes, and a set of inputs and outputs. An instance of a task running on a single set of data at its parallelization level is called a "quantum". 
A pipeline is built from the "quantum graph", which tracks the inputs and outputs between various tasks. 
When you run a pipeline with `pipetask run`, it first builds the pipeline and reports the number of quanta that will be run for each task:

```
lsst.ctrl.mpexec.cmdLineFwk INFO: QuantumGraph contains 12 quanta for 2 tasks, graph ID: '1726845383.   6842682-77840''
Quanta     Tasks    
------ -------------
    10          isr
     2 cpBiasCombine
```

The bias pipeline has only two tasks. 
In this case, they are operating on 5 exposures, each with `b` and `r` arms, so there are 10 `isr` quanta (instrument signature removal from each camera image) and 2 `cpBiasCombine` quanta (combining the bias frames from each of the cameras). 
The summary for a more complicated pipeline (running the full science pipeline on 17 exposures) is shown later.

- `-j` option specifies the number of cores to use in parallel.

- `-b` option specifies the datastore to use.

- `--instrument` option specifies the instrument. The proper PFS is `lsst.obs.pfs.PrimeFocusSpectrograph`

- `-i` option specifies the input collections (comma-separated). In this case, we are using the raw data and the calibration data (for the defects). Later we’ll add other collections as we need them.

- `-o` option specifies an output `CHAINED` collection. The pipeline will write the output datasets to a `RUN` collection named after this, with a timestamp appended (e.g., `$RERUN/bias/20240918T181715Z`), all chained together in the nominated output collection.

- `-p` option specifies the pipeline configuration file to use. This is a `YAML` file in `drp_stella/pipelines` that describes the pipeline to run. A pipeline is composed of multiple tasks, each operating on a (potentially different) set of dimensions. The pipeline configuration can also specify configuration overrides for each task, including different dataset names to use as connections between the tasks (useful for providing slightly different versions of the same dataset; there’ll be an example of this later).

- `-d` option specifies the data selection query. The query syntax is similar to the `WHERE` clause in SQL, with some extensions. In this case, we are selecting all the exposures that have a target name of BIAS and are from the PFS instrument. Strings must be quoted with single quotes (`'`). Ranges can be specified, like exposure `IN (12..34:5)`, which means all exposures from 12 to 34 (inclusive) in steps of 5.
The `exposure` dimension can be used directly to mean the exposure identifier, but also has a variety of additional fields that can be used, including:
  > - `exposure.exposure_time`: exposure time in seconds
  > - `exposure.observation_type`: type of observation (e.g., `BIAS`, `DARK`, `FLAT`, `ARC`)
  > - `exposure.target_name`: target name
  > - `exposure.science_program`: science program name
  > - `exposure.tracking_ra`, `tracking_dec`: boresight position (ICRS)
  > - `exposure.zenith_angle`: zenith angle in degrees
  > - `exposure.lamps`: comma-separated list of lamps that were on
  > - Other dimensions can also be used, for example: `exposure IN (12..34:5) AND arm = 'r' AND spectrograph = 3`.

- `-c` option provides configuration overrides for the pipeline<sup>[1](#diff_gen2_c)</sup>. In this case, we are turning on the crosstalk correction.

- `--register-dataset-types` option is used to register the dataset types from the pipeline in the butler registry. 
It is only necessary to run this once for each pipeline, and then it can be dropped for future runs of the same pipeline.

Some additional helpful options when debugging are:

- `--skip-existing-in <COLLECTION>`: don’t re-produce a dataset if it’s present in the specified collection.
This is helpful when you want to pick up from where a previous run stopped. 
Usually the `<COLLECTION>` specified here is the same as the output collection.

- `--clobber-outputs`: clobber any existing datasets for a task (usually logging or metadata by-products of running the task).

- `--pdb`: drop into the python debugger on an exception. 
This won’t work with parallel processing, so check that you’re not also using `-j`.

The above three options (used together) are very useful when debugging a python exception in a pipeline run.

Once the pipeline has run and produced the bias frame, we need to certify the calibration products:

```
$ butler certify-calibrations $DATASTORE "$RERUN"/bias PFS/calib bias --begin-date 2000-01-01T00:00:00 --end-date 2050-12-31T23:59:59
```

This command tells the butler to certify the bias datasets in the `$RERUN/bias` collection as calibration products in the `PFS/calib` calib collection. 
The `--begin-date` and `--end-date` options specify the validity range of the calibration products.

To manage calibrations, it may be necessary to certify and decertify individual datasets.
This capability is not available with LSST’s command-line tools, but we have some scripts that can do this. Here are some examples from working on real Subaru data:

```
$ butlerDecertify.py /work/datastore PFS/calib dark --begin-date 2024-08-24T00:00:00 --id instrument=PFS arm=r spectrograph=2
$ butlerDecertify.py /work/datastore PFS/calib dark --begin-date 2024-05-01T00:00:00 --end-date 2024-08-23T23:59:59 --id instrument=PFS arm=r spectrograph=2
$ butlerCertify.py /work/datastore price/pipe2d-1036/dark/run16 PFS/calib dark --begin-date 2024-05-01T00:00:00 --id instrument=PFS arm=r spectrograph=2
```

---

> **Warning**: Certifying a dataset as a calibration product tags it in the database as a calibration product and associates it with a validity timespan. It does not copy the dataset: the dataset is still a part of the `$RERUN/bias/<timestamp> RUN` collection, and removing that collection will remove the calibration dataset from the datastore.

---

However, that RUN collection also contains a bunch of intermediate datasets which are unnecessarily consuming space, in particular the `biasProc` datasets (which are the outputs of running the isr task in the bias pipeline). We can remove these with the following command:

```
$ butlerCleanRun.py $DATASTORE $RERUN/bias/* biasProc
```

This will leave the `$RERUN/bias/<timestamp>` collection containing only the bias dataset and some other small metadata datasets. Note that our pipetask command specifies an output collection of `$RERUN/bias`, but we're specifying `$RERUN/bias/*` for the `butlerCleanRun.py` command, which will delete all the timestamped `RUN` collections in the `$RERUN/bias CHAINED` collection.

You can also use the `butler remove-runs` command to completely remove `RUN` collections and `butler remove-collections` to remove `CHAINED` collections.


## Build Dark

---

With the bias calibration product built and certified, we can move on to the dark, which follow the same pattern:

First run the builder:
```
pipetask run \
--register-dataset-types -j $CORES -b $DATASTORE \
--instrument lsst.obs.pfs.PrimeFocusSpectrograph \
-i PFS/raw/all,PFS/calib \
-o "$RERUN"/dark \
-p $DRP_STELLA_DIR/pipelines/dark.yaml \
-d "instrument='PFS' AND exposure.target_name = 'DARK'" \
--fail-fast \
-c isr:doCrosstalk=True
```

Then, certify the products:
```
$ butler certify-calibrations $DATASTORE "$RERUN"/dark PFS/calib dark --begin-date 2000-01-01T00:00:00 --end-date 2050-12-31T23:59:59
$ butlerCleanRun.py $DATASTORE $RERUN/dark/* darkProc
```

## Build Flat

---

Building Flats follows the same pattern. 

First run the builder:
```
pipetask run \
--register-dataset-types -j $CORES -b $DATASTORE \
--instrument lsst.obs.pfs.PrimeFocusSpectrograph \
-i PFS/raw/all,PFS/calib \
-o "$RERUN"/flat \
-p $DRP_STELLA_DIR/pipelines/flat.yaml \
-d "instrument='PFS' AND exposure.target_name = 'FLAT'" \
--fail-fast \
-c isr:doCrosstalk=True
```

Then, certify the products:
```
$ butler certify-calibrations $DATASTORE "$RERUN"/flat PFS/calib flat --begin-date 2000-01-01T00:00:00 --end-date 2050-12-31T23:59:59
$ butlerCleanRun.py $DATASTORE $RERUN/flat/* flatProc
```

## Build Detector Map

---

The bias, dark, and flat frames characterize the detector, so now it’s time to determine the `detectorMap`. We first bootstrap a
detectorMap from an arc and quartz:

```
pipetask run \
--register-dataset-types -j $CORES -b $DATASTORE \
--instrument lsst.obs.pfs.PrimeFocusSpectrograph \
-i PFS/raw/all,PFS/raw/pfsConfig,PFS/calib 
-o "$RERUN"/bootstrap \
-p $DRP_STELLA_DIR/pipelines/bootstrap.yaml' \
-d "instrument='PFS' AND exposure IN (11,22)" \
--fail-fast \
-c isr:doCrosstalk=True
```

Then, certify the products:
```
$ butler certify-calibrations $DATASTORE "$RERUN"/bootstrap PFS/bootstrap detectorMap_bootstrap --begin-date 2000-01-01T00:00:00 --end-date 2050-12-31T23:59:59
$ butlerCleanRun.py $DATASTORE $RERUN/bootstrap/* postISRCCD
```

Here, we have added the `PFS/raw/pfsConfig` collection to the input since we need the `pfsConfig` files to determine which fibers are illuminated. Note that the arc and quartz are both specified as inputs in the same `-d` option. 
The Gen3 middleware does not support multiple `-d` options to specify them independently, but the task can determine which is which from the `lamps` field in the exposure. 
The bootstrap pipeline writes a `detectorMap_bootstrap` dataset for each camera, and we’re certifying that in the `PFS/bootstrap` collection (so it’s independent of the best-quality detectorMaps we’ll certify in `PFS/calibs`).

When working with real data, it will probably be necessary to run the bootstrap pipeline on each camera separately, so that different `-c bootstrap:spectralOffset=<WHATEVER>` values can be used for each camera.

Now we have a rough detectorMap, we can refine it and create the proper detectorMap:

```
pipetask run \
--register-dataset-types -j $CORES -b $DATASTORE \
--instrument lsst.obs.pfs.PrimeFocusSpectrograph \
-i PFS/raw/all,PFS/raw/pfsConfig,PFS/bootstrap,PFS/calib \
-o "$RERUN"/detectorMap \
-p '$DRP_STELLA_DIR/pipelines/detectorMap.yaml' \
-d "instrument='PFS' AND exposure.target_name = 'ARC'" \
-c isr:doCrosstalk=True \
-c measureCentroids:connections.calibDetectorMap=detectorMap_bootstrap \
-c fitDetectorMap:connections.slitOffsets=detectorMap_bootstrap.slitOffsets \
--fail-fast
```

Then, certify the products:
```
$ certifyDetectorMaps.py INTEGRATION $RERUN/detectorMap PFS/calib --instrument PFS --begin-date 2000-01-01T00:00:00 --end-date 2050-12-31T23:59:59
$ butlerCleanRun.py $DATASTORE $RERUN/detectorMap/* postISRCCD
```

Here, we have modified two connections in the pipeline. The `measureCentroids` task’s `calibDetectorMap` input is a detectorMap that provides the position at which to measure the centroids of the arc lines. 
Usually this is set to the calibration detectorMap (detectorMap_calib), but we don’t have one of those yet. 
Instead, we will configure this to use the bootstrap detectorMap (`detectorMap_bootstrap`) instead; notice also that we’re including the `PFS/ boostrap` collection in the input. 
Similarly, the `fitDetectorMap` task’s `slitOffsets` input is set to use the slit offsets from the bootstrap detectorMap.

The detectorMap pipeline writes a `detectorMap_candidate` dataset for each camera. 
The `certifyDetectorMaps.py` script is used to certify the detectorMap datasets instead of the usual `butler certify-calibrations` command. 
This script copies the `detectorMap_candidate` as a `detectorMap_calib` and certifies it.

## Build Fiber Profile

---

Fiber profiles can be built in two different ways. 
The `fitFiberProfiles` pipeline is equivalent to the Gen2 `reduceProfiles` script: it fits a profile to multiple exposures simultaneously. 
The `measureFiberProfiles` pipeline is equivalent to the Gen2 `constructFiberProfiles` script: it measures the profile from a single exposure. Here’s how you run them:

```
# fitFiberProfiles:
defineFiberProfilesInputs.py $DATASTORE PFS integrationProfiles --bright 26 --bright 27
pipetask run \
--register-dataset-types -j $CORES -b $DATASTORE \
--instrument lsst.obs.pfs.PrimeFocusSpectrograph \
-i PFS/raw/all,PFS/fiberProfilesInputs,PFS/raw/pfsConfig,PFS/calib \
-o "$RERUN"/fitFiberProfiles \
-p '$DRP_STELLA_DIR/pipelines/fitFiberProfiles.yaml \
-d "profiles_run = 'integrationProfiles'" \
-c fitProfiles:profiles.profileSwath=2000 \
-c fitProfiles:profiles.profileOversample=3 \
--fail-fast

# measureFiberProfiles:
pipetask run \
--register-dataset-types -j $CORES -b $DATASTORE \
--instrument lsst.obs.pfs.PrimeFocusSpectrograph \
-i PFS/raw/all,PFS/raw/pfsConfig,PFS/calib \
-o "$RERUN"/measureFiberProfiles \
-p '$DRP_STELLA_DIR/pipelines/measureFiberProfiles.yaml' \
-d "instrument='PFS' AND exposure.target_name IN ('FLAT_ODD', 'FLAT_EVEN')" \
-c isr:doCrosstalk=True \
--fail-fast

# certify the fiberProfile product
butler certify-calibrations $DATASTORE "$RERUN"/fitFiberProfiles PFS/calibfiberProfiles --begin-date 2000-01-01T00:00:00 --end-date 2050-12-31T23:59:59
butlerCleanRun.py $DATASTORE $RERUN/fitFiberProfiles/* postISRCCD
```

Because it involves multiple groups of exposures, the `fitFiberProfiles` pipeline is a bit more complicated and requires defining the inputs to the pipeline ahead of time. 
The `defineFiberProfilesInputs.py` script is used to define the inputs for the different groups of exposures. 
When working on real data, we typically have four groups of several exposures each, and each group contains “bright” (select fibers deliberately exposed) and “dark” (all fibers hidden) exposures. 
In the integration test, we only have two groups with a single bright exposure each, and no dark exposures. 
For real data, the command might look like:

```
defineFiberProfilesInputs.py $DATASTORE PFS run18_brn \
--bright 113855..113863 --dark 113845..113853 \
--bright 113903..113911 --dark 113893..113901 \
--bright 114190..114198 --dark 114180..114188 \
--bright 114238..114246 --dark 114228..114236
```

This creates a profiles_run dimension value and associates those exposures with it. 
A file describing the roles of the exposures is written in the` <instrument>/fiberProfilesInputs` collection, so this must be included in the inputs for the `fitFiberProfiles` pipeline. 
We can use the profiles_run value in the data selection query, as that is linked to all the required exposures.

## Build Fiber Norm

---

Note that in the Gen3 pipeline, `fiberProfiles` do not include quartz spectrum normalization. The quartz spectrum used for normalization is supplied by the `fiberNorms`:

```
pipetask run \
--register-dataset-types -j $CORES -b $DATASTORE \
--instrument lsst.obs.pfs.PrimeFocusSpectrograph \
-i PFS/raw/all,PFS/raw/pfsConfig,PFS/calib \
-o "$RERUN"/fiberNorms \
-p '$DRP_STELLA_DIR/pipelines/fiberNorms.yaml' \
-d "instrument='PFS' AND exposure.target_name = 'FLAT' AND dither = 0.0" \
-c isr:doCrosstalk=True \
-c reduceExposure:doApplyScreenResponse=False \
-c reduceExposure:doBlackSpotCorrection=False \
--fail-fast

butler certify-calibrations $DATASTORE "$RERUN"/fiberNorms PFS/calib fiberNorms_calib --begin-date 2000-01-01T00:00:00 --end-date 2050-12-31T23:59:59
butlerCleanRun.py $DATASTORE $RERUN/fiberNorms/* postISRCCD
```

The `fiberNorms` pipeline combines the extracted spectra from multiple quartz exposures, and writes the output as
`fiberNorms_calib`

---

<a name="diff_gen2_c">1</a> 
Note the difference in syntax from Gen2: each configuration override requires a separate `-c` option, and the overrides include a colon (`:`) between the task name and the configuration parameter name. 