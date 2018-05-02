# HextofOfflineAnalyzer
This code is used to analyze data measured at FLASH using the HEXTOF instrument. The HEXTOF uses a delay line detector (DLD) to measure the position and arrival time of single electron events.

The analysis of the data is based on clean tables as dask dataframes. The main table contains all detected electrons and can be binned according to the needs of the experiment. The second dataframe contains the FEL pulses needed for normalization.

The class `DldProcessor` contains the dask dataframes as well as the methods to do the binning in the parallelized fashion.

The `DldFlashDataframeCreator` class is used for creating the dataframes from the hdf5 files generated by the DAQ system. It may need to be modified for different beam times, whereas the `DldProcessor` should be more universal.

The data obtained from the DAQ system is read through the **pah** package provided by FLASH, which is accessible on bitbucket at https://stash.desy.de/projects/CS . The location of the downloaded repo must be set in the **SETTINGS.ini** file, under `PAH_MODULE_DIR`. This will be contained in the processor object as
`processor.PAH_MODULE_DIR`.

# Settings

To initialize the code correctly on a new machine, run the **InitializeSettings.py** file in the terminal using

Also, run the setup.py. To do this, cd to your repo folder and run the following commands,
```bash 
python setup.py build_ext --inplace
python InitializeSettings.py
```

This creates the **SETTINGS.ini** file where local settings are stored. It contains the file paths and the parallelization parameters, which can be modified by the user externally. The ini file is not synced (git-ignored) in the git repository to allow the code to adapt to different computing environments.

# Versions and package structure

* **processor** folder -- contains the latest version of the processor.
* **lib** folder -- contains the legacy version of the processor, retained for retro-compatibility.
* **XPSdoniachs** folder -- contains the Doniach-Sunjic lineshape function in C++ for fitting.

# How to use

To use this offline analysis package, first, import the dependent modules. Add the folders of the relevant repos to the system path,
```python
import sys,os
sys.path.append('/path/to/HextofOfflineAnalyzer/')

from processor.DldFlashDataframeCreator import DldFlashProcessor
```
/!\ For windows users, remember to use forward slashes

Some useful imports for initializing an IPython notebook include,
```python
import sys,os
import numpy as np
import matplotlib.pyplot as plt
from importlib import reload

import processor.utils as utils # contains commonly used functions for
treating this data.
import processor.DldFlashDataframeCreator as DldFlashProcessor
reload(dldFlashProcessor)
```



## 1. Read DAQ data

**(1)** Load the data with a given DAQ run number

```python
# create a processor isntance
processor = DldFlashProcessor()
# assign a run number
processor.runNumber = 18843
```

The data can now be loaded from either a full DAQ run:
```python
#read the data from the DAQ hdf5 dataframes
processor.readData(runNumber=processor.runNumber)
```
or from a selected range of the `pulseIdInterval`:
```python
mbFrom = 1000 # first macrobunch
mbTo = 2000 # last macrobunch
processor.readData(pulseIdInterval=(mbFrom,mbTo))
```
**(2)** Run the `postProcess` method, which generates a BAM-corrected `pumpProbeDelay` array, together with polar coordinates for the momentum axes.

```python
processor.postProcess()
```

The dask dataframe is now created and can be used directly or stored in parquet format (optimized for speed) for future use. A shorter code summary is in the following:

```python
processor = DldFlashProcessor()
processor.runNumber = 18843
processor.readData(runNumber=processor.runNumber)
processor.postProcess()
```

## 2. Save dataset to dask parquet files

For faster access to these dataframes in extended offline analysis, it is convenient to store the datasets in dask parquet dataframes. This is done using
```pthon
processor.storeDataframes('filename')
```

This saves two folders in path/to/file: **name_el** and **name_mb**. These are the two datasets `processor.dd` and `processor.ddMicrobunches`. If `'filename'` is not specified, it uses either `'run{runNumber}'` or `'mb{firstMacrobunch}to{lastMacrobunch}'`, for example, `run18843`, or `mb90000000to900000500`.



