# Working with particle data

Now let's take a look at working with the simulated particles themselves. Rather than using `h5py` as we did for the catalogues, we're now going to use the `pyread_eagle` module that we installed earlier. This will allow us to read small portions of the simulation, rather than the whole lot, speeding things up enormously for most use-cases.

The documentation on the module's [GitHub page](https://github.com/kyleaoman/pyread_eagle) does a good job of explaining the various functions available and their arguments, so I won't explain these in great detail here. I'll instead focus on giving examples of how to use the module, and you can refer to those docs for any extra information.

## How it works

So how does `pyread_eagle` load in only _part_ of a simulation snapshot? More specifically, how does it know which particles lie in the small region you've asked it to load, without first loading all the particles to check?

This is all made possible by the fact that the simulation data is stored on disk in a very specific way. To quote from the EAGLE documentation:

_Particles of each type are distributed across the different files of a snapshot in such a way that it is easy to retrieve those that are in a simply connected region - for example all particles within a given distance from a  given  location, say the centre of mass of a halo. This is done by dividing the computational volume into cubic cells, 2^6 cells on a side, calculating which 3D cell each particle is in (referred to as the hash key), and sorting particles based on this hash key. Cells are then distributed over the individual files that make up a single snapshot. The hash tables allow one to determine which files need reading to retrieve all particles in a spherical region around a given centre. Because particles are arranged in cubic cells, using the hash tables returns particles in cells - limiting the list to particles in a spherical region is left to the user.  Hash tables are constructed separately for each particle type._

The cells (hash keys) are stored on disk in an order defined by a Peano-Hilbert space filling curve, meaning that cells (and therefore particles) that are close together in space are stored closely together on disk. When we tell `pyread_eagle` the limits of the spatial region we'd like to load, it finds all the hash keys within that region, which can then be used to efficiently load _part_ of the snapshot.

Recall that the snapshots are split into several hdf5 files, defining simulation 'chunks'; `pyread_eagle` uses the hash keys to identify which of these chunks need loading with `h5py`. The important thing to remember is that **the module won't just read the region you ask it to, it'll read all the chunks containing the hash keys you selected.** This means you'll get the cubic region you asked for, plus lots of extra stuff outside that region. Further masking of the data will be required to get the exact data that you want. We'll cover this below.

## Using pyread_eagle

### Initialising a snapshot

Fortunately for us, `pyread_eagle` is very straightforward to use. First, you'll need to import the module and make a string that points to just **one** chunk of the snapshot you want to load:
```python
# Import modules
import numpy as np
import h5py as h5
import pyread_eagle

# Simulation details
sim = 'L0025N0376'
model = 'REFERENCE'
tag = '028_z000p000'

snapfile = '/hpcdata0/simulations/EAGLE/' + sim + '/' + model + '/data/snapshot_'+tag+'/snap_'+tag+'.0.hdf5'
```
We can now use this string to initialise an `EagleSnapshot` object with `pyread_eagle`:
```python
snapshot = pyread_eagle.EagleSnapshot(snapfile)
```
Note that because `EagleSnapshot` is is a python _object_, we could in theory have multiple snapshots on the go at once, for example if we wanted to study two different models. For now, though, we'll just stick to manipulating one!

### Selecting a region

The `EagleSnapshot` object has several _methods_, though the important ones for us are `select_region` and `read_dataset`. The former method is where we tell the module the spatial region we want to load, and it does its magic with the hash keys. It's used like this:
```python
# The arguments are xmin,xmax,ymin,ymax,zmin,zmax
snapshot.select_region(0.,1.,0.,1.,0.,1.)
```
In doing this, we've told `snapshot` (an _instance_ of `EagleSnapshot`) that we'd like to load from a cubic region of side length **1 cMpc/h**. Yes, you guessed it, `select` requires that the **min and max values of x, y and z that we give must be in _h_-less comoving GADGET units**. 

