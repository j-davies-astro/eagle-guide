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

[Back to top](https://j-davies-ari.github.io/eagle-guide/examples_sphviewer.md)
