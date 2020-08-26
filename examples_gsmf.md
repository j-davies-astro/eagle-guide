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

[Back to top](https://j-davies-ari.github.io/eagle-guide/examples_gsmf.md)
