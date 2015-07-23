# SUNTANS initial conditions tutorial

SUNTANS initial conditions are handled with the `InitialCond` class that is buried in [sunboundary.py](https://www.github.com/ofringer/suntanspy/blob/master/SUNTANS/sunboundary.py). This class takes care of initializing the variables and writing to the NetCDF file amongst various other tasks.

To instantiate an initial condition object in python:

```python
from sunboundary import InitialCond

#Variables
suntanspath = '/path/to/my_suntans/grid/'
starttime = '20000101.0000' # Initial condition start time 'yyyymmdd.HHMM'

# Initialise the class
IC = InitialCond(suntanspath,starttime)

# Print out the class methods and attributes
print dir(IC)
```

##Generate idealized initial conditions

This tutorial is taken from the [San Francisco Bay example](https://github.com/ofringer/suntans/blob/master/examples/sfbay-3d/scripts/suntans_driver_3d.py).

The following code uses an analytical function to intialize the temperature and 
salinity fields in San Francisco Bay.

```python

import numpy as np

icfile = 'SFBay3D_IC.nc' # Initial condition netcdf file name

h0 = -5.0 # initial free surface elevation

# Initialise the class
IC = InitialCond(suntanspath,starttime)

#Set the depth
IC.dv = grd.dv

# the initial condition arrays are stored in the following fields in the class:
#   h, uc, vc, T, S
#
# uc, vc, T and S have dimensions [1, Nk, Nc]
# h has dimensions [1, Nc]

# We want to set h to constant (other fields are zero by default)
IC.h[:] = h0

# Now set T and S based on analytical functions
def returnSalinity(x,y):
    x0 = 5.7e5
    S = 32.0*0.5*(1.0+np.tanh(-(x-x0)/1e4))
    S[y<4.2e6] = 32.
    return S
		
def returnTemperature(x,y):
    x0 = 545691.
    y0 = 4185552.
    R = 1000.
    T = np.zeros_like(x)
    ind = (x-x0)**2 + (y-y0)**2 < R*R
    T[ind] = 1.0
    return T

S = returnSalinity(IC.xv,IC.yv) # returns a 1-D array length Nc
T = returnTemperature(IC.xv,IC.yv)

# Now fill in the values in the array
for k in range(IC.Nkmax):
    IC.T[:,k,:] = T
    IC.S[:,k,:] = S

# Write the initial condition file
IC.writeNC(suntanspath+'/'+icfile)

```

##Generate initial conditions using ROMS

The `InitialCond` class has a method called `roms2ic`. This is simply called via

```python
romsfile = '/path/to/roms_netcdf.nc' # ROMS netcdf file
setUV = False # Option to use roms velocity data
seth = False # Option to use ROMS free surface

IC.roms2ic(romsfile, setUV=setUV, seth = False) 
```

Most of the heavy lifting is dealt with by [romsio.py](https://www.github.com/ofringer/suntanspy/blob/master/DataIO/romsio.py).

To download ROMS model data see the [opendap tutorial](opendap#roms)

##Generate initial conditions using HYCOM

Similar to ROMS, other ocean models can be interpolated onto a
SUNTANS grid to create initial condition data. The `InitialCond` class has a method called `oceanmodel2ic`.

To download some model data see [here](opendap#hycom-global)

```python
hycomfile = '/path/to/hycom_netcdf.nc' # ROMS netcdf file
setUV = False # Option to use roms velocity data
seth = False # Option to use ROMS free surface

IC.oceanmodel2ic(romsfile, setUV=setUV, seth = False, convert2utm=True) 
```

Most of the work here is done by the `Interp4D` class in [interpXYZ.py](https://www.github.com/ofringer/suntanspy/blob/master/Utils/interpXYZ.py) and
the `get_metocean_local` function in [mythredds.py](https://www.github.com/ofringer/suntanspy/blob/master/DataDownload/mythredds.py).

##Generate initial conditions using an existing simulation
