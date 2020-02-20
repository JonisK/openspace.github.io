---
title: Gaia
layout: default

parent: Components
nav_order: 4
---

_Valid as of July 1, 2018_

# How to render stars from Gaia DR2 in OpenSpace
This guide will explain how to render stars released by ESA's Gaia mission as their second data release (DR2) without having to create any subsets yourself.

If you instead want to render your own star data or if you want to create new subsets from the full DR2 then check out the next section down below.

### 1. Install OpenSpace 
The first step is to download and build a version of OpenSpace that can render the star data.  As of June 2018 that is only the `feature/gaia-mission` branch.  Hopefully it will be merged into master soon enough.

### 2. Define which scene to run
OpenSpace has several different scenes depending on what you want to show.  To change scene open `OpenSpace/openspace.cfg` and make sure that `Asset = "gaia_mission"` is set.  The Gaia feature will be included into the default scene soon enough but as of June 2018 it has not.

To change anything in the scene go to `OpenSpace/data/assets/gaia_mission.scene`.  By default the full Digital Universe catalog will be loaded in addition to the Gaia stars as well as the Sun, Earth, Moon and a model of the Gaia spacecraft.

### 3. Define what dataset to render
In `OpenSpace/data/assets/scene/milkyway/gaia/gaiamission.asset` the user can define what dataset to render.  By default the radial velocity dataset of 7.2 million stars will be downloaded and rendered on startup.  The size of the dataset is 335 MB and it will be stored in the sync folder within the OpenSpace directory.

An additional dataset of 618 million stars is available for download as well.  It can be downloaded by using the _TaskRunner.exe_ with "gaia_download.task".  The dataset consists of all stars with a parallax error below 0.5 and is about 28 GB in size.

The same script will also download the full DR2 dataset with 24 binary values per star.  This dataset is 151 GB in size and can be used to create new subsets.  If that is not interesting at the moment then go to `OpenSpace/data/assets/scene/milkyway/gaia/gaia_dr2_download_stars.asset` and comment out the lines regarding "Gaia DR2 Full Raw".

To change dataset use to "localStars" variable in `gaiamission.asset`to point to another directory.  Loading from disk will be much faster if using a SSD hard drive so if that is available please consider moving the dataset from `OpenSpace/sync/http/gaia_stars_rv_octree/1/` to a directory named `/DR2_rv_Octree[10kSPN,20dist]/`on the SSD drive and change the "localStars" variable accordingly.

### 4. Running OpenSpace
When running OpenSpace there are several properties that you can change during runtime.  Bring up the menu by pressing the F2 button.  The Gaia stars are under `GaiaMission/Gaia Mission/`.  For a full explanation of the properties please open `OpenSpace/documentation/Documentation.html#gaiamission_renderablegaiastars`.

The most important aspects are the following: 
**File Path**
This is used to change dataset during runtime.

**Render Option**
There are 3 different render modes;  _Static_, _Color_ and _Motion_.  _Static_ uses only the position of the stars and assume that they all have the same magnitude.  _Color_ uses position, absolute magnitude (to calculate luminosity and apparent magnitude) and color.  _Motion_ adds velocity to the mix.  For a realistic rendering use _Color_.  To be able to make the stars move use _Motion_.

**Shader Option**
Sets which technique to use for the rendering.  This will change what other options that will are available in the menu.  Generally _Points_ are faster than _Billboards_, especially for big datasets, and _SSBOs_ are faster than _VBOs_.  _SSBOs_ are not available for Mac users unfortunately. 

**Thresholds**
Used to filter the data in real-time.  Sets min and maximum.  If they are set to the same value only that value will be filtered away.

**Rendered stars**
he number of stars that are actually rendered on screen.

**Luminosity multiplier**
Used to scale the brightness of the stars.  Good to use when traveling further out. 

**LOD Pixel Threshold**
Turn this up to use LOD data.  Not necessary for small datasets (as the RV dataset) but a must for larger ones.

**Max GPU Memory**
This determines how much memory we can use on the GPUs for our buffers.  If nothing is visible and the framerate drops to below 5 fps then try to decrease this value.  When rendering large datasets and the _GPU Stream Budget_ is down to 0 is may also be a good idea to try to increase this value.  Every GPU can allocate different max sizes so it is a matter of trial and error for every new hardware.  When you have found the maximum for you computer you can store that value in `gaiamission.asset` so it will be used on startup the next time.

