# Examples of common EAGLE tasks

To help you get to grips with all the details I've covered on previous pages, I'll now show you several examples of how to do some standard "simulation tasks". These should hopefully cover many of the types of analysis you'll be doing when working with EAGLE.

## Using the catalogues I: Stellar mass - halo mass relation

Here's a very simple plot to make using only the galaxy catalogues. The stellar mass-halo mass relation is very important as it describes how efficient galaxy formation is in haloes of a given mass. We can make this plot by creating a sample of haloes from the `FOF` table and getting their central galaxy stellar masses from the `Subhalo` table, like so:

```python
import numpy as np
import matplotlib.pyplot as plt
from catalogue_reading import *

# Let's look at the largest simulation now
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
![smhm_basic](/images/smhm_basic.png)




## Computing running means, medians and "fancy medians"

## Using the catalogues II: Galaxy stellar mass function

## Loading particles within a spherical aperture around a galaxy

## Calculating a quantity using particles for all haloes in a sample

## Making a radial profile

## Making pretty pictures with py-sphviewer

## Tracing galaxies through time