This adds a little extra layer of complexity - you'll need to have the appropriate values of _h_ and _a_ in hand if you want to specify a region in physical units (which you will, most of the time). We can grab these from the header of `snapfile` that we specified earlier. To load in a similar cubic region, but with a side length of **1 pMpc**, we could instead do:
```python
with h5.File(snapfile,'r') as f:
    h = f['Header'].attrs['HubbleParam']
    a = f['Header'].attrs['ExpansionFactor']

side_length = 1. * h/a

snapshot.select_region(0.,side_length,0.,side_length,0.,side_length)
```

### Loading particle quantities

Now that we've selected our region, we can load in the properties of all the particles in that region with the `read_dataset` method. There are only two arguments to worry about here - the **particle type** and the **dataset name**. For example, lets load the masses (the `Mass` dataset) of all gas particles (particle type 0):
```python
gas_mass = snapshot.read_dataset(0,'Mass')

print(gas_mass)
print(len(gas_mass))
```
These will be in **h-less comoving GADGET units**, and you can find the conversion factors as attributes of the dataset in the hdf5 file, just like for the catalogues.

If you run this, you'll see that this is a rather uninteresting "chunk of universe"! There are only 154 gas particles, and they all have the same mass. We'll look at loading something more interesting in soon!

## Automating unit conversions and extra complexities

As we know from working with the catalogues, doing the unit conversions manually for each dataset we want to load can be a bit of a pain, so let's automate this for the particles as well.

In theory, we could just look at the attributes of the dataset in question in `snapfile` to get our conversion factors. This almost always works, but for the rarer particle types (typically stars and black holes) there can sometimes be no particles of the required type in chunk 0. A workaround for this is to simply look in the next chunk!

The function below takes this approach. You'll notice that it's very similar to our function for loading the catalogues, but now works with particles instead. The user must pass in the particle type `ptype`, the `quantity` of interest, our `snapshot` instance (after running `snapshot.select_region`) and `snapfile`, the path to the data.
```python
def particle_read(ptype,quantity,snapshot,snapfile,
            phys_units=True,
            cgs_units=False):

    # Trim off the "0.hdf5" from our file string so we can loop over the chunks if needed
    trimmed_snapfile = snapfile[:-6]
    # Initialise an index
    snapfile_ind = 0
    while True:
        try:
            with h5.File(trimmed_snapfile+str(snapfile_ind)+'.hdf5', 'r') as f:
                # Grab all the conversion factors from the header and dataset attributes
                h = f['Header'].attrs['HubbleParam']
                a = f['Header'].attrs['ExpansionFactor']
                h_scale_exponent = f['/PartType%i/%s'%((ptype,quantity))].attrs['h-scale-exponent']
                a_scale_exponent = f['/PartType%i/%s'%((ptype,quantity))].attrs['aexp-scale-exponent']
                cgs_conversion_factor = f['/PartType%i/%s'%((ptype,quantity))].attrs['CGSConversionFactor']
        except:
            # If there are no particles of the right type in chunk 0, move to the next one
            print('No particles of type ',ptype,' in snapfile ',snapfile_ind)
            snapfile_ind += 1
            continue
        # If we got what we needed, break from the while loop
        break

    # Load in the quantity
    data_arr = snapshot.read_dataset(ptype,quantity)

    # Cast the data into a numpy array of the correct type
    dt = data_arr.dtype
    data_arr = np.array(data_arr,dtype=dt)

    if np.issubdtype(dt,np.integer):
        # Don't do any corrections if loading in integers
        return data_arr

    else:
        # Do unit corrections, like we did for the catalogues
        if phys_units:
            data_arr *= np.power(h,h_scale_exponent) * np.power(a,a_scale_exponent)

        if cgs_units:
            data_arr = np.array(data_arr,dtype=np.float64)
            data_arr *= cgs_conversion_factor

        return data_arr
```
With this function, it's now easy to load in particle data from a region already specified with the `select_region` method:
```python
# Get the masses of gas particles in solar masses
gas_mass = particle_read(0,'Mass',snapshot,snapfile) * 1e10

print(gas_mass)
```
Keep this function in hand, as we'll use it along with the catalogue-reading function later on. I'm going to keep it in a file called `particle_reading.py`.

## Particle positions and periodic boxes

Until now I've referred to the simulations as "boxes of particles". You may then wonder: what happens at the edges of this box? If the centre of a galaxy sits right on the edge of the box, is only half of the galaxy in the simulation, and how would that even work in terms of galaxy formation??