For _Points_ you then have **Filter Size** and **Normal Distribution Sigma** that determines the size and form of the stars.  A bigger filter size will decrease the performance.

For _Billboards_ you instead have the **Billboard Size** and **Close-up Boost Distance** for changing the size.  A larger size will decrease the performance here as well.

# How to create new subsets from Gaia DR2 and how to render them (or your own star data) in OpenSpace 
This guide will go through the different steps needed for creating new subsets from big star datasets such as Gaia DR2 and also how to render your own star dataset with the technique used for Gaia stars. 

A pre-processed version of the full DR2 have also been uploaded.  From this dataset endless new subsets can be created.  Step 0 describes how to download that dataset.

The basic steps in the OpenSpace pipeline are the following:

0. (Possibly) Download datasets 
1. Get the data in a readable format
2. Read the raw data and keep a number of interesting values per star 
3. Construct an octree structure, possibly by filtering away some stars
4. Render the structured stars in OpenSpace

Steps 2-4 can either be done as separate tasks or on start-up with OpenSpace.

First off, there is a difference if the dataset is stored in one file or in several files.  If the dataset is stored in only one file then we assume that it can fit in RAM because even if the process is split into different steps we will keep the "single file format" and it will thus not be possible to stream from disk later.  If the dataset is stored in several files it doesn't matter if it can fit in RAM or not, the steps will be the same nevertheless.  However, when reading from a folder you NEED to do steps 2-4 as separate tasks! 

### 0. (Possibly) Download datasets
This step is for everybody that wants to create new subsets from the full DR2 data without having to download the full 1.2 TB from the Gaia Archive.  If that does not apply to you then please proceed to the next step. 

To download a processed version of the full DR2 start a _TaskRunner.exe_ with "gaia_download.task".  This will download both the DR2 dataset (151 GB, 8 files) and a generated subset with 618 million stars (28 GB, ~50k files, generated by filtering away all stars with parallax errors above 0.5) 

The DR2 dataset contains 24 values per star (positions (x3), velocities (x3), magnitudes (G, Bp, Rp), colors (Bp-Rp, Bp-G, G-Rp), ra, dec, parallax, proper motion (ra, dec), radial velocity and errors for the last 6).  Positions, velocities, G magnitude and Bp-Rp color will be used for rendering, the rest can be used to filter the stars later.

For how generate new subset skip to step 3.

### 1. Get the data in a readable format
As of June 2018 OpenSpace can read a single file in Speck, FITS, Binary and BinaryOctree formats.  Binary files are produced by the _ReadFitsTask_ and _ReadSpeckTask_ (Step 2) while BinaryOctree files are produced by the _ConstructOctreeTask_ (Step 3).

When reading a single Speck file the order of the star values are assumed to be:

    0 x
    1 y
    2 z
    3 color
    4 - not used
    5 absmag
    6 - not used
    7 - not used
    8 - not used
    9 - not used
    10 - not used
    11 - not used
    12 - not used
    13 vx
    14 vy
    15 vz
    16 speed

When reading a single FITS file the order is assumed to be: 

    "Position_X",
    "Position_Y",
    "Position_Z",
    "Velocity_X",
    "Velocity_Y",
    "Velocity_Z",
    "Gaia_Parallax",
    "Gaia_G_Mag",
    "Tycho_B_Mag",
    "Tycho_V_Mag",
    "Gaia_Parallax_Err", 
    "Gaia_Proper_Motion_RA", 
    "Gaia_Proper_Motion_RA_Err",
    "Gaia_Proper_Motion_Dec",
    "Gaia_Proper_Motion_Dec_Err",
    "Tycho_B_Mag_Err",
    "Tycho_V_Mag_Err"

If you want to use other values or different names then it is possible to change it in `OpenSpace/modules/fitsfilereader/src/fitsfilereader.cpp` in _readFitsFile()_ or _readSpeckFile()_.  After any code changes you have to recompile OpenSpace.

Reading of multiple files is only available for FITS files right now and the names of the columns follow the Gaia DR2 standard.  That means that we expect following naming convention for the columns: 

    "ra"
    "ra_error"
    "dec"
    "dec_error"
    "parallax"
    "parallax_error"
    "pmra"
    "pmra_error"
    "pmdec"
    "pmdec_error"
    "phot_g_mean_mag"
    "phot_bp_mean_mag"
    "phot_rp_mean_mag"
    "bp_rp"
    "bp_g"
    "g_rp"
    "radial_velocity"
    "radial_velocity_error"

