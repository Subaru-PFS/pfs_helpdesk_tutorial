# File Access

After running the PFS 2D DRP, data reduction are performed and the spectrum products are constructed.

## The Location of Data Products

---

First, where can we find the product files? 
In this assumed working directory and user rerun, the output files are located under `$WORKDIR/pfs/data/datastore/u/(username)/object/`. 

You can check by:

```
ls $WORKDIR/pfs/data/datastore/u/(username)/object/
```

You may find a folder named by date and time, e.g., `20250218T070224Z`. This is one of the reruns for your `object` collection. 
If you run the processing multiple times, different folders with specific timestamps will be created.

Let's inspect the contents of  `20250218T070224Z/`:

```
$ ls 20250218T070224Z

apCorr                 cosmicray2_metadata         fitPfsFluxReference_metadata  mergeArms_log       pfsCoadd               reduceExposure_log
calexp                 detectorMap                 fluxCal                       mergeArms_metadata  pfsCoaddLsf            reduceExposure_metadata
coaddSpectra_config    fitFluxCal_config           isr_config                    packages            pfsFluxReference       sky1d
coaddSpectra_log       fitFluxCal_log              isr_log                       pfsArm              pfsMerged
coaddSpectra_metadata  fitFluxCal_metadata         isr_metadata                  pfsArmLsf           pfsMergedLsf
cosmicray2_config      fitPfsFluxReference_config  lines                         pfsCalibrated       postISRCCD
cosmicray2_log         fitPfsFluxReference_log     mergeArms_config              pfsCalibratedLsf    reduceExposure_config
```

You will see a bunch of directories. Refer to the release document as well as to the [datamodel](https://github.com/Subaru-PFS/datamodel/blob/master/datamodel.txt) for details.
Here is a brief summary of some of the most important files: `pfsArm` contains 1D-extracted, wavelength-calibrated spectra for each arm (`b`, `r`, `n`, `m`) separately. Here, `b`=blue, `r`=red, `n`=nearIR, and `m`=medium resolution red-arm. The arms are combined in a `pfsMerged` file. Note that the sky subtraction has been performed in `pfsMerged`. Then the pipeline applies the flux calibration and `pfsSingle` is a fully calibrated 1d spectrum for a visit. Finally, `pfsCoadd` is a coadd spectrum from multiple visits.

## Export the Data Products

---

You may notice that `pfsObject`, which is specified in the data model, is missing from the current directory. 
Since Gen3 PFS 2D DRP, the products have been stored in`pfsCoadd`. However, after running:

```
$ exportPfsProducts.py -b $DATASTORE -i PFS/raw/pfsConfig,"$RERUN"/science -o $EXPORT
```

You may still be able to retrieve the data structure as follows:

```
$EXPORT/
 ├── detectorMap : directory for detectorMap of different arm & spectrograph
 ├── images : directory for raw 2D images of each visit
 ├── pfsConfig : directory for pfsConfig files
 ├── pfsArm : directory for pfsArm products
 ├── pfsMerged : directory for pfsMerged products
 │     ├── 20241024 : exposures on date 2024/10/24
 │     ├── 20241025 : exposures on date 2024/10/25
 │     └── 20241026 : exposures on date 2024/10/26
 │           ├── pfsMerged-116413.fits : psfMerged spectra of visit=116413
 │           ├── pfsMergedLsf-116413.pickle : psfMerged LSF of visit=116413
 │           ├── ...
 ├── pfsSingle : directory for pfsSingle products
 │     ├── -0001 : targets with catId=-1
 │     ├── 01002 : targets with catId=1002, i.e., PS1
 │     ├── 03006 : targets with catId=3006
 │     └── 10086 : targets with catId=10086
 │           └── 00001 : tract number, meaningless at the moment
 │                 └── 1,1 : patch number, meaningless at the moment
 │                      ├── pfsSingle-[catId]-[tract]-[patch]-[objId]-[visit].fits : psfSingle spectra of catId=10086
 │                      ├── pfsSingleLsf-[catId]-[tract]-[patch]-[objId]-[visit].fits : psfSingle LSF of catId=10086
 │                      ├── ... 
 └── pfsObject
       ├── -0001: (same as above)
```

!!! note
      This is not recommended since Gen3, and we suggest working directly with `pfsCoadd` instead.


