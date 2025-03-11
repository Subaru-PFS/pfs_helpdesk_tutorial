# Process Science Data

## Basic Information

---

With the calibration products built, we can now process the science data. There are a few pipelines available:

- `reduceExposure`: process an exposure through merging arms, producing `postISRCCD`, `pfsArm`, `lines`, `detectorMap`, `pfsMerged`, `sky1d`, and `fiberNorms`. 
This can be used to process quartz exposures (or sky exposures when flux calibration is not wanted). 
The `calexp` data product, familiar from the Gen2 pipeline, is not produced by the Gen3 pipeline; instead, use the `postISRCCD` data product for the processed image. 
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
$ defineCombination.py $DATASTORE PFS quartz --where "exposure.target_name = 'FLAT' AND dither = 0.0"
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
--instrument lsst.obs.pfs.PrimeFocusSpectrograph \
-i PFS/raw/all,PFS/raw/pfsConfig,PFS/calib \
-o "$RERUN"/reduceExposure \
-p '$DRP_STELLA_DIR/pipelines/reduceExposure.yaml' \
-d "combination IN ('object', 'quartz')" \
--fail-fast \
-c isr:doCrosstalk=True \
-c reduceExposure:doApplyScreenResponse=False \
-c reduceExposure:doBlackSpotCorrection=False \
-c 'reduceExposure:targetType=[SCIENCE, SKY, FLUXSTD]'
```

Alternatively, you can run the science pipeline for an entire data collection:

```
# Science Pipeline
pipetask run \
--register-dataset-types -j $CORES -b $DATASTORE \
--instrument lsst.obs.pfs.PrimeFocusSpectrograph \
-i PFS/raw/all,PFS/raw/pfsConfig,PFS/calib \
-o "$RERUN"/science \
-p '$DRP_STELLA_DIR/pipelines/science.yaml' \
-d "combination = 'object'" \
--fail-fast \
-c isr:doCrosstalk=True \
-c fitFluxCal:fitFocalPlane.polyOrder=0 \
-c reduceExposure:doApplyScreenResponse=False \
-c reduceExposure:doBlackSpotCorrection=False \
-c 'reduceExposure:targetType=[SCIENCE, SKY, FLUXSTD]'
```

Notice that in the first case we’re running the `reduceExposure` pipeline, selecting the `object` and `quartz` combinations that we defined earlier. 
We’ve turned off the screen response and black spot correction, and we’ve set the `targetType` configuration parameter to disable extracting spectra for the (many) unilluminated fibers (we don’t have good fiber profiles for them anyway).

The `science` pipeline is similar, with the only important change that we’re setting the flux calibration fitting order to zero (because there aren’t enough fibers to fit a higher-order polynomial). 
Note that we do not have to run the `reduceExposure` pipeline before we run the `science` pipeline (a single command is sufficient to run the entire pipeline): the `science` pipeline knows how to produce all the necessary intermediate datasets itself, and the above two commands are completely independent: they do not share any intermediate datasets. 
However, we could have first run the `reduceExposure` pipeline and then fed its outputs into the `science` pipeline by including `$RERUN/reduceExposure` in the list of input collections for the `science` pipeline.

## Retrieve Data Products

---

There are some important differences in the data products produced by the Gen3 pipeline compared to the Gen2 pipeline. For the sake of efficiency (both in terms of processing time and reduced file numbers), the single-spectrum Gen2 products (`pfsSingle` and `pfsObject`) are written as multiple-spectrum Gen3 products per `cat_id` (`pfsCalibrated` and `pfsCoadd`). The equivalent of a `pfsObject` can be retrieved directly from the `pfsCoadd` dataset:

```
from lsst.daf.butler import Butler
butler = Butler("INTEGRATION", collections="integration/science")
pfsObject = butler.get("pfsCoadd.single", cat_id=1, combination="object", parameters=dict(objId=55))
```

Note that the `objId` needs to be specified in the `parameters` dictionary, rather than as a separate argument to the `get` method because it’s a parameter for the formatter that reads the dataset and not a dimension of the dataset itself.

Another key difference is that the Gen3 `butler` assigns filenames in its own way, rather than following the PFS datamodel. In order to deliver products according to the PFS datamodel, we need to export the products from the `butler`:

```
$ exportPfsProducts.py -b $DATASTORE -i PFS/raw/pfsConfig,"$RERUN"/science -o $EXPORT
```

This creates a directory tree within the export directory `$EXPORT` with links to the files in the butler datastore, and individual
spectrum files for the `pfsSingle` and `pfsObject` datasets. 
You can then deliver the export directory to the consumer.
Running the entire integration test with 20 cores takes approximately 18 minutes of runtime.
