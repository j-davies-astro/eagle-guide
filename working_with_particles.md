# Working with particle data

Now let's take a look at working with the simulated particles themselves. Rather than using `h5py` as we did for the catalogues, we're now going to use the `pyread_eagle` module that we installed earlier. This will allow us to read small portions of the simulation, rather than the whole lot, speeding things up enormously for most use-cases.

The documentation on the module's [GitHub page](https://github.com/kyleaoman/pyread_eagle) does a good job of explaining the various functions available and their arguments, so I won't explain these in great detail here. I'll instead focus on giving examples of how to use the module, and you can refer to those docs for any extra information.

## How it works

So how does `pyread_eagle` load in only _part_ of a simulation snapshot? More specifically, how does it know which particles lie in the small region you've asked it to load, without first loading all the particles to check?

This is all made possible by the fact that the simulation data is stored on disk in a very specific way. To quote from the EAGLE documentation:

_Particles of each type are distributed across the different files of a snapshot in such a way that it is easy to retrieve those that are in a simply connected region - for example all particles within a given distance from a  given  location, say the centre of mass of a halo. This is done by dividing the computational volume into cubic cells, 2^6 cells on a side, calculating which 3D cell each particle is in (referred to as the hash key), and sorting particles based on this hash key. Cells are then distributed over the individual files that make up a single snapshot. The hash tables allow one to determine which files need reading to retrieve all particles in a spherical region around a given centre. Because particles are arranged in cubic cells, using the hash tables returns particles in cells - limiting the list to particles in a spherical region is left to the user.  Hash tables are constructed separately for each particle type._


## Initialising pyread_eagle

## Selecting a region and loading particle quantities

## Particle positions and periodic boxes

## Loading particles within a spherical aperture