If you want to read the full DR2 this means that you have to download the files from the Gaia archive, extract them (preferably with a console using 7-zip for example) and converting the CSV files to FITS (for example by using astropy).  An already processed dataset is available for download as well, see Step 0.


### 2. Read the raw data
If you want to read a single-file dataset on start-up then skip to step 4.

To read either a single-file or multiple-files dataset as a separate process start the OpenSpace _TaskRunner.exe_ and type in "gaia_read.task".

The task is specified in `OpenSpace/data/tasks/gaia_read.task` as 

    local dataFolder = "E:/path/to/dataDir"
    return {
        {
            Type = "ReadFitsTask",
            InFileOrFolderPath = dataFolder .. "/Gaia_DR2/gaia_source/fits/",
            OutFileOrFolderPath = dataFolder .. "/Gaia_DR2_full_24columns/",
            SingleFileProcess = false,
            ThreadsToUse = 8,
        },
    }

for **FITS files** or

    local dataFolder = "E:/path/to/dataDir"
    return {
        {
            Type = "ReadSpeckTask",
            InFilePath = dataFolder .. "/GaiaUMS.speck",
            OutFilePath = dataFolder .. "/GaiaUMS.bin",
        },
    }

for reading **speck files**.

The task will take a path to the file or folder which should be read.  If `SingleFileProcess` is true then it must point to a file, if it is false then it must point to a folder.

When reading from a folder the task will write 24 values per star to 8 binary files in the specified output folder.  That folder later has to be processed by the _ConstructOctreeTask_.  When reading a single file it will output a single file that can either be read directly by OpenSpace or be processed by the _ConstructOctreeTask_.

It is also possible to specify how many threads to use when reading from a folder.

If reading the full DR2 dataset this task will take about 7h using 8 threads on a moderate computer.  This only has to be done once for a dataset however, as long as you don't want to add more values to filter by. The output of this task can be downloaded as explained in Step 0.


### 3. Construct the octree
To create the octree run a _TaskRunner.exe_ with "gaia_octree.task".

The task is specified in `OpenSpace/data/tasks/gaia_octree.task` as 

    local dataFolder = "E:/path/to/dataDir"
    return {
        {
            Type = "ConstructOctreeTask",
            InFileOrFolderPath = dataFolder .. "/Gaia_DR2_full_24columns/",
            OutFileOrFolderPath = dataFolder .. "/DR2_full_Octree_test_50,50/",
            MaxDist = 500,
            MaxStarsPerNode = 50000,
            SingleFileInput = false,
            -- Specify filter thresholds
            --FilterPosX = {0.0, 0.0},
            --FilterPosY = {0.0, 0.0},
            --FilterPosZ = {0.0, 0.0},
            FilterGMag = {20.0, 20.0},
            FilterBpRp = {0.0, 0.0},
            --FilterVelX = {0.0, 0.0},
            --FilterVelY = {0.0, 0.0},
            --FilterVelZ = {0.0, 0.0},
            --FilterBpMag = {20.0, 20.0},
            --FilterRpMag = {20.0, 20.0},
            --FilterBpG = {0.0, 0.0},
            --FilterGRp = {0.0, 0.0},
            --FilterRa = {0.0, 0.0},
            --FilterRaError = {0.0, 0.0},
            --FilterDec = {0.0, 0.0},
            --FilterDecError = {0.0, 0.0},
            FilterParallax = {0.01, 0.0},
            FilterParallaxError = {0.00001, 0.5},
            --FilterPmra = {0.0, 0.0},
            --FilterPmraError = {0.0, 0.0},
            --FilterPmdec = {0.0, 0.0},
            --FilterPmdecError = {0.0, 0.0},
            --FilterRv = {0.0, 0.0},
            --FilterRvError = {0.0, 0.0},
        }
    }

This will create an octree from the dataset.  If the example above would be used it would read the full DR2 dataset with 24 values per star, filter away all stars that does not have either G magnitude, Bp-Rp color, a parallax value over 0.01 or a parallax error value below 0.5.  The resulting dataset is the 618 million dataset that is available for download (with "gaia_download.task").

Regardless of how many values that was read only 8 render values will be stored in the octree.  These are positions, magnitude, color and velocities.

All the filter properties are optional.  If defined they will filter away everything outside the set range.  If both min and max are set to the same value then all stars with that exact value will be filtered away. 0.0 (or 20.0 for magnitude) is the default value and will be interpreted as positive or minus infinity.

