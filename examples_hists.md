# Examples of common EAGLE tasks

[Using the catalogues I: Stellar mass - halo mass relation](https://j-davies-ari.github.io/eagle-guide/examples_smhm.md)

[Showing running means, medians and "fancy medians"](https://j-davies-ari.github.io/eagle-guide/examples_stats.md)

[Using the catalogues II: Galaxy stellar mass function](https://j-davies-ari.github.io/eagle-guide/examples_gsmf.md)

[Loading particles within a spherical aperture around a galaxy](https://j-davies-ari.github.io/eagle-guide/examples_aperture.md)

[Calculating a quantity using particles for all haloes in a sample](https://j-davies-ari.github.io/eagle-guide/examples_sample.md)

[Making histograms of particle properties](https://j-davies-ari.github.io/eagle-guide/examples_hists.md)

[Making radial profiles](https://j-davies-ari.github.io/eagle-guide/examples_profile.md)

[Tracing galaxies through time](https://j-davies-ari.github.io/eagle-guide/examples_tracing.md)

[Making pretty pictures with py-sphviewer](https://j-davies-ari.github.io/eagle-guide/examples_sphviewer.md)

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

[Back to top](https://j-davies-ari.github.io/eagle-guide/examples_hists.md)
