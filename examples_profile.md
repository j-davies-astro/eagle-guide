# Examples of common EAGLE tasks

[Using the catalogues I: Stellar mass - halo mass relation](https://j-davies-ari.github.io/eagle-guide/examples_smhm)

[Showing running means, medians and "fancy medians"](https://j-davies-ari.github.io/eagle-guide/examples_stats)

[Using the catalogues II: Galaxy stellar mass function](https://j-davies-ari.github.io/eagle-guide/examples_gsmf)

[Loading particles within a spherical aperture around a galaxy](https://j-davies-ari.github.io/eagle-guide/examples_aperture)

[Calculating a quantity using particles for all haloes in a sample](https://j-davies-ari.github.io/eagle-guide/examples_sample)

[Making histograms of particle properties](https://j-davies-ari.github.io/eagle-guide/examples_hists)

[Making radial profiles](https://j-davies-ari.github.io/eagle-guide/examples_profile)

[Tracing galaxies through time](https://j-davies-ari.github.io/eagle-guide/examples_tracing)

[Making pretty pictures with py-sphviewer](https://j-davies-ari.github.io/eagle-guide/examples_sphviewer)

## Making radial profiles

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

[Back to top](https://j-davies-ari.github.io/eagle-guide/examples_profile)