If a single binary file was read then a single binary octree file will be written.  If the stars were read from a folder then the structure of the octree will be stored as a binary index file and then every node of the octree will be stored as a single file. 

The time it takes to run this task depends mostly on how many files that will be written to disk (i.e. how big the octree is), which in turn depends on if any filtering was applied, the max distance of the octree and the max of stars per node (SPN).  If no filtering was applied the full DR2 dataset took about 30 minutes to finish on a moderate desktop computer when using 150kSPN and 250kpc as max distance.

A small _MaxDist_ is preferable as it means a smaller depth of the octree which will speed up traversals.  However, that can also mean that stars will be placed outside the initial octree.  Those stars still need to be stored in the octree so _MaxStarsPerNode_ has to be big enough to swallow all nodes that falls outside otherwise the TaskRunner will run out of memory stack and crash.  If that happens don't worry, just try to increase either of those values.

A smaller value for _MaxStarsPerNode_ is better for data uploads to the GPU while a bigger value means fewer files to write to disk and faster traversals.  There is no general rule for how to decide what to use.  A bit of trial and error is required for most datasets.  For bigger datasets generated from DR2 a recommended starting span would be 50k-150k SPN, and for smaller datasets (20 million stars and less) 1k - 30k SPN should work fine!


### 4. Run in OpenSpace
To render Gaia stars in OpenSpace first make sure that the correct scene is used.  This is done by setting `Asset = "gaia_mission"` in `OpenSpace/openspace.cfg`.  The scene is then defined in `OpenSpace/data/assets/gaia_mission.scene`.  Here you can define if anything else should be rendered in OpenSpace.  By default the full digital Universe catalog will be included as well as the Sun, Earth, Moon, a model of the Gaia spacecraft and its trail.

Which stars to render can be changed in `OpenSpace/data/assets/scene/milkyway/gaia/gaiamission.asset`.  By default the official radial velocity dataset will be downloaded and rendered.  That dataset consists of the 7.2 million stars that were released with any radial velocity in DR2. Its size is 335 MB and it is stored in ~3k files.

If you instead want to render your own dataset or a newly created subset change the _localStars_ variable to point to the path to the data folder.  It is preferably to store the stars on a SSD if possible.  Then point the **File** variable (in **RenderableGaiaStars**) to either the single file or the folder with the dataset. 

The **FileReaderOption** defines in what format the dataset will be read.  It can be either _Speck_, _Fits_, _BinaryRaw_, _BinaryOctree_ or _StreamOctree_.  If streaming is defined then **File** MUST point to a dataset folder, otherwise it MUST point to a single file. _Speck_ and _Fits_ will read an unprocessed raw file while _BinaryRaw_ will read the processed output from either _ReadSpeckTask_ or a _ReadFitsTask_.  All three will construct the octree on startup and will thus be slower than the others.  Reading a binary file is also faster that reading a raw Speck or Fits file!

_BinaryOctree_ on the other hand reads the single file output from the _ConstructOctreeTask_ while _StreamOctree_ make use of the multi-file output of the same task.

Most of the other values are optional and can be switched from the default values during runtime.  For a full documentation please see `OpenSpace/documentation/Documentation.html#gaiamission_renderablegaiastars`.

However, other properties that might be of interest on startup (apart from **Type**, **File** and **FileReaderOption**) are: 

* **PsfTexture**
Sets the point spread texture used when rendering billboards.  Not optional.

* **ColorTexture**
Colormap used as lookup table for the color of the stars.  Not optional.

* **AdditionalNodes**
Defines how many nodes around the camera that should be fetched when streaming from disk.  The first value defines how many upper layers of parents that should be found around the camera and the second value defines how many layers of descendants that will be fetched from the found parents.  Higher values will decrease performance.  A recommended start would be "{3.0, 2.0}".

* **MaxCpuMemoryPercent**
Defines the max percentage of the existing RAM budget that will be used for storing star data.  This _cannot_ be changed during runtime.

* **MaxGpuMemoryPercent**
Defines the max percentage of the dedicated GPU memory that will be used for streaming data.  This _can_ be changed during runtime.  If the screen goes black and the performance drops to below 5 fps then it could be that you are trying to reserve too much memory on the GPU, try to decrease this value!  A resulting value of < 4 GB should work fine for most GPUs.