Datasets in parquet format can be loaded back into the processor using the `readDataframes` method.
```python
processor = DldFlashProcessor()
processor.readDataframes('filename')
```


An optional parameter for both `storeDataframes` and `readDataframes` is `path=''`. If it is unspecified, (left as default None) the values from`DATA_PARQUET_DIR` or `DATA_H5_DIR` in **SETTINGS.ini** is used.

Alternatively, it is possible to store these datasets similarly in hdf5 format, using the same function,
```pthon
processor.storeDataframes('filename', format='hdf5')
```
However, this is NOT advised, since the parquet format outperforms the hdf5 in reading and data manipulation. This functionality is mainly kept for retro-compatibility with older datasets.

## 3a. Binning

In order to get n-dimensional np.arrays from the generated datasets, it is necessary to bin data along the desired axes. An example starting from loading parquet data is in the following,
```python
processor = DldFlashProcessor()
processor.runNumber = 18843
processor.readDataframes('path/to/file/name')
```
This can be also done from direct raw data read with `readData` To create the bin array structure, run
```python
processor.addBinning('posX',480,980,10)
processor.addBinning('posY',480,980,10)
```
This adds binning along the kx and ky directions, from point 480 to point 980 with bin size of 10. Bins can be created defining start and end points and either step size or number of steps. The resulting array can be obtained using
```python
result = processor.ComputeBinnedData()
```
where the resulting np.array of float64 values will have the axis order same as the order in which we generated the bins. Other binning axes commonly used are,

|          name          |       string        | typical values | units |
| :--------------------: | :-----------------: | :------------: | ----: |
|     ToF delay (ns)     |      'dldTime'      |  620,670,10 *  |    ns |
| pump-probe time delay  |  'pumpProbeDelay'   |    -10,10,1    |    ps |
| separate dld detectors |   'dldDetectors'    |     -1,2,1     |    ID |
| microbunch (pulse) ID  |   'microbunchId'    |   0,500,1 **   |    ID |
|       Auxiliary        |      'dldAux'       |                |       |
|  Beam Arrival Monitor  |        'bam'        |                |    fs |
|    FEL Bunch Charge    |    'bunchCharge'    |                |       |
|     macrobunch ID      | 'macroBunchPulseId' |                |    ID |
|      laser diode       |   'opticalDiode'    | 1000,2000,100  |       |
|           ?            |     'gmdTunnel'     |                |       |
|           ?            |      'gmdBda'       |                |       |



\* ToF delay bin size needs to be multiplied by `processor.TOF_STEP_TO_NS` in order to avoid artifacts.

\** binning on microbunch works only when not binning on any other dimension

Binning is created using np.linspace (formerly was done with `np.arange`). The implementation allows to choose between setting a step size (`useStepSize=True, default`) or using a number of bins (`useStepSize=False`).

In general, it is not possible to satisfy all 3 parameters: start, end, steps. For this reason, you can choose to give priority to the step size or to the interval size. In the case of `forceEnds=False`, the steps parameter is given priority and the end parameter is redefined, so the interval can actually be larger than expected. In the case of `forceEnds = true`, the stepSize is not enforced, and the interval is divided by the closest step that divides it cleanly. This of course only has meaning when choosing steps that do not cleanly divide the interval.

## 3b. Extracting data without binning

Sometimes it is not necessary to bin the electrons to extract the data. It is actually possible to directly extract data from the appropriate dataframe. This is useful if, for example, you just want to plot some parameters, not involving the number of electrons that happen to have such a value (this would require
binning).

Because of the structure of the dataframe, which is divided in dd and ddMicrobunches, it is possible to get electron-resolved data (the electron number will be on the x axis), or `uBunchID` resolved data (the `uBid` will be on the x axis).

The data you can get from the dd dataframe (electron-resolved) includes:

