Different treatments of uStar threshold
=======================================

The recommended way of dealing with the uncertain uStar threshold for
filtering the half-hourly data, is to repeat all the processing steps
with several bootstrapped estimates of the threshold as in
`vignette('useCase')`.

First, some setup.

    #+++ load libraries used in this vignette
    library(REddyProc)
    library(dplyr)
    #+++ define directory for outputs
    outDir <- tempdir()  # CRAN policy dictates to write only to this dir in examples
    #outDir <- "out"     # to write to subdirectory of current users dir
    #+++ Add time stamp in POSIX time format to example data 
    # and filter long runs of equal NEE values
    EddyDataWithPosix <- fConvertTimeToPosix(
      filterLongRuns(Example_DETha98, "NEE")
      , 'YDH', Year = 'Year', Day = 'DoY', Hour = 'Hour')

Not applying uStar filtering
----------------------------

Subsequent processing steps can be performed without further uStar
filtering using `sEddyProc_sMDSGapFill`. Corresponding result columns
then have no uStar specific suffix.

    EProc <- sEddyProc$new(
      'DE-Tha', EddyDataWithPosix, c('NEE','Rg','Tair','VPD', 'Ustar'))
    EProc$sMDSGapFill('NEE')
    grep("NEE.*_f$",names(EProc$sExportResults()), value = TRUE)

    ## [1] "NEE_f"

User-specified uStar threshold
------------------------------

The user can provide value for uStar-filtering before gapfilling, using
`sEddyProc_sMDSGapFillAfterUstar`. Output columns for this uStar
scenario use the suffix as specified by argument `uStarSuffix` which
defaults to "uStar".

The friction velocity, uStar, needs to be in column named "Ustar" of the
input dataset.

    EProc <- sEddyProc$new(
      'DE-Tha', EddyDataWithPosix, c('NEE','Rg','Tair','VPD', 'Ustar'))
    uStar <- 0.46
    EProc$sMDSGapFillAfterUstar('NEE', uStarTh = uStar)
    grep("NEE.*_f$",names(EProc$sExportResults()), value = TRUE)

    ## [1] "NEE_uStar_f"

Single uStar threshold estimate
-------------------------------

The uStar threshold can be estimated from the uStar-NEE relationship
from the data without estimating its uncertainty by a bootstrap.

    EProc <- sEddyProc$new(
      'DE-Tha', EddyDataWithPosix, c('NEE','Rg','Tair','VPD', 'Ustar'))
    # estimating the thresholds based on the data (without bootstrap)
    (uStarTh <- EProc$sEstUstarThold())

    ##   aggregationMode seasonYear  season     uStar
    ## 1          single         NA    <NA> 0.4162500
    ## 2            year       1998    <NA> 0.4162500
    ## 3          season       1998 1998001 0.4162500
    ## 4          season       1998 1998003 0.4162500
    ## 5          season       1998 1998006 0.3520000
    ## 6          season       1998 1998009 0.3369231
    ## 7          season       1998 1998012 0.1740000

    # may plot saturation of NEE with UStar for a specified season to pdf
    EProc$sPlotNEEVersusUStarForSeason(levels(uStarTh$season)[3], dir = outDir )

Next, the annual estimate is used as the default in gap-filling. Output
columns use the suffix as specified by argument `uSstarSuffix` which
defaults to "uStar".

    #EProc$useAnnualUStarThresholds()
    EProc$sMDSGapFillAfterUstar('NEE')

    ## Warning in .self$sGetUstarScenarios(): uStar scenarios not set yet. Setting
    ## to annual mapping.

    grep("NEE.*_f$",names(EProc$sExportResults()), value = TRUE)

    ## [1] "NEE_uStar_f"

Scenarios across distribution of u\* threshold estimate
-------------------------------------------------------

Choosing a different u\* threshold effects filtering and the subsequent
processing steps of gap-filling, and flux-partitioning. In order to
quantify the uncertainty due to not exactly knowing the u\* threshold,
these processing steps should be repeated for different threshold
scenarios, and the spread across the results should be investigated.

