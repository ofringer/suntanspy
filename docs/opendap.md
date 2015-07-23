# Downloading model data with opendap

These examples use the [get_metocean_dap](../blob/master/DataDownload/get_metocean_dap.py). 

The [xray](https://github.com/xray/xray) package can probably be used for the same thing. 

##HYCOM Global

```python
from get_metocean_dap import get_metocean_dap


# Set the time-space domain here
xrange = [118.0,127.0]
yrange = [-19.0,-11.0]
zrange = [0.,6000.]
trange = ['20140401.000000','20140501.000000']
vars = ['temp','salt','u','v','ssh']

outfile = 'DATA/HYCOM_ScottReef_Apr2014.nc'
#gridfile = 'DATA/HYCOM_Global_grid.nc' # Set to None
gridfile = None


# Call the function
get_metocean_dap(xrange, yrange, zrange, trange, outfile,\
    name='HYCOM', gridfile=gridfile, oceanvars = vars)

```
##GFS

Getting the Global Forecasting System data is a little more tricky because 
each time step is stored in a differnt opendap file. See the `get_gfs_function`
in [get_metocean_dap](../blob/master/DataDownload/get_metocean_dap.py) for
tips on how to modify this.

```python
from get_metocean_dap import get_gfs_tds

xrange = [118.0,127.0]
yrange = [-19.0,-11.0]
zrange = [0.,0.]
#trange = ['20150113.000000','20150120.000000']
trange = ['20150318.000000','20150326.000000']

outfile = 'DATA/GFS_ScottReef_March2015.nc'

get_gfs_tds(xrange,yrange,zrange,trange,outfile)

```

## Arbitrary ocean model opendap server

**Note that this has not been fully tested.**

The code should work for any ocean model (or other dataset). Simply 
replace the url and variable names.

```python

# Dictionary contains the important information
oceandict = {\
    'ncurl':'http://tds.hycom.org/thredds/dodsC/glb_analysis',\
    'type':'ocean',\
    'u':'u',\
    'v':'v',\
    'temp':'temperature',\
    'salt':'salinity',\
    'ssh':'ssh',\
    }

    get_metocean_dap(xrange, yrange, zrange, trange, outfile,\
	oceanvars = vars, oceandict=oceandict)



```

##ROMS

This example downloads ROMS data from a Texas A&M opendap server. This code was
written prior to the `get_metocean_dap` code so is slightly different.

```python
"""
Download a subset of the Tx-La shelf ROMS model output
"""
from romsio import roms_subset

#ncfiles = ['http://barataria.tamu.edu:8080/thredds/dodsC/txla_nesting6/ocean_his_%04d.nc'%i for i in range(100,196)]
ncfiles = ['http://barataria.tamu.edu:8080/thredds/dodsC/NcML/txla_nesting6.nc']
bbox = [-95.53,-94.25,28.3,30.0]
grdfile = '../DATA/txla_grd_v4_new.nc'

###
# 2007
timelims = ('20140101000000','20140701000000')
outfile = '../DATA/txla_subset_HIS_2014.nc'

# Writes the data file
roms = roms_subset(ncfiles,bbox,timelims,gridfile=grdfile)
roms.Writefile(outfile)
roms.Go()
```


