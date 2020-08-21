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

## Loading particles within a spherical aperture around a galaxy

## Calculating a quantity using particles for all haloes in a sample

## Making a radial profile

## Making 1D and 2D histograms of particle properties

## Making pretty pictures with py-sphviewer

## Tracing galaxies through time