|            name            | string |
| :------------------------: | :----: |
| x position of the electron | 'posX' |
|x position of the electron| 'posY'
|time of flight| 'dldTime'|
|pump probe delay stage reading| 'delayStageTime'
|beam arrival monitor jitter|    'bam'
|microbunch ID| 'microbunchId'
|which detector| 'dldDetectorId'
|electron bunch charge|  'bunchCharge'
|pump laser optical diode reading|  'opticalDiode'
|gas monitor detector reading before gas attenuator| 'gmdTunnel'
|gas monitor detector reading after gas attenuator| 'gmdBda'
|macrobunch ID| 'macroBunchPulseId'

The data you can get from the `ddMicrobunches` (uBID-resolved) dataframe includes:

| name                   | string                   |
|:----------------------:|:------------------------:|
|pump probe delay stage reading| 'delayStageTime'
|beam arrival monitor jitter|    'bam'
|auxillarey channel 0| 'aux0'
|auxillarey channel 1| 'aux1'
|electron bunch charge|  'bunchCharge'
|pump laser optical diode reading|  'opticalDiode'
|macrobunch ID| 'macroBunchPulseId'

Some of the values overlap, and in these cases, you can get the values either uBid-resolved or electron-resolved.

An example of how to retrieve values both from the `dd` and `ddMicrobunches` dataframes:
```python
bam_dd=processor.dd['bam'].values.compute()
bam_uBid=processor.ddMicrobunches['bam'].values.compute()
```
Be careful when reading the data not to include IDs that contain NaNs (usually at the beginning), otherwise this method will return all NaNs.

It is also possible to access the electron-resolved data on a `uBid` basis by using
```python
uBid=processor.dd['microbunchId'].values.compute()
value[int(uBid[j])]
```
or to plot the values as a function of `uBid` by using
```python
uBid=processor.dd['microbunchId'].values.compute()
MBid=processor.dd['macroBunchPulseId'].values.compute()
bam=processor.dd['bam'].values.compute()

pl.plot(uBid+processor.bam.shape[1]*MBid,bam)
```

The following code, as an example, averages `gmdTunnel` values for electrons that have the same `uBid` (it effectively also bins the electrons in `avgNorm` as a side effect):

```python
uBid=processor.dd['microbunchId'].values.compute()
pow=processor.dd['gmdBDA'].values.compute()

avgPow=np.zeros(500)
avgNorm=np.zeros(500)
for j in range(0,len(uBid)):
    if(uBid[j]>0 and uBid[j]<500 and pow[j]>0):
        avgNorm[int(uBid[j])]+=1
        avgPow1[int(uBid[j])]=(avgPow1[int(uBid[j])]*(avgNorm[int(uBid[j])])+pow[j])/(avgNorm[int(uBid[j])]+1.0)

```

## 4. Complete code example

Complete examples suitable for use in an IPython notebook:



**(1)** Importing packages and modules

```python
import sys,os
import math
import numpy as np
import h5py
import time

import matplotlib
import matplotlib.pyplot as plt
import matplotlib.cm as cm
import matplotlib.colors as colors
import scipy.signal as spsignal
# %matplotlib inline # uncomment in Ipython

from imp import reload
from scipy.ndimage import gaussian_filter
from processor import utils, DldFlashDataframeCreator as DldFlashProcessor
# reload(dldFlashProcessor)

```
**(2)** Loading raw data
```python
reload(dldFlashProcessor) # in case code has changed since import.

runNumber = 12345
read_from_raw = True # set false to go straight for the stored parquet data
save_and_use_parquet = True # set false to skip saving as parquet and reloading.

processor = DldFlashProcessor()
processor.runNumber = runNumber
if read_from_raw:
    processor.readData()
    processor.postProcess()
    if save_and_use_parquet:
        processor.storeDataframes()
        del processor
        processor = DldFlashProcessor()
        processor.runNumber = runNumber
        processor.readDataframes()
else:
    processor.readDataframes()

#start binning procedure
processor.addBinning('posX',480,980,10)
processor.addBinning('posY',480,980,10)

result = processor.ComputeBinnedData()
result = nan_to_num(result)
plt.imshow(result)
```

FLASH DAQ info
--

A brief information about each channel of the DAQ can be found at this link:
http://www.desy.de/~wwwuser/hdf5main.html