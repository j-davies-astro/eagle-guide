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

## Tracing galaxies through time

As I've mentioned earlier in this guide, galaxies and haloes can be a little tricky to trace forward and backward in time in EAGLE. This is because SUBFIND and FOF are run on each snapshot independently, and so the `GroupNumber` and `SubGroupNumber` of an object are **not** preserved in different snapshots.

The key quantities that _are_ preserved in the simulation, however, are the `ParticleIDs` of particles. By tracing the most tightly-bound particles throughout the simulation, one can build up a _merger tree_ - a sort of "family tree" for a galaxy that links it (the _descendant_) to its _progenitors_. In a merger tree, you have the _main branch_, which contains the greatest integrated mass, and many other branches that merge onto this main branch.

With this merger tree in hand, the user can find the `GroupNumber` and `SubGroupNumber` of a halo's progenitor in any prior snapshot. The trees are available via EAGLE's SQL database, which can be accessed [here](http://icc.dur.ac.uk/Eagle/database.php), and are best described by [Qu et al. (2016)](https://academic.oup.com/mnras/article/464/2/1659/2290988).

I won't go into examples of how to use the database here, as great examples can be found in the release paper by [McAlpine et al. (2016)](https://www.sciencedirect.com/science/article/pii/S2213133716300130?via%3Dihub).

[Back to top](https://j-davies-ari.github.io/eagle-guide/examples_tracing.md)
