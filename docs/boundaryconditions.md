# Boundary conditions
**TODO**

 - Setting idealized boundary conditions
 - Setting type-3 water level using observations
 - Setting type-2/3 water level and/or currents using OTIS tide model
 - Setting boundary conditions with an ocean model i.e., ROMS
 - Setting river boundaries

---
# Boundary types 
SUNTANS has two main boundary conditions types:

 - Type 2: Edge boundary. Specify velocity and tracer only.
 - Type 3: Cell center boundary. Specify free-surface and tracer concentration.

There is also a flow boundary condition that allocates flow across a number of edge (see [River flow](#usgs-river-flow) section below). This is still a type-2 boundary condition - the information in the netcdf file tells SUNTANS that the cell is connected.

---
# Boundary condition python class
Boundary condtions are handled with the `Boundary` class located in [sunboundary.py](https://www.github.com/ofringer/suntanspy/blob/master/SUNTANS/sunboundary.py)

To instantiate a boundary condition object in python:

```python
from sunboundary import Boundary

#Variables
suntanspath = '/path/to/my_suntans/grid/'
starttime = '20000101.0000' # Boundary condition start time 'yyyymmdd.HHMM'
endtime = '20000101.0000' # Boundary condition end time 'yyyymmdd.HHMM'
dt = 3600. # Time step in seconds

---
# Initialise the class
bnd = Boundary(suntanspath, (starttime, endtime, dt) )

# Print out the class methods and attributes
print dir(bnd)
```

---
# Modifying the boundary types

When the `Boundary` class reads in the grid files (cells.dat, edges.dat, points.dat), it 
reads in the populates the grid attributes variables. The grid attributes relevant to the marker type are:

 - `bnd.mark`: vector of edge type (0, 1, 2 or 3).
 - `bnd.edge_id`: vector of edge identification tags used for group flow boundaries.

---
** Using a shapefile **

To use a shapefile to modify the boundary condition types:

First, create a shapefile in your favorite GIS software (recommend [QGIS](www.qgis.org)).
The shapefile must:

 - be a **polygon** type
 - contain two attributes named (both integer type): 
    - *mark*
    - *edge_id* 
 - use the same projection as the grid i.e. "UTM zone 15 N" for Galveston Bay

Then use the `modifyBCmarker` function in [sunboundary.py](https://www.github.com/ofringer/suntanspy/blob/master/SUNTANS/sunboundary.py).

```python
from sunboundary import modifyBCmarker

suntanspath = '/path/to/my_suntans/grid/'
bc_shape_file = '/path/to/my_boundary_shapefile.shp'

modifyBCmarker(suntanspath, bc_shape_file)
```

and you are done. **Note** that this overwrites *edges.dat* in your suntanspath folder. Just reload the `Boundary` class and the appropriate variables will be populated.

---
** Manually setting the boundary types **

The following shows how edges.dat can be updated using the SUNTANS `Grid` class. 
The main idea is to update the ```mark``` and ```edge_id``` attributes.

```python

from sunpy import Grid
import operator

suntanspath = '/path/to/my_suntans/grid/'

grd = Grid(suntanspath)

# Convert to a hybridgrid type because this stores more attributes
hgrd = grd.convert2hybrid()

grd.mark[grd.mark>0]=1 # reset all edges to land

# convert edges +/- half a grid cell from the edge
dx = hgrd.dg.max()
xmin = hgrd.xe.min()+dx/4.0
xmax = hgrd.xe.max()-dx/4.0

# Modify the left and right edges and convert to type 2
indleft = operator.and_(hgrd.mark>0, hgrd.xe < xmin) # all boundaries
grd.mark[indleft]=2
indright = operator.and_(hgrd.mark>0, hgrd.xe > xmax) # all boundaries
grd.mark[indright]=2

# Write over the old edges.dat file
edgefile = suntanspath+'/edges.dat'
grd.saveEdges(edgefile)
print 'Updated markers written to: %s'%(edgefile)

```

---
# Specifying idealized boundary data

This tutorial is taken from the [San Francisco Bay example](https://github.com/ofringer/suntans/blob/master/examples/sfbay-3d/scripts/suntans_driver_3d.py).

The following code specifies:

 - a sinusoidal function to specify water level along type-3 (ocean) boundaries
 - sets constant temperature and salinity at type-3 (ocean) boundaries
 - has two rivers with a different time-constant discharge rate


```python
bcfile = 'SFBay3D_BC.nc' # output boundary netcdf file name

hamp = 0.5 # tidal range
h0 = -5.0
omega = 2*PI/(12.42 * 3600.0) # Tidal frequency
T0 = 0 # Open boundary and initial condition background temperature
S0 = 32 # Open boundary and initial condition background salinity

# These must correspond to the edge_id field in the boundary polygon shapefile
# For this case 101 : Sacramento Rv, 102 : San Joaquin Rv
edge_id = [101,102] # Unique identifier of each flux edge (list)
Q = [0.0,0.0] # Volume flux rate of each segment boundary [m^3/s]

    
#Load the boundary object from the grid
#   Note that this zeros all of the boundary arrays
bnd = Boundary(suntanspath,(starttime,endtime,dt))

#Fill the variables in the boundary object with your data here...
#
#Type 3 variables are at the cell-centre (xv,yv) and are named:
#            uv, vc, wc, h, T, S
#            
#            Dimensions: [Nt,Nk,N3]
#            
#Type 2 variables are at the cell edges (xe, ye) and are named:
#            boundary_u
#            boundary_v
#            boundary_w
#            boundary_T
#            boundary_S
#            (no h)
#            
#            Dimensions: [Nt, Nk, N2]

t = bnd.ncTime()

# Set an oscillatory water level boundary along type 3 cells (T and S constant)
for ii in range(bnd.N3):
    bnd.h[:,ii] = h0 + hamp*np.cos(omega*t)
    for k in range(bnd.Nk):
        bnd.T[:,k,ii] = T0
        bnd.S[:,k,ii] = S0
        
# Set a constant discharge along the type 2 boundary edge
for ID,flux in zip(edge_id,Q):
    ind = np.argwhere(bnd.segp==ID)
    bnd.boundary_Q[:,ind]=flux
    
# Set all type-2 (river) boundary T and S constant
bnd.boundary_S[:,:,:] = 0.0
bnd.boundary_T[:,:,:] = T0

# Write the boundary file
bnd.write2NC(suntanspath+'/'+bcfile)

```

---
# Specifying realistic boundary data

---
**Tides from OSU Tide Model (OTPS)**

The following example shows how to allocate time and spatially varying tidal boundary conditions from a realistic tidal harmonic data set. 

First, download and extract and OTPS binary file from [here](http://volkov.oce.orst.edu/tides/otps.html).

Run the following code:
```python
from sunboundary import Boundary
from sunpy import Grid

#Variables
suntanspath = '/path/to/my_suntans/grid/'
bcfile = 'my_bcfile.nc'
starttime = '20000101.0000' # Boundary condition start time 'yyyymmdd.HHMM'
endtime = '20000101.0000' # Boundary condition end time 'yyyymmdd.HHMM'
dt = 3600. # Time step in seconds
utmzone = 15
isnorth = True

# Path to the OTIS binary file
otisfile = '/path/to/DATA/Model_tpxo7.2' 


# Load the model grid and bathymetry into the object 'grd' and initialise the vertical spacing
grd = Grid(suntanspath)

# Load the depth data into the grid object
grd.loadBathy(suntanspath+'/depths.dat-voro')

# Load the boundary object. Note that we set utmzone attributes so we can project the 
# OTIS data onto the grid coordinates correctly
bnd = Boundary(suntanspath, (starttime,endtime,dt),\
    utmzone=utmzone, isnorth=isnorth)

# We need to know the depths when we set tidal velocities along type-2 edges
# The code does a flux correction if the depths in OTIS and suntans are not equal.
bnd.setDepth(grd.dv)

# Finally, interpolate the otis data onto the boundaries
bnd.utmzone=utmzone
bnd.isnorth=isnorth
bnd.otis2boundary(otisfile, conlist=None)

# Write the boundary file
bnd.write2NC(suntanspath+'/'+bcfile)

```

---
**HYCOM global model**

Using data from a coarser ocean model data set is simply performed with the `Boundary.oceanmodel2bdy()` method. 

See [here](opendap.md#hycom-global) for obtaining some HYCOM data.

The method is called via something like this:

```python
oceanfile = '/path/to/my_hycom_file.nc'
seth = True # Add HYCOM ssh to boundary ssh
setUV = False # Add HYCOM uv to the boundary velocity

# Set using an ocean model
bnd.oceanmodel2bdy(oceanfile, seth=True)

```

This code is a wrapper for a *lot* of code that does a full 4D interpolation of the HYCOM variables onto the SUNTANS type-2/3 boundary points. 

---
** USGS river flow **
