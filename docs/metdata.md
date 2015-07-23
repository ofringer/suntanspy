# Create a NetCDF met file from NARR model

This tutorial shows how to download the North American Regional Reanalysis
data for a give space-time domain.

Most of the work is done by [getNARR.py](../blob/master/DataDownload/getNARR.py).

This script can be found in the [Lake Michigan SUNTANS example](https://github.com/ofringer/suntans/tree/quad-netcdf-debug/examples/lakemichigan).

```python
from createSunMetFile import narr2suntans

bbox = [-88.2, -84.5, 41.5,46.2]
utmzone = 16
tstart = '20120301'
tend = '20120308'
ncfile = 'rundata/LakeMichigan_NARR_2012.nc'

narr2suntans(ncfile,tstart,tend,bbox,utmzone)
```
