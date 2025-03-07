# PFS Datamodel

There are implementations of Python classes that represent many of the files in the PFS pipeline, supporting
reading and writing FITS files that conform to this model.  To use them in the Python environment, you can import, e.g. `pfs.datamodel.pfsConfig`.

In this section, we only summarize the essential information of the datamodel that can be helpful for users to understand how the pipeline works. Please note that the datamodel is not fixed, and a complete description with possible updates of the datamodel can be found in the dedicated GitHub document: [datamodel.txt](https://github.com/Subaru-PFS/datamodel/blob/master/datamodel.txt).

## Files and Their Formats

---

| Category                | File Type                      | File Name Format                                             | Variables                                          | Note                                                                                           |
|-------------------------|--------------------------------|--------------------------------------------------------------|----------------------------------------------------|------------------------------------------------------------------------------------------------|
| **Configuration**       | pfsDesign                      | `pfsDesign-%016x.fits`                                        | `pfsDesignId`                                      | PFS design configuration (SHA-1 truncated to 64 bits, rendered in 16 hexadecimal digits).       |
|                         | pfsConfig                      | `pfsConfig-0x%016x-%06d.fits`                                  | `pfsDesignId`, `visit`                             | Instrument configuration for a visit; includes the design ID (prefixed with `0x`) and visit number. |
| **Raw & Auxiliaries**   | Auto-Guider image data         | `agcc_{pfs_visit:06d}_{agc_exposure_id:08d}.fits`             | `pfs_visit`, `agc_exposure_id`                     | Unique auto-guider exposure; contains PDU, CAM1–CAM6, and optional TABLE HDUs with moment/centroid data. |
|                         | Raw Data (spectrograph cameras)| `PF%1sA%06d%1d%1d.fits`                                       | `site`, `visit`, `spectrograph`, `armNum`          | "A" frames produced by the spectrograph cameras; note the order: spectrograph then armNum (integer code).  |
|                         | Raw Data (MCS)                 | `PF%1sC%06d%02d.fits`                                         | `site`, `visit`, `sequenceId`                      | Frames from the MCS using a 2-digit sequenceId.                                                 |
| **Calibration Files**   | Flats              | `pfsFlat-%s-%06d-%1s%1d.fits`                                  | `calibDate`, `visit0`, `arm`, `spectrograph`        | Processed flat-field files; `visit0` is the first valid visit for these calibrations.           |
|                         | Biases             | `pfsBias-%s-%06d-%1s%1d.fits`                                  | `calibDate`, `visit0`, `arm`, `spectrograph`        | Processed bias frames.                                                                          |
|                         | Darks              | `pfsDark-%s-%06d-%1s%1d.fits`                                  | `calibDate`, `visit0`, `arm`, `spectrograph`        | Processed dark frames.                                                                          |
|                         | Detector Map                   | `pfsDetectorMap-%06d-%s%1d.fits`                               | `visit0`, `arm`, `spectrograph`                    | Maps fiber/wavelength to detector (x,y); internal format varies (e.g. Splined, MultipleDistortions). |
|                         | Fiber Profiles                 | `pfsFiberProfiles-%s-%06d-%1s%1d.fits`                         | `calibDate`, `visit0`, `arm`, `spectrograph`        | Empirical, oversampled fiber profiles from 2-D flat-field images.                               |
|                         | pfsFiberNorms                  | `pfsFiberNorms-%1s%1s%06d-%1s%1d.fits`                         | `site`, `category`, `visit`, `arm`, `spectrograph`  | Normalization coefficients for individual fibers.                                             |
| **Intermediate Products** | Calibrated Images (postISRCCD) | `postISRCCD-%1s%1s%06d-%1s%1d.fits`                           | `site`, `category`, `visit`, `arm`, `spectrograph`  | Persisted LSST Exposure (postISRCCD)                                                            |
|                         | Calibrated Images (calexp)      | `calexp-%1s%1s%06d-%1s%1d.fits`                                | `site`, `category`, `visit`, `arm`, `spectrograph`  | Persisted LSST Exposure (calexp).                                                               |
| **Wavelength Calibration**   | (Gen 2) arcLines                        | `arcLines-%1s%1s%06d-%1s%1d.fits`                              | `site`, `category`, `visit`, `arm`, `spectrograph`  | Contains arc lamp line identifications for wavelength calibration.                              |
|                         | (Gen 3) lineCentroids                | `lineCentroids-%1s%1s%06d-%1s%1d.fits`                         | `site`, `category`, `visit`, `arm`, `spectrograph`  | Measurements of spectral line centroids.                                                      |
|                         | (Gen 3) linePhotometry                 | `linePhotometry-%1s%1s%06d-%1s%1d.fits`                        | `site`, `category`, `visit`, `arm`, `spectrograph`  | Photometric measurements of spectral lines (e.g. flux, width).                                  |
|                         | apCorr                         | `apCorr-%1s%1s%06d-%1s%1d.fits`                                | `site`, `category`, `visit`, `arm`, `spectrograph`  | Aperture correction data used in flux calibration.                                            |
|                         | sky2d                          | `sky2d-%1s%1s%06d-%1s%1d.fits`                                 | `site`, `category`, `visit`, `arm`, `spectrograph`  | Two-dimensional sky background model for sky subtraction.                                     |
|                         | sky1d                          | `sky1d-%1s%1s%06d-%1s%1d.fits`                                 | `site`, `category`, `visit`, `arm`, `spectrograph`  | One-dimensional extracted sky spectrum.                                                       |
| **Flux Calibration** | fluxCal                     | `fluxCal-%1s%1s%06d-%1s%1d.fits`                               | `site`, `category`, `visit`, `arm`, `spectrograph`  | Flux calibration factors/curves for converting counts to physical flux units.                   |
|                         | pfsPSF                         | `pfsPSF-%1s%1s%06d-%1s%1d.fits`                                | `site`, `category`, `visit`, `arm`, `spectrograph`  | Point Spread Function model used in spectral extraction.                                      |
|                         | pfsFluxReference               | `pfsFluxReference-%1s%1s%06d-%1s%1d.fits`                      | `site`, `category`, `visit`, `arm`, `spectrograph`  | Reference flux scale for calibration.                                                         |
|  **Products**           | pfsArm                         | `pfsArm-%1s%06d-%1s%1d.fits`                                   | `site`, `visit`, `arm`, `spectrograph`             | Arm-specific metadata/processing results.                                                     |
|                         | pfsMerged                      | `pfsMerged-%1s%1s%06d-%1s%1d.fits`                             | `site`, `category`, `visit`, `arm`, `spectrograph`  | Merged spectra from multiple visits/exposures.                                                  |
|                         | pfsSingle                      | `pfsSingle-%1s%1s%06d-%1s%1d.fits`                             | `site`, `category`, `visit`, `arm`, `spectrograph`  | Single-visit spectra product.                                                                   |
|                         | (Gen 2) pfsObject                | `pfsObject-%1s%1s%06d-%1s%1d.fits`                             | `site`, `category`, `visit`, `arm`, `spectrograph`  | Catalog of extracted objects; includes a unique 64-bit object ID (formatted in hexadecimal).     |
|                         | (Gen 3) pfsCalibrated                  | `pfsCalibrated-%1s%1s%06d-%1s%1d.fits`                         | `site`, `category`, `visit`, `arm`, `spectrograph`  | Fully calibrated spectra after applying wavelength and flux corrections.                      |
|                         | (Gen 3) pfsCoadd                       | `pfsCoadd-%1s%1s%06d-%1s%1d.fits`                              | `site`, `category`, `visit`, `arm`, `spectrograph`  | Coadded spectra produced by combining multiple exposures/visits to improve signal-to-noise.      |
|                         | (LAM 1D DRP) pfsZcandidates                 | `pfsZcandidates-%1s%1s%06d-%1s%1d.fits`                        | `site`, `category`, `visit`, `arm`, `spectrograph`  | Contains redshift candidate measurements.                                                     |



## Variables in the Filename Formats 

---

There are many variables shown in the filename as shown above, and you can check their meaning as listed below.

### Site 
| `site` | meaning  |
|------|-------|
| J    | JHU   |
| L    | LAM   |
| X    | Subaru Offline |
| I    | IPMU  |
| A    | ASIAA |
| S    | Summit |
| P    | Princeton |
| F    | Simulation (Fake) |

### Camera Categories
| `category` |  meaning |
|------|----------|
| A    | Science  |
| B    | UTR (Up The Ramp)     |
| C    | Metrology Camera |
| D    | Auto-guider |

### Visit  
`visit` is an incrementing exposure number, unique to any site.

### Spectrograph
`spectrograph` is an integer ranging from 1-4.

### Arm
| `armNum` | `arm`| meaning  |
|----------|------|----------|
| 1        | b    | Blue     |
| 2        | r    | Red      |
| 3        | n    | IR       |
| 4        | m    | Medium resolution red |

!!! note armNum 
    `armNum` is used only in raw filenames.

### Tract  
`tract` is an integer in the range (0, 99999) specifying an area of the sky.

### Patch  
`patch` is a string of the form `m,n` specifying a region within a tract.

### Object ID 
`objId` is a unique 64-bit object ID for an object (e.g., HSC object ID from the database).  
Written out using `%016x` format for compactness.

### Catalog ID 
`catId` is a small integer specifying the source of the `objId`. Defined values:

| catId  | Description |
|--------|------------|
| 0      | Simulated catalog |
| 1      | Gaia DR1 |
| 2      | Gaia DR2 |
| 3      | Gaia EDR3 |
| 4      | Gaia DR3 |
| 5      | HSC-SSP PDR1 (Wide) |
| 6      | HSC-SSP PDR1 (Deep+UltraDeep) |
| 7      | HSC-SSP PDR2 (Wide) |
| 8      | HSC-SSP PDR2 (Deep+UltraDeep) |
| 9      | HSC-SSP PDR3 (Wide) |
| 10     | HSC-SSP PDR3 (Deep+UltraDeep) |
| 11     | HSC-SSP PDR4 (Wide) |
| 12     | HSC-SSP PDR4 (Deep+UltraDeep) |
| 1001   | Sky positions from S21A HSC-SSP (Wide) |
| 1002   | Sky positions from PS1 |
| 1003   | Sky positions from Gaia |
| 1004   | Sky positions for regions without PS1 data |
| 90001  | Messier 15 stars (Engineering Obs.) |
| 90002  | NGC1904 (Nov 2022 Eng. Run) |
| 90003  | NGC2419 (Nov 2022 Eng. Run) |
| 90004  | NGC5272 (Feb 2023 Eng. Run) |
| 90005  | NGC5904 (Feb 2023 Eng. Run) |
| 90006  | NGC7078 (July 2023 Eng. Run) |
| 90007  | NGC7089 (July 2023 Eng. Run) |
| 90008  | NGC7099 (July 2023 Eng. Run) |

### Design ID

`pfsDesignId` is an integer uniquely specifying the configuration of the PFI; specifically a SHA-1 of the
  (fiberId, ra, dec) tuples (with position rounded to the nearest arcsecond) truncated to 64 bits.

### Visit Hash

`pfsVisitHash` is an integer uniquely defining the set of visits contributing to a reduced spectrum; this will be calculated as a SHA-1 truncated to 64 bits


 