To get around this issue, cosmological simulations employ **periodic boundary conditions**. This means that the structure at one face of the box is perfectly contiguous with what's on the opposite face - one could take two copies of the simulation, place them next to each other, and the interface between them would be perfectly seamless. One could even tile the boxes infinitely and make a simulation of arbitrary size, though of course no extra information about galaxy formation would be gained by doing so. All the gravitational forces, hydrodynamics and EAGLE physics work in this periodic fashion. It's more than a little mind-blowing to think about - I prefer to think of the simulation as simply not having edges, while having a finite volume.

Returning to the example of a galaxy sitting right at the simulation's 'edge': while one half the galaxy sits at one end of the box, the rest of it sits at the opposite end. This has important consequences for us when we load in the co-ordinates of particles - we must "wrap the box" and move half of the galaxy over to where it's "supposed to be".

`pyread_eagle` is clever, and takes the periodicity of the box into account. If you enter values into `select_region` that are outside of the bounds of the simulation, it will load in the particles that should be in that region from the opposite end of the box. We can test this using our new `particle_read` function:
```python
import numpy as np
import pyread_eagle
import matplotlib.pyplot as plt

from particle_reading import *

sim = 'L0025N0376'
model = 'REFERENCE'
tag = '028_z000p000'

snapfile = '/hpcdata0/simulations/EAGLE/' + sim + '/' + model + '/data/snapshot_'+tag+'/snap_'+tag+'.0.hdf5'

snapshot = pyread_eagle.EagleSnapshot(snapfile)

with h5.File(snapfile,'r') as f:
    h = f['Header'].attrs['HubbleParam']
    a = f['Header'].attrs['ExpansionFactor']

# Select a region 2 pMpc across that straddles the upper corner of the 25 Mpc box in x, y and z
min_coord = 24. * h/a
max_coord = 26. * h/a

# Run select_region with these limits
snapshot.select_region(min_coord,max_coord,
                        min_coord,max_coord,
                        min_coord,max_coord)

# Use our new function to automate unit corrections
# We'll get the coordinates in physical Mpc
coords = particle_read(0,'Coordinates',snapshot,snapfile)

# Make a scatter plot of the particles
fig, ax = plt.subplots(figsize=(16,16))

ax.scatter(coords[:,0],coords[:,1],marker=',',c='k',s=1)

ax.set_xlabel(r'$x\,[{\rm pMpc}]$')
ax.set_ylabel(r'$y\,[{\rm pMpc}]$')

plt.show()
```
If you run this, you'll see that because the region straddles the edges of the box in all three dimensions, `pyread_eagle` loaded in a roughly cubic section from all corners of the box:
![Image](/images/periodic_prewrap.png)

We can fix this by "wrapping the box". I do this by doing a co-ordinate transformation such that the centre of our region (which in this case is [25,25,25]) is at [0,0,0], and then make use of python's remainder (`%`) operator.
```python
# Get the simulation box size (in physical units)
with h5.File(snapfile,'r') as f:
    boxsize = f['Header'].attrs['BoxSize'] * a/h # Note the unit conversion!
    
# Centre the co-ordinates on the centre of the region
coords -= 25.

# Wrap the box
coords += boxsize/2.
coords %= boxsize
coords -= boxsize/2.

# Make another scatter plot
fig, ax = plt.subplots(figsize=(16,16))
ax.scatter(coords[:,0],coords[:,1],marker=',',c='k',s=1)
ax.set_xlabel(r'$x\,[{\rm pMpc}]$')
ax.set_ylabel(r'$y\,[{\rm pMpc}]$')
plt.show()
```
Now, the region is contiguous and all in one place. As you can see, this is a very sparse and uninteresting (2 Mpc)^3 of space! This technique with the remainder operator is very quick and clean (I pinched it from someone else!) - one could also use `np.where` to find the particles that need their co-ordinates shifting by the size of the box.

**It's important to remember to do this box wrapping when working with co-ordinates. You can always check whether it's necessary for your chosen region by testing whether your bounds go outside the box!**

