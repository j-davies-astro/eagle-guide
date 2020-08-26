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

[Back to top](https://j-davies-ari.github.io/eagle-guide/examples_smhm.md)
