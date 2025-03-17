# Process Science Data

## Basic Information

---

With the calibration products built, we can now process the science data. There are a few pipelines available:

- `reduceExposure`: process an exposure through merging arms, producing `postISRCCD`, `pfsArm`, `lines`, `detectorMap`, `pfsMerged`, `sky1d`, and `fiberNorms`. 
This can be used to process quartz exposures (or sky exposures when flux calibration is not wanted). 
The `fiberNorms` dataset is only produced for quartz exposures; Unlike the `fiberNorms_calib` product, this is a residual normalization equal to the ratio of the observed quartz spectrum to the `fiberNorms_calib` spectrum (after applying screen responses and other corrections).
  
- `calibrateExposure`: adds the flux calibration to `reduceExposure`, producing `pfsFluxReference`, `fluxCal` and `pfsCalibrated`. 
This can be used to process single sky exposures. This is not demonstrated below, but its use is similar to that for `reduceExposure`.

- `science`: adds the spectral coaddition, producing `pfsCoadd`. 
This can be used to process multiple sky exposures together.

## Define Collections

---

Because we need to be able to distinguish coadds formed from different combinations of exposures, it’s necessary to define the inputs to the coaddition before running the `science` pipeline. 
This is not required for the `reduceExposure` or `calibrateExposure` pipelines, but defining the inputs can provide a convenient way to reference them.. 
The integration test defines two combinations:

```
$ defineCombination.py $DATASTORE PFS object --where "exposure.target_name = 'OBJECT'"
```

A combination can be defined with a `--where` option, which takes a query string like for the `-d` option of `pipetask run`. 
Alternatively, a combination can be defined by simply listing the exposure identifiers:

```
$ defineCombination.py $DATASTORE PFS someExposures 123 124 125
```

## Process Science Data

---

Now we can run the pipeline process a specific single exposure:
```
# Single Exposure
pipetask run \
--register-dataset-types -j $CORES -b $DATASTORE \
--instrument $INSTRUMENT \
-i PFS/raw/all,PFS/raw/pfsConfig,PFS/calib \
-o "$RERUN"/reduceExposure \
-p '$DRP_STELLA_DIR/pipelines/reduceExposure.yaml' \
-d "combination = 'object'" \
--fail-fast \
-c 'reduceExposure:targetType=[SCIENCE, SKY, FLUXSTD]'
```

Alternatively, you can run the science pipeline for an entire data collection:

```
# Science Pipeline
pipetask run \
--register-dataset-types -j $CORES -b $DATASTORE \
--instrument $INSTRUMENT \
-i PFS/raw/all,PFS/raw/pfsConfig,PFS/calib \
-o "$RERUN"/science \
-p '$DRP_STELLA_DIR/pipelines/science.yaml' \
-d "combination = 'object'" \
--fail-fast \
-c isr:doCrosstalk=True \
-c fitFluxCal:fitFocalPlane.polyOrder=4 \
-c 'reduceExposure:targetType=[SCIENCE, SKY, FLUXSTD]'
```

Notice that in the first case we’re running the `reduceExposure` pipeline, selecting the `object` combinations that we defined earlier. The `science` pipeline is similar.

Note that we do not have to run the `reduceExposure` pipeline before we run the `science` pipeline (a single command is sufficient to run the entire pipeline): the `science` pipeline knows how to produce all the necessary intermediate datasets itself, and the above two commands are completely independent: they do not share any intermediate datasets. 
However, we could have first run the `reduceExposure` pipeline and then fed its outputs into the `science` pipeline by including `$RERUN/reduceExposure` in the list of input collections for the `science` pipeline.

## Retrieve Data Products

---

There are some important differences in the data products produced by the Gen3 pipeline compared to the Gen2 pipeline. For the sake of efficiency (both in terms of processing time and reduced file numbers), the single-spectrum Gen2 products (`pfsSingle` and `pfsObject`) are written as multiple-spectrum Gen3 products per `cat_id` (`pfsCalibrated` and `pfsCoadd`). The equivalent of a `pfsObject` can be retrieved directly from the `pfsCoadd` dataset:

``` python
from lsst.daf.butler import Butler
butler = Butler.from_config($DATASTORE, collection=["$RERUN/object"])
pfsObject = butler.get("pfsCoadd.single", cat_id=1, combination="object", parameters=dict(objId=55))
```

Note that the `objId` needs to be specified in the `parameters` dictionary, rather than as a separate argument to the `get` method because it’s a parameter for the formatter that reads the dataset and not a dimension of the dataset itself.

!!! note
        In LSST version 26.0.2, the `butler` command was `butler = Butler($DATASTORE, collection=["$RERUN/object"])`, but in the latest LSST version 28, this is recommended to be `butler = Butler.from_config($DATASTORE, collection=["$RERUN/object"])`. Although the former expression may still work, `mypy` will deny this construction with reporting "class Butler is an abstract class. Abstract classes must not be instantiated". <br> **In the next section for data analysis, we will only use the latest construction.**
