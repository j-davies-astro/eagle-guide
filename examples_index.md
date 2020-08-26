# Examples of common EAGLE tasks

To help you get to grips with all the details I've covered on previous pages, I'll now show you several examples of how to do some standard "simulation tasks". These should hopefully cover many of the types of analysis you'll be doing when working with EAGLE.

I strongly recommend you go through these in order, as some sections rely on information and/or code in previous ones!

[Using the catalogues I: Stellar mass - halo mass relation](https://j-davies-ari.github.io/eagle-guide/examples_smhm.md)

[Showing running means, medians and "fancy medians"](https://j-davies-ari.github.io/eagle-guide/examples_stats.md)

[Using the catalogues II: Galaxy stellar mass function](https://j-davies-ari.github.io/eagle-guide/examples_gsmf.md)

[Loading particles within a spherical aperture around a galaxy](https://j-davies-ari.github.io/eagle-guide/examples_aperture.md)

[Calculating a quantity using particles for all haloes in a sample](https://j-davies-ari.github.io/eagle-guide/examples_sample.md)

[Making histograms of particle properties](https://j-davies-ari.github.io/eagle-guide/examples_hists.md)

[Tracing galaxies through time](https://j-davies-ari.github.io/eagle-guide/examples_tracing.md)

[Making pretty pictures with py-sphviewer](https://j-davies-ari.github.io/eagle-guide/examples_sphviewer.md)

## Using the catalogues I: Stellar mass - halo mass relation

Here's a very simple plot to make using only the galaxy catalogues. The stellar mass-halo mass relation is very important as it describes how efficient galaxy formation is in haloes of a given mass. We can make this plot by creating a sample of haloes from the `FOF` table and getting their central galaxy stellar masses from the `Subhalo` table, like so:

```python
import numpy as np
import matplotlib.pyplot as plt
from catalogue_reading import *

# We'll make the z=0 SMHM relation in the largest simulation
sim = 'L0100N1504'
model = 'REFERENCE'
tag = '028_z000p000'

# Grab the halo masses and FirstSubhaloIDs from FOF
M200 = catalogue_read('FOF','Group_M_Crit200',sim=sim,model=model,tag=tag) * 1e10
first_subhalo = catalogue_read('FOF','FirstSubhaloID',sim=sim,model=model,tag=tag)

# Set a minimum mass to cut down what we're plotting, and mask the FOF arrays
mass_selection = np.where(M200>np.power(10.,10.5))[0]
M200 = M200[mass_selection]
first_subhalo = first_subhalo[mass_selection]    

# Get the stellar masses in 30kpc apertures
# Note that I've multiplied M200 and Mstar by 1e10 to get them in solar masses
Mstar_30kpc = catalogue_read('Subhalo','ApertureMeasurements/Mass/030kpc',sim=sim,model=model,tag=tag)[first_subhalo,4] * 1e10

# Convert to the quantities we want to plot. Most simulation quantities are best plotted logarithmically!
log_M200 = np.log10(M200)
mstar_over_mhalo = np.log10(Mstar_30kpc/M200)

# Make a quick scatter plot of the relation
fig, ax = plt.subplots(figsize=(8,6))

ax.scatter(log_M200,mstar_over_mhalo,marker='o',edgecolors='k',facecolors='none',s=5)

ax.set_xlabel(r'$\log(M_{200}/{\rm M}_\odot)$',fontsize=16)
ax.set_ylabel(r'$\log(M_\star/M_{200})$',fontsize=16)
plt.savefig('/path/to/your/directory/smhm_basic.png')

plt.show()
```
This should produce the following figure:
![smhm_basic](/images/smhm_basic.png)

This tells you a couple of things about galaxy formation straight away:
- In a given volume, very massive objects are rare and low-mass objects are very numerous. This is because the universe is hierarchical.
- Galaxy formation is most efficient in haloes of M_200 ~ 10^12 solar masses, which tend to host Milky Way-like galaxies. 

## Showing running means, medians and "fancy medians"

As you can see, the above plot is very 'saturated' - there are so many points at low mass that they form an unreadable black blob. To interpret the data, we really need to show some kind of statistic as a function of what's on the x axis. To do this quickly and easily, I recommend using `scipy.stats.binned_statistic`. You can read the [docs](https://docs.scipy.org/doc/scipy/reference/generated/scipy.stats.binned_statistic.html) for info on the module's functionality; it will essentially compute various useful quantities such as the mean, median, standard deviation, sum and others, within a specified number of bins, or within pre-defined bins.

Let's use this module to compute mean and median curves for the above plot:
```python
from scipy.stats import binned_statistic

# Get the statistics we want in 20 equally spaced bins in log(M_200)
# The module also gives us the edges of the bins
mean, bin_edges, bin_number = binned_statistic(log_M200,mstar_over_mhalo,statistic='mean',bins=20)
median, bin_edges, bin_number = binned_statistic(log_M200,mstar_over_mhalo,statistic='median',bins=20)

# Find the centres of the bins
bin_centres = (bin_edges[:-1]+bin_edges[1:])/2.

fig, ax = plt.subplots(figsize=(8,6))

ax.scatter(log_M200,mstar_over_mhalo,marker='o',edgecolors='k',facecolors='none',s=5)
ax.plot(bin_centres,mean,c='maroon',lw=2,ls='-',label='Mean')     # Plot the mean with a solid line
ax.plot(bin_centres,median,c='skyblue',lw=2,ls='--',label='Median')  # Plot the median with a dashed line

ax.legend(loc='upper right',prop={'size': 12})

ax.set_xlabel(r'$\log(M_{200}/{\rm M}_\odot)$',fontsize=16)
ax.set_ylabel(r'$\log(M_\star/M_{200})$',fontsize=16)
plt.savefig('/path/to/your/directory/smhm_meanmedian.png')

plt.show()
```
This produces the following plot:

![Image](/images/smhm_meanmedian.png)

As you can see, the curves are very similar, but there are some differences in detail. The median becomes a poor representation of the data at high mass, where there are very few data points per bin. At low mass, you'll see that the mean cuts off before the median - this is because there are a few low-mass haloes containing no stars, producing `log(0)=-inf` errors that break the mean. You'll need to use your judgement to decide which of the mean or median is the appropriate statistic. 

Generally, for large datasets, or for datasets with large dynamic ranges, I prefer to use the median as the mean can be very strongly weighted towards larger values. For example, see the plot in the Appendix of [this paper](https://arxiv.org/pdf/1810.07696.pdf) - the mean is at least a factor of 10 higher than the median in the top panel because X-ray luminosity has an enormous dynamic range at fixed M_200.

### Fancy medians

Rather than binning your data to show the median, there's another technique you can use. It's called "locally weighted scatterplot smoothing" (LOWESS, described by Cleveland 1979), and it's a method of obtaining a fitted value for every datapoint, giving a lovely, smooth running median. LOWESS does this by doing a weighted linear regression of a fraction of the data (that you specify) that's closest to each datapoint, giving a fitted value.

There's a python module that will do this for you, and you can read about it [here](https://www.statsmodels.org/stable/generated/statsmodels.nonparametric.smoothers_lowess.lowess.html). You'll need to install it; this is simply a matter of typing `pip3.5 install statsmodels --user` on the command line. Here's how we can use it to get a lovely, smooth median for our data:

```python
# Import the module
import statsmodels.nonparametric.smoothers_lowess as lowess

# First, we need to make sure there are no dodgy NaNs or infs in our data
make_safe = np.where(np.isfinite(mstar_over_mhalo))[0]
log_M200 = log_M200[make_safe]
mstar_over_mhalo = mstar_over_mhalo[make_safe]

# Then we need to sort the data in ascending x value
sort_M200 = np.argsort(log_M200) # argsort gives us the indices that sort the array
log_M200 = log_M200[sort_M200]
mstar_over_mhalo = mstar_over_mhalo[sort_M200]

# Now we can get the running median
running_median = lowess.lowess(mstar_over_mhalo, log_M200, frac=0.2, it=3, delta=0.0, is_sorted=True, missing='none', return_sorted=False)

fig, ax = plt.subplots(figsize=(8,6))

ax.scatter(log_M200,mstar_over_mhalo,marker='o',edgecolors='k',facecolors='none',s=5)
ax.plot(log_M200,running_median,c='skyblue',lw=2)  # Plot the median with a dashed line

ax.set_xlabel(r'$\log(M_{200}/{\rm M}_\odot)$',fontsize=16)
ax.set_ylabel(r'$\log(M_\star/M_{200})$',fontsize=16)
plt.savefig('/path/to/your/directory/smhm_lowess.png')

plt.show()
```
As you can see, there are a couple of extra steps; we must make sure there are no invalid datapoints (NaNs or infs), and sort the data in order of ascending x value. We get the following:

![lowess](/images/smhm_lowess.png)

You'll immediately notice that this method gets the answer rather wrong at high mass. This is because there are very few datapoints here and LOWESS is, in this case, still fitting to the closest 20% of datapoints, skewing the fitting. You can get around this by adjusting the fraction of the data used for fitting.

## Using the catalogues II: Galaxy stellar mass function

The galaxy stellar mass function is of the most important descriptors of a galaxy population, defining how many galaxies of a given stellar mass occupy a given volume of space. It encodes information about the efficiency of galaxy formation and the hierarchical nature of the universe. **It's also one of the key diagnostics on which the EAGLE model was calibrated**, since, in simple terms, it tells you how many galaxies of a certain mass your model needs to produce to be realistic.

Let's have a go at reproducing Fig. 4 of Schaye et al. (2015), which shows the z=0.1 GSMFs of three of the key EAGLE runs. This is a good example of how you can work with multiple simulations in one script:
```python
import numpy as np
import matplotlib.pyplot as plt
from catalogue_reading import *

# Establish the bin edges for our mass function
# 24 equal size bins in log space
bin_edges = np.linspace(7,12,25)

# Get the bin sizes - we'll need these for our normalisation
bin_sizes = bin_edges[1:]-bin_edges[:-1]

# Get the bin centres for plotting
bin_centres = (bin_edges[:-1]+bin_edges[1:])/2.

# Load the stellar masses in 30kpc apertures, convert to solar masses and take the log_10
Ref_100_mstar = np.log10(catalogue_read('Subhalo','ApertureMeasurements/Mass/030kpc',sim='L0100N1504',model='REFERENCE',tag='027_z000p101')[:,4] * 1e10)

# Make a histogram in our pre-defined bins.
# The second output is the bin edges, which we already have, so I use a dummy variable
Ref_100_histogram, _ = np.histogram(Ref_100_mstar,bins=bin_edges)

# Normalise the mass function. We take the number in each bin and divide by the comoving simulation volume times the logarithmic bin size
# This tells us how many galaxies of each mass are in each comoving Mpc^3 of simulation volume
Ref_100_gsmf = np.log10(Ref_100_histogram/(100.**3 * bin_sizes))

# Repeat the process for the two other models in the Schaye+15 plot
Recal_25_mstar = np.log10(catalogue_read('Subhalo','ApertureMeasurements/Mass/030kpc',sim='L0025N0752',model='RECALIBRATED',tag='027_z000p101')[:,4] * 1e10)
Recal_25_histogram, _ = np.histogram(Recal_25_mstar,bins=bin_edges)
Recal_25_gsmf = np.log10(Recal_25_histogram/(25.**3 * bin_sizes))

AGNdT9_50_mstar = np.log10(catalogue_read('Subhalo','ApertureMeasurements/Mass/030kpc',sim='L0050N0752',model='S15_AGNdT9',tag='027_z000p101')[:,4] * 1e10)
AGNdT9_50_histogram, _ = np.histogram(AGNdT9_50_mstar,bins=bin_edges)
AGNdT9_50_gsmf = np.log10(AGNdT9_50_histogram/(50.**3 * bin_sizes))


fig, ax = plt.subplots(figsize=(8,6))

ax.plot(bin_centres,AGNdT9_50_gsmf,lw=2,c='maroon',label='AGNdT9-L0050N0752')
ax.plot(bin_centres,Recal_25_gsmf,lw=2,c='turquoise',label='Recal-L0025N0752')
ax.plot(bin_centres,Ref_100_gsmf,lw=2,c='navy',label='Ref-L0100N1504')

ax.set_ylabel(r'$\log_{10}({\rm d}n/{\rm d}\log_{10} M_*)\,[{\rm cMpc}^{-3}]$',fontsize=16)
ax.set_xlabel(r'$\log_{10}(M_\star/M_\odot)$',fontsize=16)

ax.legend(loc='lower left',prop={'size':12})

plt.savefig('/path/to/your/directory/gsmf.png')
plt.show()
```
This produces the following plot:
![gsmf](/images/gsmf.png)

This produces a good match to the Schaye+15 plot, though not an exact one as the bin sizes differ slightly. As you can see, there are very many low-mass galaxies, very few high-mass galaxies, and a characteristic 'knee' at M_* = 10^10.5-10^11 solar masses. Galaxies in this mass range dominate the present-day mass density of the Universe.

The plot also gives you an idea of the _sampling_ in each simulation volume; in a nutshell, it tells you what's 'available' in each box. As you can see, the bigger the boxes probe to higher masses and rarer objects. For example, the Ref-L0100N1504 volume is the only one to host any galaxies more massive than 10^11.5 solar masses, while the Recal-L0025N0752 volume contains no galaxies above 10^11 solar masses.

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

Of course, this whole process can be wrapped up in a function that incorporates `particle_read`, but does the region selection and aperture masking as well. I won't go into this here, but you can check out my `eagle_tools.read` module [here](https://github.com/j-davies-ari/eagle_tools/blob/master/read.py) for a rather involved (and currently poorly-documented) example. It's essentially a wrapper for `pyread_eagle` in which I treat a simulation snapshot as a python `class` and automate the process of selecting a region and masking a spherical (or cubic) aperture, among other useful methods.

## Calculating a quantity using particles for all haloes in a sample

With the above framework in hand, we can apply it to large samples of galaxies/haloes by introducing a `for` loop. A great feature of `pyread_eagle` is that you need only initialise the `EagleSnapshot` object once, then you can run `select_region` over and over to grab different chunks of the box.

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

## Making histograms of particle properties

Let's take a look at the properties of the gas in haloes, which can broadly be separated into the interstellar medium (ISM) and the circumgalactic medium (CGM). We can explore the distributions of the properties of this gas by making histograms in one or two dimensions.

### 1D histograms

Two of the most important properties of gas particles are their **temperatures** and **densities**. First, lets return to Group 32 in Ref-L0025N0376 and load in these particle properties:

```python
import numpy as np
import matplotlib.pyplot as plt
from catalogue_reading import *
from particle_reading import *

groupnumber = 32
sim = 'L0025N0376'
model = 'REFERENCE'
tag = '028_z000p000'

snapfile = '/hpcdata0/simulations/EAGLE/' + sim + '/' + model + '/data/snapshot_'+tag+'/snap_'+tag+'.0.hdf5'

first_subhalo = catalogue_read('FOF','FirstSubhaloID',sim=sim,model=model,tag=tag)
COP = catalogue_read('Subhalo','CentreOfPotential',sim=sim,model=model,tag=tag)[first_subhalo,:]
centre = COP[groupnumber-1,:]

# Our region size will be the virial radius r_200 for our chosen group
region_size = catalogue_read('FOF','Group_R_Crit200',sim=sim,model=model,tag=tag)[groupnumber-1]

snapshot = pyread_eagle.EagleSnapshot(snapfile)

with h5.File(snapfile,'r') as f:
    h = f['Header'].attrs['HubbleParam']
    a = f['Header'].attrs['ExpansionFactor']
    boxsize = f['Header'].attrs['BoxSize'] * a/h

centre_codeunits = centre * h/a
region_size_codeunits = region_size * h/a

snapshot.select_region(centre_codeunits[0]-region_size_codeunits,centre_codeunits[0]+region_size_codeunits,
                        centre_codeunits[1]-region_size_codeunits,centre_codeunits[1]+region_size_codeunits,
                        centre_codeunits[2]-region_size_codeunits,centre_codeunits[2]+region_size_codeunits)

# Load in coordinates of the gas this time (PartType 0)
coords = particle_read(0,'Coordinates',snapshot,snapfile)

coords -= centre
coords += boxsize/2.
coords %= boxsize
coords -= boxsize/2.

r2 = np.einsum('...j,...j->...',coords,coords)
particle_selection = np.where(r2<region_size**2)[0]

# Load density and temperature.
# I'm going to use particle_read to get the density in cgs units (g/cm^3)
density = particle_read(0,'Density',snapshot,snapfile,cgs_units=True)[particle_selection]
temperature = particle_read(0,'Temperature',snapshot,snapfile)[particle_selection]

# When working with densities, we typically use the hydrogen number density n_H (in cm^-3)
# To get this, we need the hydrogen abundance of the particle (m_H/m_tot)
H_abund = particle_read(0,'SmoothedElementAbundance/Hydrogen',snapshot,snapfile)[particle_selection]

n_H = density * H_abund/1.6737e-24 # multiply by abundance and divide by hydrogen mass
```

We now have the temperature (in K) and hydrogen number density (in cm^-3) for all gas within the virial radius r_200 for this group. To examine the distributions of these quantities, we should take the logarithm and establish some bins, before calling `np.histogram` and using `plt.step` to show the histograms:
```python
n_H = np.log10(n_H)
temperature = np.log10(temperature)

n_H_bins = np.linspace(-8.,2.,51)
T_bins = np.linspace(2.,9.,51)

n_H_bincentres = (n_H_bins[:-1]+n_H_bins[1:])/2.
T_bincentres = (T_bins[:-1]+T_bins[1:])/2.

n_H_hist, _ = np.histogram(n_H,bins=n_H_bins)
T_hist, _ = np.histogram(temperature,bins=T_bins)

fig, (ax1,ax2) = plt.subplots(1,2,figsize=(12,6),sharey=True)
ax1.step(n_H_bincentres,np.log10(n_H_hist),where='mid',c='k')
ax2.step(T_bincentres,np.log10(T_hist),where='mid',c='k')
ax1.set_ylabel(r'$\log_{10}(N)$',fontsize=16)
ax1.set_xlabel(r'$\log_{10}(n_{\rm H})\,[{\rm cm}^{-3}]$',fontsize=16)
ax2.set_xlabel(r'$\log_{10}(T)\,[{\rm K}]$',fontsize=16)
plt.show()
```
Note that we could have achieved the same result by making and plotting the histogram with `plt.hist`, however this method is not very flexible for further analysis, weighting or plotting - I strongly recommend manually binning the data with `np.histogram` and plotting the results. The above code produces this:

![hists](/images/1dhists.png)

These distributions are shown in terms of particle number, however it is typically better to show them in terms of **mass**, since the particles can have different masses depending on their chemical enrichment and other physics. To show this, we can make a _weighted_ histogram, adding up the mass in every bin and dividing through by the total mass. `np.histogram` makes this easy to do with its `weights` and `density` arguments:
```python
mass = particle_read(0,'Mass',snapshot,snapfile)[particle_selection] * 1e10

n_H_hist, _ = np.histogram(n_H,bins=n_H_bins,weights=mass,density=True)
T_hist, _ = np.histogram(temperature,bins=T_bins,weights=mass,density=True)

fig, (ax1,ax2) = plt.subplots(1,2,figsize=(12,6),sharey=True)
ax1.step(n_H_bincentres,np.log10(n_H_hist),where='mid',c='k')
ax2.step(T_bincentres,np.log10(T_hist),where='mid',c='k')
ax1.set_ylabel(r'$\log_{10}(M_{\rm gas}/M_{\rm gas,tot})$',fontsize=16)
ax1.set_xlabel(r'$\log_{10}(n_{\rm H})\,[{\rm cm}^{-3}]$',fontsize=16)
ax2.set_xlabel(r'$\log_{10}(T)\,[{\rm K}]$',fontsize=16)
plt.show()
```
The `density` argument does the normalisation to the total mass. Now we get:

![hists_normed](/images/1dhists_weighted.png)

These distributions are informative as they are, however to really gain a grasp of the particle distribution in _phase space_, it's very useful to be able to see both the densities and temperatures of particles at the same time. There are a variety of ways to do this, though in my opinion the most straightforward and aesthetically pleasing method is to use `plt.hexbin` as follows:
```python
fig, ax = plt.subplots(figsize=(8,6))
hexes = ax.hexbin(n_H,temperature,C=mass,gridsize=50,bins='log',cmap='viridis',mincnt=1,reduce_C_function=np.sum)

cbar = plt.colorbar(hexes)
cbar.ax.set_ylabel(r'$\log_{10}(M_{\rm gas})\,[{\rm M}_\odot]$',fontsize=16)

ax.set_xlabel(r'$\log_{10}(n_{\rm H})\,[{\rm cm}^{-3}]$',fontsize=16)
ax.set_ylabel(r'$\log_{10}(T)\,[{\rm K}]$',fontsize=16)
plt.show()
```
This code will plot a 2D histogram as a 50x50 grid of hexagons, coloured by the log of the total mass in each hexagon:

![phase](/images/phase.png)

Plots like these are brilliant for gaining insight into the state of gas in the simulations. For example, in the bottom right you can see dense, cool particles forming a straight line - these are particles in the ISM on EAGLE's effective equation of state. In the upper left, you can see a big cloud of hot, low-density material - this is the "hot CGM", and the trail of gas connecting this to the ISM represents material cooling onto the disc.

## Making a radial profile

When it comes to understanding the _structure_ of a halo, showing a property as a function of radius in a **radial profile** is the way to go. There are a few different ways to show this information, which I'll cover in this section.

Let's assume that you've loaded the masses, temperatures and radial distances of gas within r_200 of group 32 as in the previous section, and you want to make a **temperature profile**. First, we have a bit of extra housekeeping to do - the temperatures of star-forming gas in the simulation are not physical, so we should remove star-forming gas before making our profiles:
```python
sfr = particle_read(0,'StarFormationRate',snapshot,snapfile)[particle_selection]
no_sf = np.where(sfr==0.)[0]

r2 = r2[particle_selection][no_sf] # Remember this is the square of the radius
mass = mass[no_sf]
temperature = temperature[no_sf]
```
Now, we could simply plot our profile as a function of radius, but it's good practice to plot as a function of something dimensionless like r/r_200. Doing this also means that we could _stack_ profiles of several haloes if we so wished, as they'd all be properly scaled. Time to scale the radial distances:
```python
r200 = catalogue_read('FOF','Group_R_Crit200',sim=sim,model=model,tag=tag)[groupnumber-1]
r_ov_r200 = np.log10(np.sqrt(r2) / r200)
```
There are two primary methods of defining radial bins: bins of fixed width (in either log or linear units), or bins of fixed particle number. Which method you choose will depend on your use-case, as we will see later. Here's how you could make 20 radial bins in both of these ways:
```python
nbins = 20

# Fixed-width bins
bins_fixed_width = np.linspace(-2.,0.,nbins+1)

# Fixed-number bins
r_sorted = np.sort(r_ov_r200)
# Get the indices that split the sorted radii into equal-length chunks
split_indices = np.arange(0,len(r_sorted),step=np.int64(np.floor(len(r_sorted)/nbins)))
# Get the radii corresponding to those indices
bins_fixed_number = r_sorted[split_indices]
```

### Median and mean profiles

If we want to show the median or mean value of a property as a function of radius, `scipy.stats.binned_statistic` can be employed again. The following code will compute these statistics for both fixed-width and fixed-number bins, so that we can compare them:
```python
from scipy.stats import binned_statistic

median_fixedwidth,_,_ = binned_statistic(r_ov_r200,temperature,bins=bins_fixed_width,statistic='median')
mean_fixedwidth,_,_ = binned_statistic(r_ov_r200,temperature,bins=bins_fixed_width,statistic='mean')
median_fixednumber,_,_ = binned_statistic(r_ov_r200,temperature,bins=bins_fixed_number,statistic='median')
mean_fixednumber,_,_ = binned_statistic(r_ov_r200,temperature,bins=bins_fixed_number,statistic='mean')

centres_fixed_width = (bins_fixed_width[:-1]+bins_fixed_width[1:])/2.
centres_fixed_number = (bins_fixed_number[:-1]+bins_fixed_number[1:])/2.

fig, ax = plt.subplots(figsize=(8,6))

ax.plot(centres_fixed_width,np.log10(median_fixedwidth),lw=2,ls='--',c='maroon',label='Median, fixed width')
ax.plot(centres_fixed_width,np.log10(mean_fixedwidth),lw=2,ls='-',c='maroon',label='Mean, fixed width')
ax.plot(centres_fixed_number,np.log10(median_fixednumber),lw=2,ls='--',c='skyblue',label='Median, fixed number')
ax.plot(centres_fixed_number,np.log10(mean_fixednumber),lw=2,ls='-',c='skyblue',label='Mean, fixed number')

ax.legend(loc='best',prop={'size':12})

ax.set_ylabel(r'$\log_{10}(T)\,[{\rm K}]$',fontsize=16)
ax.set_xlabel(r'$\log_{10}(r/r_{200})$',fontsize=16)

plt.show()
```

![profiles1](/images/profiles.png)

Our temperature profiles can look very different depending on the chosen method! First, compare the mean profiles (solid lines) with median profiles (dashed lines). The mean profiles are calculated in linear space, despite being plotted in log units, and so are biased to high temperatures - a classic issue when you have a large dynamic range. The median curves here are likely a far better representation of the distribution as a function of radius.

Now, compare the two binning methods, fixed-width (maroon) with fixed-number (blue). Using a fixed number of particles per bin ensures that each one is _sampled_ equally well - at smaller r/r_200 there are fewer particles and hence only a couple of bins. The fixed-width bins at low r/r_200 appear noisy as there are only a hanful of particles per bin.

### Weighted profiles

It can also be useful to weight a profile by a certain quantity. This can be done by computing the sum of `your_quantity * weight` in each bin and dividing by the sum of the weights in each bin. To get a mass-weighted temperature profile in fixed-number bins:
```python
weighted_temp,_,_ = binned_statistic(r_ov_r200,temperature*mass,bins=bins_fixed_number,statistic='sum')
weights,_,_ = binned_statistic(r_ov_r200,mass,bins=bins_fixed_number,statistic='sum')
weighted_profile = weighted_temp/weights
```
Adding this curve to our previous plot gives:

![profiles2](/images/profiles_extra.png)

The new curve lies right on the standard mean, because most of the particles have similar masses. However, were we to weight the curve by, for example, the X-ray luminosity, the answer would be very different as X-ray emission is dominated by hot (and dense) particles.

### Density profiles in shells

The final method I'll demonstrate concerns only density profiles. One can avoid the problems inherent in choosing the mean or the median by instead computing the total mass within each radial bin, and dividing through by the volume of each "spherical shell" to obtain the average density in each bin.

In this example, I'll show the density profile as a fraction of the mean density of the halo. This time, we'll use fixed-width bins, as we're not computing a statistic but a sum, so sampling isn't an issue.
```python
mass_binned,_,_ = binned_statistic(r_ov_r200,mass,bins=bins_fixed_width,statistic='sum')

# Get the mean gas density within r_200
mean_gas_density = np.sum(mass) / ((4./3.) * np.pi * r200**3)

# Convert our bins in r/r_200 into pure radii and compute the volumes
linear_r_bins = 10.**bins_fixed_width * r200
shell_volumes = (4./3.) * np.pi * (linear_r_bins[1:]**3-linear_r_bins[:-1]**3)

density_profile = (mass_binned/shell_volumes) / mean_gas_density

fig, ax = plt.subplots(figsize=(8,6))
ax.plot(centres_fixed_width,np.log10(density_profile),lw=2,ls='-',c='k')
ax.set_ylabel(r'$\log_{10}(\rho/\bar{\rho})$',fontsize=16)
ax.set_xlabel(r'$\log_{10}(r/r_{200})$',fontsize=16)
plt.show()
```

![shells](/images/density_profile.png)

This is generally the best way to produce density profiles.

## Tracing galaxies through time

As I've mentioned earlier in this guide, galaxies and haloes can be a little tricky to trace forward and backward in time in EAGLE. This is because SUBFIND and FOF are run on each snapshot independently, and so the `GroupNumber` and `SubGroupNumber` of an object are **not** preserved in different snapshots.

The key quantities that _are_ preserved in the simulation, however, are the `ParticleIDs` of particles. By tracing the most tightly-bound particles throughout the simulation, one can build up a _merger tree_ - a sort of "family tree" for a galaxy that links it (the _descendant_) to its _progenitors_. In a merger tree, you have the _main branch_, which contains the greatest integrated mass, and many other branches that merge onto this main branch.

With this merger tree in hand, the user can find the `GroupNumber` and `SubGroupNumber` of a halo's progenitor in any prior snapshot. The trees are available via EAGLE's SQL database, which can be accessed [here](http://icc.dur.ac.uk/Eagle/database.php), and are best described by [Qu et al. (2016)](https://academic.oup.com/mnras/article/464/2/1659/2290988).

I won't go into examples of how to use the database here, as great examples can be found in the release paper by [McAlpine et al. (2016)](https://www.sciencedirect.com/science/article/pii/S2213133716300130?via%3Dihub).


## Making pretty pictures with py-sphviewer

Finally, let's have a go at doing what we all got into simulations for in the first place - making cool pictures of the universe. This is most easily achieved with the python module `py-sphviewer`, by Alejandro Benitez-Llambay, described [here](http://alejandrobll.github.io/py-sphviewer/index.html).

Head back into the folder in which you stored `pyread-eagle` and other software and do:
```
git clone https://github.com/alejandrobll/py-sphviewer.git
cd py-sphviewer
python setup.py install --user
```
The module may be installed with `pip`, however you may want to make your own alterations to the C code so it's good to have your own copy. 

`py-sphviewer` makes it really easy to make images from smoothed particle hydrodynamics (SPH) simulations, which is easier said than done. If you've read up on how SPH works (which you should), you'll know that to simulate a fluid with discrete particles, the properties of SPH particles are smoothed over a kernel with a scale length depending on the distance to an Nth nearest neighbour. The properties of the fluid (such as density, temperature) are then given by convolving the kernels of all particles at any point in space.

To make an image of an SPH simulation, we must first establish a grid of pixels, with a size determined by the resolution of our desired image. We then 'paint' the kernel of each SPH particle onto the grid, summing the values where kernels overlap. Typically, one would smooth the mass of each particle over the grid and divide the resulting image through by the pixel area to show the surface density. As you can imagine, this becomes a very computationally demanding task for large number of particles, and high-resolution images.

The process is straightforward to parallelise, however, and `py-sphviewer` is written in parallel C code, utilising OpenMP. All the above processes are wrapped up in a few python classes that are intuitive to understand.

As an example, let's show the gaseous environment of the group we've been examining in these examples - number 32 in Ref-L0025N0376. We'll make a 5 Mpc wide image, so we'll need to load the masses, smoothing lengths and positions of particles in a region of this size:
```python
import numpy as np
import matplotlib.pyplot as plt
from catalogue_reading import *
from particle_reading import *

import sphviewer

groupnumber = 32
sim = 'L0025N0376'
model = 'REFERENCE'
tag = '028_z000p000'

region_size=2.5

snapfile = '/hpcdata0/simulations/EAGLE/' + sim + '/' + model + '/data/snapshot_'+tag+'/snap_'+tag+'.0.hdf5'

first_subhalo = catalogue_read('FOF','FirstSubhaloID',sim=sim,model=model,tag=tag)
M200 = catalogue_read('FOF','Group_M_Crit200',sim=sim,model=model,tag=tag)[groupnumber-1] * 1e10
COP = catalogue_read('Subhalo','CentreOfPotential',sim=sim,model=model,tag=tag)[first_subhalo,:]
centre = COP[groupnumber-1,:]

snapshot = pyread_eagle.EagleSnapshot(snapfile)

with h5.File(snapfile,'r') as f:
    h = f['Header'].attrs['HubbleParam']
    a = f['Header'].attrs['ExpansionFactor']
    boxsize = f['Header'].attrs['BoxSize'] * a/h

centre_codeunits = centre * h/a
region_size_codeunits = region_size * h/a

snapshot.select_region(centre_codeunits[0]-region_size_codeunits,centre_codeunits[0]+region_size_codeunits,
                        centre_codeunits[1]-region_size_codeunits,centre_codeunits[1]+region_size_codeunits,
                        centre_codeunits[2]-region_size_codeunits,centre_codeunits[2]+region_size_codeunits)

coords = particle_read(0,'Coordinates',snapshot,snapfile)

# Centre and wrap the co-ordinates
coords -= centre
coords += boxsize/2.
coords %= boxsize
coords -= boxsize/2.

# Get masses and smoothing lengths
mass = particle_read(0,'Mass',snapshot,snapfile)
hsml = particle_read(0,'SmoothingLength',snapshot,snapfile)
```
I call the smoothing lengths `hsml` as they are typically denoted as such in codes, and by h in equations. We use these values to initialise a `Particles` instance in the module, and pass this into a `Scene` instance, which sets things up for rendering:
```python
Particles = sphviewer.Particles(coords,mass,hsml=hsml)
Scene = sphviewer.Scene(Particles)
```
`Scene` is clever, as once it's been initialised we can adjust the "camera" that produces the image. The camera can be set at a physical distance from your particles and produce images in angular units for mock observations, or be set at infinity (more useful for theoretical work). The aperture of the image can be changed, as can the position the camera points at, the angle (theta and phi) from which the camera is pointing and the resolution of the image. Manipulation of this camera can be used to make rotating movies, or 'fly-throughs', for example. For now, we'll set the camera at infinity, set the aperture to 5 Mpc by establishing the `extent` as `[-2.5,2.5,-2.5,2.5]` and set the resolution to 1024x1024.
```python
Scene.update_camera(r='infinity', t=0., p=0., extent=[-1.*region_size, region_size, -1.*region_size, region_size], xsize=1024, ysize=1024)
```
For more details on this, see [this iPython notebook](https://github.com/alejandrobll/py-sphviewer/blob/master/wiki/tutorial_sphviewer.ipynb).

Now, `py-sphviewer` does the hard work when we invoke the `Render` class:
```python
Render = sphviewer.Render(Scene)
img = Render.get_image()
extent = Render.get_extent()
```

`img` is a numpy array of the surface density in every pixel, while `extent` is the same as what we passed into the `Camera` and is used for plotting. We can now plot the image using `plt.imshow`:
```python
fig = plt.figure()
ax = fig.add_subplot(111)
im = plt.imshow(np.log10(img), extent=extent, origin='lower', cmap='afmhot')
col = plt.colorbar()
ax.set_xlabel(r'$x\,[{\rm Mpc}]$',fontsize=16)
ax.set_ylabel(r'$x\,[{\rm Mpc}]$',fontsize=16)
col.ax.set_ylabel(r'$\Sigma_{\rm gas}\,[{\rm M}_\odot\,{\rm Mpc}^{-3}]$',fontsize=16)
plt.show()
```

![sph](/images/sph_image.png)

Have a play with the camera and try making images of the different particle types. Note that when making images of the dark matter, you need to account for the fact that `PartType 1` does not have a smoothing length - you can make `py-sphviewer` compute one for you (see the docs).

Finally, a caveat: `py-sphviewer` does not use the Wendland C2 kernel that is actually used for smoothing in EAGLE, and is therefore inconsistent with the SPH implementation of the simulation. You can fix this yourself fairly easily in the code by adding your own kernel, if you know a bit of C. There are also a few issues with small-scale smoothing (check out the [pull requests on GitHub](https://github.com/alejandrobll/py-sphviewer/pull/19)) that I don't think have been resolved as yet, so I recommend only using the module for visualisation purposes and not for detailed scientific studies. 



