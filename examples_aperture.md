# Examples of common EAGLE tasks

[Using the catalogues I: Stellar mass - halo mass relation](https://j-davies-astro.github.io/eagle-guide/examples_smhm)

[Showing running means, medians and "fancy medians"](https://j-davies-astro.github.io/eagle-guide/examples_stats)

[Using the catalogues II: Galaxy stellar mass function](https://j-davies-astro.github.io/eagle-guide/examples_gsmf)

[Loading particles within a spherical aperture around a galaxy](https://j-davies-astro.github.io/eagle-guide/examples_aperture)

[Calculating a quantity using particles for all haloes in a sample](https://j-davies-astro.github.io/eagle-guide/examples_sample)

[Making histograms of particle properties](https://j-davies-astro.github.io/eagle-guide/examples_hists)

[Making radial profiles](https://j-davies-astro.github.io/eagle-guide/examples_profile)

[Tracing galaxies through time](https://j-davies-astro.github.io/eagle-guide/examples_tracing)

[Making pretty pictures with py-sphviewer](https://j-davies-astro.github.io/eagle-guide/examples_sphviewer)

## Loading particles within a spherical aperture around a galaxy

In this section we'll cover possibly the most common task in computational galaxy formation: finding all the particles that exist within a spherical aperture around a point, most likely a galaxy. Depending on the project you're undertaking with EAGLE, doing this process efficiently may become your bread-and-butter. The good news is that the function we put together when learning how to use `pyread_eagle` makes this straightforward to do.

I'll skip any more introductory waffle and get straight into an example. In the following code, we'll use the catalogues to find the centre of potential of a halo hosting a central Milky Way-like spiral galaxy, and find the star particles residing in a 100 physical kpc (pkpc) aperture around this point.

```python
import numpy as np
import matplotlib.pyplot as plt
from catalogue_reading import *
from particle_reading import *

# Let's load in a region around group number 32 in Ref-L0025N0376
# This group has a central spiral galaxy in a ~10^12 solar mass halo
groupnumber = 32
sim = 'L0025N0376'
model = 'REFERENCE'
tag = '028_z000p000'

# First, let's find the centre of potential (COP) of the halo.
# Get the locations of the centrals in the Subhalo table
first_subhalo = catalogue_read('FOF','FirstSubhaloID',sim=sim,model=model,tag=tag)
# Get the centres of potential
COP = catalogue_read('Subhalo','CentreOfPotential',sim=sim,model=model,tag=tag)[first_subhalo,:]

# We want group 32. Remember to subtract 1 for the index!
centre = COP[groupnumber-1,:]

# These three steps are not necessary, they're just for demonstration purposes
# Print the coordinates of the COP in pMpc
print(centre)
# Let's check the log halo mass
print(np.log10(catalogue_read('FOF','Group_M_Crit200',sim=sim,model=model,tag=tag)[groupnumber-1] * 1e10))
# and the central galaxy log stellar mass
print(np.log10(catalogue_read('Subhalo','ApertureMeasurements/Mass/030kpc',sim=sim,model=model,tag=tag)[first_subhalo,4][groupnumber-1] * 1e10))

# Time to initialise pyread_eagle. First, we need a string pointing to the first snapshot chunk
snapfile = '/hpcdata0/simulations/EAGLE/' + sim + '/' + model + '/data/snapshot_'+tag+'/snap_'+tag+'.0.hdf5'

# Initialise the EagleSnapshot instance
snapshot = pyread_eagle.EagleSnapshot(snapfile)

# Get our factors of h and a, and the box size
with h5.File(snapfile,'r') as f:
    h = f['Header'].attrs['HubbleParam']
    a = f['Header'].attrs['ExpansionFactor']
    boxsize = f['Header'].attrs['BoxSize'] * a/h # Note the unit conversion!

# Let's load in something visually interesting - the stars within a 100 pkpc radius.
# We therefore need to select a cube of side length 200 pkpc around the COP.
# Lengths are in Mpc, so xmin = centre[0]-0.1, xmax = centre[0]+0.1 etc.

region_size = 0.1

# First, we need to convert our region to "code units" to feed into pyread_eagle
centre_codeunits = centre * h/a
region_size_codeunits = region_size * h/a

# Run the select_region method
snapshot.select_region(centre_codeunits[0]-region_size_codeunits,centre_codeunits[0]+region_size_codeunits,
                        centre_codeunits[1]-region_size_codeunits,centre_codeunits[1]+region_size_codeunits,
                        centre_codeunits[2]-region_size_codeunits,centre_codeunits[2]+region_size_codeunits)

# Load in the star particle co-ordinates. Remember our function does h and a corrections for us.
coords = particle_read(4,'Coordinates',snapshot,snapfile)

# Centre our co-ordinate system on the COP
coords -= centre

# Because we're working with co-ordinates, we may need to wrap the box.
# Let's just do it, to be safe.
coords += boxsize/2.
coords %= boxsize
coords -= boxsize/2.

# Make a test plot of the particles
fig, ax = plt.subplots(figsize=(8,8))
ax.scatter(coords[:,0],coords[:,1],marker=',',c='k',s=1)
ax.set_xlabel(r'$x\,[{\rm pMpc}]$',fontsize=14)
ax.set_ylabel(r'$y\,[{\rm pMpc}]$',fontsize=14)
plt.show()
```
This code produces the following image:

![stars_test](/images/star_test.png)

Great! We have something that looks somewhat like a spiral galaxy, with a few clumpy satellites around it. However, we didn't get what we asked for - some of the particles are from outside the 200 pkpc cube that we specified with `select_region`, because _the module has loaded in all particles in the chunks that contain our hash keys_. We have a little more work to do now to 'mask' this region to a spherical aperture of radius 100 pkpc.

```python
# Get the radii^2 of all particles from the centre

# You could do:
r2 = coords[0]**2 + coords[1]**2 + coords[2]**2
# However I like to use np.einsum, as it's faster for very large datasets
# It does Einstein summation operations
r2 = np.einsum('...j,...j->...',coords,coords)

# Now we can make a mask to the spherical aperture
particle_selection = np.where(r2<region_size**2)[0]

coords = coords[particle_selection,:]

# Now let's make another scatter plot
fig, ax = plt.subplots(figsize=(8,8))
ax.scatter(coords[:,0],coords[:,1],marker=',',c='k',s=1)
ax.set_xlabel(r'$x\,[{\rm pMpc}]$',fontsize=14)
ax.set_ylabel(r'$y\,[{\rm pMpc}]$',fontsize=14)
plt.show()
```
Now we get:

![stars_masked](/images/star_particles_masked.png)

Perfect - we now have only the particles within our spherical aperture, and (if you squint a little) it does look like a spiral galaxy. Note that the shape looks different to before, because the aspect ratio of the plot has changed.

This all seems rather laborious for the sake of just loading a few particles in. The beauty of it, however, is that now our `EagleSnapshot` instance is initialised and we have our `particle_selection` mask for cutting the data down to a spherical aperture, working with other particle properties within this aperture is very simple. For example, if we wanted to know the total stellar mass within this aperture, this can now be done in one line:
```python
Mstar_100kpc = np.sum(particle_read(4,'Mass',snapshot,snapfile)[particle_selection]) * 1e10
print(Mstar_100kpc)
```
You'll see that this is a little more than the 30 pkpc stellar mass we printed earlier, due to the wider aperture.

Of course, this whole process can be wrapped up in a function that incorporates `particle_read`, but does the region selection and aperture masking as well. I won't go into this here, but you can check out my `eagle_tools.read` module [here](https://github.com/j-davies-astro/eagle_tools/blob/master/read.py) for a rather involved (and currently poorly-documented) example. It's essentially a wrapper for `pyread_eagle` in which I treat a simulation snapshot as a python `class` and automate the process of selecting a region and masking a spherical (or cubic) aperture, among other useful methods.

[Back to top](https://j-davies-astro.github.io/eagle-guide/examples_aperture)
