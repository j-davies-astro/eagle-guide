# Examples of common EAGLE tasks

To help you get to grips with all the details I've covered on previous pages, I'll now show you several examples of how to do some standard "simulation tasks". These should hopefully cover many of the types of analysis you'll be doing when working with EAGLE.

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
region_size = region_size * h/a

# Run the select_region method
snapshot.select_region(centre_codeunits[0]-region_size,centre_codeunits[0]+region_size,
                        centre_codeunits[1]-region_size,centre_codeunits[1]+region_size,
                        centre_codeunits[2]-region_size,centre_codeunits[2]+region_size)

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
![stars_masked](/images/star_masked.png)

Perfect - we now have only the particles within our spherical aperture, and (if you squint a little) it does look like a spiral galaxy. Note that the shape looks different to before, because the aspect ratio of the plot has changed.

This all seems rather laborious for the sake of just loading a few particles in. The beauty of it, however, is that now our `EagleSnapshot` instance is initialised and we have our `particle_selection` mask for cutting the data down to a spherical aperture, working with other particle properties within this aperture is very simple. For example, if we wanted to know the total stellar mass within this aperture, this can now be done in one line:
```python
Mstar_100kpc = np.sum(particle_read(4,'Mass',snapshot,snapfile)[particle_selection]) * 1e10
print(Mstar_100kpc)
```
You'll see that this is a little more than the 30 pkpc stellar mass we printed earlier, due to the wider aperture.

## Calculating a quantity using particles for all haloes in a sample



## Making a radial profile

## Making 1D and 2D histograms of particle properties

## Making pretty pictures with py-sphviewer

## Tracing galaxies through time