First, the quantiles of the threshold distribution are estimated by
bootstrap.

    EProc <- sEddyProc$new(
      'DE-Tha', EddyDataWithPosix, c('NEE','Rg','Tair','VPD', 'Ustar'))

    ## New sEddyProc class for site 'DE-Tha'

    EProc$sEstimateUstarScenarios(
        nSample = 100L, probs = c(0.05, 0.5, 0.95))

    ## 

    ## Estimated UStar distribution of:
    ##      uStar       5%       50%       95%
    ## 1 0.41625 0.378475 0.4524588 0.6209465 
    ## by using  100 bootstrap samples and controls:
    ##                        taClasses                    UstarClasses 
    ##                               7                              20 
    ##                           swThr            minRecordsWithinTemp 
    ##                              10                             100 
    ##          minRecordsWithinSeason            minRecordsWithinYear 
    ##                             160                            3000 
    ## isUsingOneBigSeasonOnFewRecords 
    ##                               1

    # inspect the thresholds to be used by default
    EProc$sGetUstarScenarios()

    ##    season   uStar      U05       U50       U95
    ## 1 1998001 0.41625 0.378475 0.4524588 0.6209465
    ## 2 1998003 0.41625 0.378475 0.4524588 0.6209465
    ## 3 1998006 0.41625 0.378475 0.4524588 0.6209465
    ## 4 1998009 0.41625 0.378475 0.4524588 0.6209465
    ## 5 1998012 0.41625 0.378475 0.4524588 0.6209465

By default the annually aggregated threshold estimates are used for each
season within one year as in the original method publication. To see the
estimates for different aggregation levels, use method
`sEddyProc_sGetEstimatedUstarThresholdDistribution`:

    (uStarThAgg <- EProc$sGetEstimatedUstarThresholdDistribution())

    ##   aggregationMode seasonYear  season     uStar        5%       50%
    ## 1          single         NA    <NA> 0.4162500 0.3784750 0.4524588
    ## 2            year       1998    <NA> 0.4162500 0.3784750 0.4524588
    ## 3          season       1998 1998001 0.4162500 0.3784750 0.4524588
    ## 4          season       1998 1998003 0.4162500 0.3289559 0.4000000
    ## 5          season       1998 1998006 0.3520000 0.3318286 0.3864683
    ## 6          season       1998 1998009 0.3369231 0.2297673 0.4044653
    ## 7          season       1998 1998012 0.1740000 0.1849500 0.4325682
    ##         95%
    ## 1 0.6209465
    ## 2 0.6209465
    ## 3 0.6209465
    ## 4 0.5890722
    ## 5 0.4483500
    ## 6 0.5360426
    ## 7 0.5890722

In conjunction with method `usGetSeasonalSeasonUStarMap` and
`sEddyProc_sSetUstarScenarios` this can be used to set seasonally
different u\* threshold. However, this common case supported by method
`sEddyProc_useSeaonsalUStarThresholds`.

    #EProc$sSetUstarScenarios(
    #  usGetSeasonalSeasonUStarMap(uStarThAgg)[,-2])
    EProc$useSeaonsalUStarThresholds()
    # inspect the changed thresholds to be used
    EProc$sGetUstarScenarios()

    ##    season     uStar       U05       U50       U95
    ## 3 1998001 0.4162500 0.3784750 0.4524588 0.6209465
    ## 4 1998003 0.4162500 0.3289559 0.4000000 0.5890722
    ## 5 1998006 0.3520000 0.3318286 0.3864683 0.4483500
    ## 6 1998009 0.3369231 0.2297673 0.4044653 0.5360426
    ## 7 1998012 0.1740000 0.1849500 0.4325682 0.5890722

Several function whose name ends with 'UstarScens' perform the
subsequent processing steps for all uStar scenarios. They operate and
create columns that differ between threshold scenarios by a suffix.

    EProc$sMDSGapFillUStarScens("NEE")
    grep("NEE_.*_f$",names(EProc$sExportResults()), value = TRUE)

    ## [1] "NEE_uStar_f" "NEE_U05_f"   "NEE_U50_f"   "NEE_U95_f"

    EProc$sSetLocationInfo(LatDeg = 51.0, LongDeg = 13.6, TimeZoneHour = 1)
    EProc$sMDSGapFill('Tair', FillAll = FALSE, minNWarnRunLength = NA)
    EProc$sMDSGapFill('Rg', FillAll = FALSE, minNWarnRunLength = NA)
    EProc$sMDSGapFill('VPD', FillAll = FALSE, minNWarnRunLength = NA)
    EProc$sMRFluxPartitionUStarScens()
    grep("GPP_.*_f$",names(EProc$sExportResults()), value = TRUE)

    ## [1] "GPP_U95_f"   "GPP_U50_f"   "GPP_U05_f"   "GPP_uStar_f"

    if (FALSE) {
      # run only interactively, because it takes long
      EProc$sGLFluxPartitionUStarScens(uStarScenKeep = "U50")
      grep("GPP_DT_.*_f$",names(EProc$sExportResults()), value = TRUE)
    }

The argument `uStarScenKeep = "U50"` specifies that the outputs that are
not distinguished by the suffix, e.g. `FP_GPP2000`, should be reported
for the median u\* threshold scenario with suffix `U50`, instead of the
default first scenario.

See also
--------

A more advanced case of user-specified seasons for uStar threshold
estimate is given in [`vignette('DEGebExample')`](DEGebExample.html).
