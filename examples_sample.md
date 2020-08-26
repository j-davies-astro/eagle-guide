# Examples of common EAGLE tasks

[Using the catalogues I: Stellar mass - halo mass relation](https://j-davies-ari.github.io/eagle-guide/examples_smhm.md)

[Showing running means, medians and "fancy medians"](https://j-davies-ari.github.io/eagle-guide/examples_stats.md)

[Using the catalogues II: Galaxy stellar mass function](https://j-davies-ari.github.io/eagle-guide/examples_gsmf.md)

[Loading particles within a spherical aperture around a galaxy](https://j-davies-ari.github.io/eagle-guide/examples_aperture.md)

[Calculating a quantity using particles for all haloes in a sample](https://j-davies-ari.github.io/eagle-guide/examples_sample.md)

[Making histograms of particle properties](https://j-davies-ari.github.io/eagle-guide/examples_hists.md)

[Tracing galaxies through time](https://j-davies-ari.github.io/eagle-guide/examples_tracing.md)

[Making pretty pictures with py-sphviewer](https://j-davies-ari.github.io/eagle-guide/examples_sphviewer.md)

## Calculating a quantity using particles for all haloes in a sample

We can apply the framework in the previous section to large samples of galaxies/haloes by introducing a `for` loop. A great feature of `pyread_eagle` is that you need only initialise the `EagleSnapshot` object once, then you can run `select_region` over and over to grab different chunks of the box.

Here's an example of how you can compute a quantity from the particle data for each object in a sample. In the following code, we'll produce the stellar mass-halo mass relation from earlier, except that now we'll **compute the stellar masses manually from the particles themselves**. For now, we'll just look at the smaller Ref-L0025N0376 volume for speed.

**N.B.** In the following code I use the `tqdm` module, which you can install on the command line with `pip3.5 install tqdm --user`. The module adds a progress bar to your `for` loops - all you need to do is wrap a `tqdm()` function call around your iterator as shown below. I can't recommend this module enough for working through large datasets!

```python
import numpy as np
import matplotlib.pyplot as plt
from catalogue_reading import *
from particle_reading import *
from tqdm import tqdm

sim = 'L0025N0376'
model = 'REFERENCE'
tag = '028_z000p000'

snapfile = '/hpcdata0/simulations/EAGLE/' + sim + '/' + model + '/data/snapshot_'+tag+'/snap_'+tag+'.0.hdf5'

# Load what we need from the catalogues
M200 = catalogue_read('FOF','Group_M_Crit200',sim=sim,model=model,tag=tag) * 1e10
first_subhalo = catalogue_read('FOF','FirstSubhaloID',sim=sim,model=model,tag=tag)
COP = catalogue_read('Subhalo','CentreOfPotential',sim=sim,model=model,tag=tag)[first_subhalo,:]

# Establish our halo sample - groups with M200>10^10.5
mass_selection = np.where(M200>np.power(10.,10.5))[0]
M200 = M200[mass_selection]
COP = COP[mass_selection,:]

# Initialise the EagleSnapshot instance
snapshot = pyread_eagle.EagleSnapshot(snapfile)

# Get our factors of h and a, and the box size
with h5.File(snapfile,'r') as f:
    h = f['Header'].attrs['HubbleParam']
    a = f['Header'].attrs['ExpansionFactor']
    boxsize = f['Header'].attrs['BoxSize'] * a/h # Note the unit conversion!

# Establish the size of the region - 30 pkpc, but in "code units" for reading the particles
region_size = 0.03
region_size_codeunits = region_size * h/a

# Make an empty array to store our results in
Mstar_30kpc = np.zeros(len(mass_selection))

# Loop over the haloes in the sample
for i in tqdm(range(len(mass_selection))):

    # Get the centre of potential for the halo in question
    centre = COP[i]
    centre_codeunits = centre * (h/a)

    # Run select_region
    snapshot.select_region(centre_codeunits[0]-region_size_codeunits,centre_codeunits[0]+region_size_codeunits,
                            centre_codeunits[1]-region_size_codeunits,centre_codeunits[1]+region_size_codeunits,
                            centre_codeunits[2]-region_size_codeunits,centre_codeunits[2]+region_size_codeunits)

    # Load the coordinates
    coords = particle_read(4,'Coordinates',snapshot,snapfile)

    # Centre our coordinates and wrap the box
    coords -= centre
    coords += boxsize/2.
    coords %= boxsize
    coords -= boxsize/2.

    # Compute radial distances and mask to 30 pkpc
    r2 = np.einsum('...j,...j->...',coords,coords)
    particle_selection = np.where(r2<region_size**2)[0]

    # Compute stellar masses in apertures and write to output array
    Mstar_30kpc[i] = np.sum(particle_read(4,'Mass',snapshot,snapfile)[particle_selection]) * 1e10

# Make the plot

fig, ax = plt.subplots(figsize=(8,6))

ax.scatter(np.log10(M200),np.log10(Mstar_30kpc/M200),marker='o',edgecolors='k',facecolors='none',s=5)

ax.set_xlabel(r'$\log(M_{200}/{\rm M}_\odot)$',fontsize=16)
ax.set_ylabel(r'$\log(M_\star/M_{200})$',fontsize=16)

plt.show()
```
This is the first script we've used that takes a decent length of time to run - at time of testing, it took me about 8 minutes to run on `phoenix` but your mileage may vary. As soon as you start dealing with particles, your computation time will shoot up! You should see the following plot at the end:

![smhmparticle](/images/smhm_from_particles.png). 

Comparison with the plots we made earlier reveal that this is a pared-down version of the SMHM relation for Ref-L0100N1504, with far fewer objects due to the 16x smaller volume. Have a go at running the above code for the 100 Mpc simulation and you'll see from `tqdm` that the code is going to take far, far longer to run. This is partly due to us simply having many more objects to loop over, but is also the case because the larger simulation has many more very massive galaxies/haloes, which live in busier environments. `pyread_eagle` therefore has much more data to load in each time. These objects have to be worked through first, as the FOF catalogues go from more- to less-massive; you'll therefore notice that the iteration gradually speeds up until the zippy speeds of `Ref-L0025N0376` analysis are reached. This is why we always test our code on the smaller volumes first, before going for the big box.

There are a few ways to overcome this issue:
- Patience! You can leave your code running in a NoMachine session and it will happily tick away until you've got what you need
- Parallel processing. The `mpi4py` module allows you to run Pythong code in parallel with MPI - you could split up the objects you want to work through onto different MPI ranks, potentially speeding up your analysis dramatically. Doing this is a little beyond the scope of this guide, however if you fancy giving it a try there are some nice explanations of how it works [here](https://rabernat.github.io/research_computing/parallel-programming-with-mpi-for-python.html). Be careful of how much memory you're using when doing this, and be considerate to others using the machines.
- In some use-cases, it may be faster to load in the whole simulation volume at once and mask out the regions you need. In my experience, however, it tends to be faster to use `pyread_eagle` in its intended fashion.

### Saving your pre-crunched numbers

Given that our analysis can take a very long time, it's a good idea to save the results of your code for future use. It would be mad to re-run code like the above just to make a small adjustment to the plot, for example - it's far more efficient to output your data to an hdf5 file, and load that in when you want to create your plot.

Doing this is easy with the `h5py` module, like so:
```python
import h5py as h5

f = h5.File('test.hdf5','w')

f.create_dataset('M200',data=M200)
f.create_dataset('Mstar_30kpc',data=Mstar_30kpc)

f.close()
```
You can then explore your newly-created file with HDFView and open it in python whenever you need to. `h5py` also has a `create_group` function, should you be writing many datasets that you need to keep organised. I like to wrap my data up in a dictionary, so that writing to an hdf5 file at the end is very efficient, for example if we have a `dict` called `data`:

```python
for key in data.keys():
    f.create_dataset(key,data[key])
```

[Back to top](https://j-davies-ari.github.io/eagle-guide/examples_sample.md)
