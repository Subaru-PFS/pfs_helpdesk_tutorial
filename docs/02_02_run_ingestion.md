# Data Ingestion

---

We can ingest data into the butler repository. 
There are two kinds of data that we need to ingest: raw images and the `pfsConfig` files. 
These are ingested using two separate commands:

```
$ butler ingest-raws $DATASTORE $drp_stella_data/raw/PFSA*.fits --ingest-task lsst.obs.pfs.gen3.PfsRawIngestTask --transfer link --fail-fast
$ ingestPfsConfig.py $DATASTORE PFS PFS/raw/pfsConfig $drp_stella_data/raw/pfsConfig*.fits --transfer link
```

The parameters include:

- `--transfer`: The method by which data is added to the repository, including `link`, `copy`, and `move`. 
- `--fail-fast`: The process will immediately stop the ingestion process if an error occurs. This is useful for debugging.

The ingestion process places the files (referred to as “datasets” in the butler) in the repository and records them in the registry database. Each file is placed in a **collection**, which can be thought of as a *directory* (i.e., `$DATASTORE/PFS/` in the example) in the butler (and in the case of the datastore on a traditional filesystem, it is implemented as a directory). 
The raw data is placed in the collection <instrument\>/raw/all, while we’ve specified above that the `pfsConfig` files are placed in the collection `PFS/raw/pfsConfig`.

There are different kinds of [collections](https://pipelines.lsst.io/modules/lsst.daf.butler/organizing.html#collections):

- `RUN` collection always associates the datasets. 
- `CALIBRATION` collections associate datasets with a timespan indicating the validity range. 
- `CHAINED` collections provide a search path through multiple collections.

Each dataset is specified by a “`dataId`”, which is a dictionary of key-value pairs representing the dimensions.

For example, 

- `raw` image may have a `dataId` like {'`instrument`': '`PFS`', '`exposure`': `123`, '`arm`': '`r`', '`spectrograph`': `3`}. 
- `pfsConfig` file is valid for an entire exposure, so may have a `dataId` like {'`instrument`': '`PFS`', '`exposure`': `123`}.

In general, users should treat the files in the datastore as a `butler` implementation detail, and use the `butler` commands and Python API to access the data products. 
There are some kinds of datastores that do not use a traditional filesystem (e.g., the S3 datastore), and so the files may not be directly accessible.

!!! warning
    The registry database tracks all files in the datastore. Do not delete files from the datastore without using the appropriate butler commands.

You can see what raw datasets are in the datastore with the following command:

```
$ butler query-datasets $DATASTORE --collections PFS/raw/all
```

The result looks something like this:

```
type     run                         id                 instrument arm dither pfs_design_id spectrograph detector exposure
---- ------------- ------------------------------------ ---------- --- ------ ------------- ------------ -------- --------
raw  PFS/raw/all 27217522-a357-5071-a32b-af97b5b8bee6          PFS   b  0.0             1            1        0        0
raw  PFS/raw/all 0ce0cbea-fe7c-589e-8259-30060bf20500          PFS   b  0.0             1            1        0        1
[...]
raw  PFS/raw/all 570092eb-f571-5631-8d20-11acbeabc640          PFS   r  0.0             3            1        1        26
raw  PFS/raw/all f8e3ae71-2cdf-5e55-bc42-4a4fb913770c          PFS   r  0.0             4            1        1        27
```

Datasets can be accessed from Python using the `butler` API, which has some similarities to the Gen2 `butler`:

```
from lsst.daf.butler import Butler

butler = Butler("/path/to/datastore", collections="PFS/raw/all")
raw = butler.get("raw", instrument="PFS", exposure=12, arm="r", spectrograph=1)
rawImage = raw.getImage()
```

Note that the raw data returned from the butler is now of type `PfsRaw`, which provides a common interface for both CCD and NIR detectors. 
You can use `butler.get("raw.exposure", ...)` to get the exposure from the raw data directly.