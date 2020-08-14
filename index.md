## Introduction

With this brief guide, I hope to get you up and running with the EAGLE simulation data at the Astrophysics Research Institute. I'll explain how to install useful python modules for reading simulation data, how to use them to extract information about dark matter haloes, galaxies and simulated particles, and give examples of some basic data reduction and analysis you'll likely need to do when working with EAGLE. This guide is aimed at new Masters and PhD students - I'll try my best to explain everything in plain English and with as little jargon as possible!

### Prerequisites

- I'll assume that you've read the original EAGLE papers: [Schaye et al. (2015)](https://academic.oup.com/mnras/article/446/1/521/1316115) and [Crain et al. (2015)](https://academic.oup.com/mnras/article/450/2/1937/984366). They introduce the simulation nomenclature and both explain and justify the physics in the EAGLE model.

- We will be working in the Ubuntu (Linux) operating system, which runs on all ARI _starpc_ machines. If you're working remotely or using a personal computer, you can access the ARI systems via the machines _external1_ and _external2_. This can be done two different ways:

  - **Command line:** Type `ssh <your_username>@external2.astro.ljmu.ac.uk>` and type in your password when prompted. Unfortunately, X11 forwarding is blocked, so you won't be able to use any graphical interfaces through this method. A better method is to use...
  - **NoMachine:** You can access a full Linux environment remotely using NoMachine. Please see [the ARI Docs pages](https://www.astro.ljmu.ac.uk/docs/) for information on how to set this up.

- All code will be run from a Linux terminal/command line. By default, the ARI systems use `csh`. I recommend you get familiar with some basic Linux commands if you aren't already - commands like `cd`, `ls`, `top`, `grep`, `more` and `tail` will be invaluable to you.

- This guide is for working with EAGLE in Python 3, and I'll assume that you're already proficient with Python. We will primarily make use of the standard packages `numpy`, `matplotlib`, and `h5py`. The Python environment and these packages should already be set up for you on the ARI systems. Other very useful packages include:
  
  - Other standard python modules such as `sys`, `os`, and `copy`
  - `astropy` for doing, among other things, cosmological calculations
  - `scipy` for statistics
  - `tqdm` for showing progress bars while your code runs


### Running your code

You should **not** run your code on either a _starpc_ or _external1/2_. When it comes to analysing cosmological simulations, machines like these are simply not up to the task! The EAGLE simulations follow the evolution of billions of particles and consume enormous amounts of memory when loaded in from disk - for example, simply loading the co-ordinates of all gas particles in the Ref-L100N1504 EAGLE simulation will use ~80 GB of RAM.

To work with EAGLE, first get yourself on _starpc_ or _external1/2_, open a terminal window. There are then two ways you can run code:

- The more straightforward way is to directly run code on one of the ARI's many cluster computers. These are listed [here](https://www.astro.ljmu.ac.uk/docs/knowledge-base/slurm-partitions/) - those working on HPC projects should primarily use one of `cyclops`, `phoenix`, the "four horsemen" (`war`, `famine`, `pestilence` and `death`) or `mclovin`. In a terminal on _starpc_ or _external1/2_, type `ssh -Y cyclops` or similar, type in your password, and you'll be ready to run code on that machine.

- The alternative method is to use the SLURM resource manager, which will submit your code to a machine with computing resources available. For information on this, see the ARI docs [here](https://www.astro.ljmu.ac.uk/docs/knowledge-base/slurm-introduction-2/).


## Installing software

Let's install two key bits of software for working with EAGLE.

### Python read routines

First, we'll want to install the [pyread_eagle](https://github.com/kyleaoman/pyread_eagle) python package. To do this, you'll want to create a folder somewhere for storing software/modules. Let's create one in our home directory and enter it:
```
cd ~
mkdir software
cd software
```
Now, clone the `pyread_eagle` git repository here by pasting the following into your terminal:
```
git clone https://github.com/kyleaoman/pyread_eagle.git
```
You should now have a directory named `pyread_eagle` - this is your copy of the git repository. Now we can enter it and install the module:
```
cd pyread_eagle
pip3.5 install --user -e .
```
to install the package into your `$PYTHONPATH` via symlink (this means that when you update your copy of the repository, the installed version of the module also updates). Check that the module installed by importing it into python:
```
python3.6
import pyread_eagle
```

This module is Kyle Oman's pure-python port of `read_eagle`, by John Helly. The original module is written in C, with a wrapper for use in Python. `pyread_eagle` is, in Kyle's words, between "roughly 2x faster and 5x slower than the C version" depending on what you're loading in. The disadvantage with `read_eagle`, however, is that it's very difficult to install on the ARI systems; you need to have HDF5 configured and installed in a specific way and your installations of Python, HDF5 and the module must have been compiled with the same compilers. You can find `read_eagle` [here](https://github.com/jchelly/read_eagle), but I don't recommend trying it at the ARI - I made it work at the start of my PhD but have been unable to replicate that success for others since!

### GUI for examining HDF5 files

EAGLE simulation data is stored in the [HDF5](https://www.hdfgroup.org/solutions/hdf5/) file format, which allows for fast I/O and storage of very large datasets. These files act more like Python dictionaries than simple text files, and so it's a little more difficult to have a direct look at what's actually _in_ the files. 

To make this easier, download the HDFView software, which can be found [here](https://www.hdfgroup.org/downloads/hdfview/#download), onto your space on the /home drive. Inside this package should be a file named `hdfview.sh`. Run this to open the application.

### Creating aliases (command-line shortcuts)

It's a bit cumbersome to dig into that HDFView folder every time you want to run HDFView. To speed up tasks on the command line, you can create 'aliases' for much longer commands. To do this, you need to add lines to your `.cshrc` file, which is a little script which runs every time you open a new Linux terminal.

Open up your `.cshrc` file in the `gedit` text editor by typing `gedit ~/.cshrc &`. The `&` here allows you to open the editor, but still use your terminal. You can add any terminal commands you'd like to run on startup here. For example, to speed up running python code I like to add this alias:
```
alias py "python3.6"
```
Now I can simply type `py` rather than `python3.6` to run python code. This is a very lazy example - aliases are mainly useful for shortening much longer commands, such as:
```
alias hdfview "/home/<my-username>/HDFView-3.1.0-Linux/HDFView/3.1.0/hdfview.sh"
```
This allows me to run HDFView by simply typing `hdfview`, rather than by typing out the whole directory structure to get to the script. By now, I'm sure you've worked out the syntax - you simply type `alias <shortcut> "<full command>"`.

Once you're done entering aliases, save your `.cshrc` file, type `source ~/.cshrc` into your terminal to rerun the script, and your new aliases should work.


## Getting to grips with the EAGLE data

In this section we'll start exploring the simulation data. I'll explain the format the data is stored in, go over the units system and introduce you to the galaxy catalogues.

### The directory structure

At the ARI, we have many different simulations run with the EAGLE model saved to disk - they're stored on disk in a systematic fashion based on the simulation volume, resolution and model variation. We'll start exploring the data by opening up HDFView and clicking the 'Open' button at the top-left. Then: 

Navigate to the EAGLE simulations, which can be found at `/hpcdata0/simulations/EAGLE/`.

Here, among a few other folders, you'll find some folders with names `LxxxxNxxxx`. These identifiers specify the co-moving simulation 'box size' (e.g. L0100 is the flagship volume with 100 cMpc on a side), and the number of particles in the 'box' which is defined by the number along one edge (e.g. the flagship volume, N1504, contains 1504^3 dark matter particles and, initially, 1504^3 gas particles). These numbers are set by the **initial conditions** of the simulation. For simulations of the same resolution, these numbers scale as you would expect:

- `L0100N1504`, `L0050N0752`, `L0025N0376` and `L0012N0188` all have the same, standard, EAGLE resolution
- `L0025N0752` and `L0034N1034` are examples of high-resolution simulations
- **N.B.** Since EAGLE uses smoothed-particle hydrodynamics (SPH), which is a Lagrangian scheme, the resolution of the simulation is in fact determined by the _masses_ of the simulated particles. See Schaye et al. (2015) for more info on this.

Head into the `L0025N0376` directory. Here you'll see a whole bunch of different folders; their names describe the **model variation** used in that simulation. The standard EAGLE model is called `REFERENCE`, and we'll focus on that for now. 

Head into that `REFERENCE`, and then into `data`. Here you'll see three types of folder, corresponding to the three types of output that the simulation produces. 

- Folders starting with `snapshot_` contain the raw simulation output. These contain information on the state of every gas, dark matter, star and black hole particle in the simulation.
  
- Folders starting with `groups_` contain information on the structures in the simulation that are identified as 'bound', according to the SUBFIND structure finder. I'll sometimes refer to these as "the catalogues" - more about this later.
  
- Folders starting with `particledata_` contain similar 'raw' data to the `snapshot_` folders, but only for particles that are identified as bound to those structures mentioned above. These files contain additional info that only means anything in relation to the structure the particle is bound to.
 
The remainder of the folder names specify which **snapshot** output is in the folder. The simulation is 'dumped' to file 29 times over the course of its evolution, starting at redshift _z=20_ and ending at _z=0_. The first number specifies the snapshot number, with `000` being the first output and `028` being the last. The somewhat cryptic remainder of the folder name specifies the redshift of the snapshot - for example, `_z001p004` means _z=1.004_.

Some simulations will also show "snipshot" outputs here. The simulations only dump 29 full outputs because of the immense amount of storage space required, however reduced outputs, called snipshots, can also be dumped out roughly 400 times. These contain only the 'bare essentials' such as the positions, velocities and densities of particles.

Let's take a look at the full output at _z=1_ and open up the `snapshot_019_z001p004` folder. Note that this isn't _exactly_ redshift 1 - the outputs are generated when it's 'convenient' for the simulation to do so. For all intents and purposes, it's _z=1_.

In here, there are several `.hdf5` files with essentially the same name, just with a different number at the end. These are simulation `chunks`, which split the box up, and if we wanted to work with all the particles in Python, we'd need to load them all. The simulation is split into chunks, however, so that you _don't_ need to read in the whole simulation if you're only interested in a small spatial region. This is usually the case - most of the time you'll only be interested in a single galaxy or dark matter halo. I'll discuss this more in a later section. Thankfully, `pyread_eagle` takes care of this for us, so you don't need to worry.
  
### Simulation snapshots

Let's take a look at what's in a snapshot - open any of the `.hdf5` files that we just got to. You'll now see what looks like a file browser on the left, which we can use to navigate the contents of the file. HDF5 files are made up of _datasets_, which can be organised into _groups_. The most important part of the snapshot files are the groups entitled `PartType[0-5]`, which contain the particle data, and the `Header` and `Units` datasets.

Clicking the arrows next to any of the `PartType[0-5]` groups will reveal a drop-down menu of all the different datasets available for that particle type. Several datasets are very general and apply to all particles in the simulation, such as the particle positions (`Co-ordinates`), velocities (`Velocity`) and unique identifiers (`ParticleIDs`), which are unchanging throughout the simulation and can be used to trace particles between snapshots. Each particle type then has its own set of properties (datasets):

- `PartType0` are gas particles. There are initially as many gas particles as dark matter particles, however they reduce in number over time as they are converted into stars. A myriad of gas properties are tracked in the simulation, such as density, temperature, internal energy, chemical enrichment and star formation activity.

- `PartType1` are dark matter (DM) particles. By contrast to gas particles, DM particles have very few properties, as they interact through gravity only. The number of DM particles is unchanging, and they all have identical masses that do not change - for this reason, their masses are not stored (it would be a waste of disk space!). The DM particle mass can be found in the Header (more on this later).

- `PartType2/3` are not used in the EAGLE simulation suite. They are a component of the GADGET simulation code which represent 'boundary particles', which are used for performing cosmological 'zoom' simulations. The simulations we're interested in are 'periodic' (more on this later), so these particles aren't needed.

- `PartType4` are star particles. Gas particles can stochastically convert to star particles when their star formation rates are non-zero, and they do this at a rate set by the empirical Kennicutt-Schmidt relation - for an explanation of this, see the EAGLE release papers and [Schaye & Dalla Vecchia (2008)](https://academic.oup.com/mnras/article/383/3/1210/1037943). They inject feedback energy associated with their formation into the surrounding gas, after a short delay. Many useful properties, such as metallicity, initial mass and formation time are stored for star particles.

- `PartType5` are black holes, which form at the centres of dark matter haloes when they reach a certain mass. Black hole particles can accrete mass from their neighbouring gas particles and grow, injecting AGN feedback energy into their surroundings. Several quantities related to this accretion and feedback process are tracked for black hole particles.

Double-clicking on any dataset will show you an Excel-like view of the data in that dataset. This isn't too informative when looking at raw particle data, but when you create your own hdf5 files, the ability to examine your results for a quick 'sanity check' is invaluable.

When you've clicked on a dataset in HDFView, you should be able to see a list of _attributes_ in the main window. For most particle properties, these should be `CGSConversionFactor`, `VarDescription`, `aexp-scale-exponent` and `h-scale-exponent`. These attributes are essentially metadata for the selected dataset - as I will explain in the next section, they are invaluable!

The `Header` in the snapshot file contains several useful attributes that describe the current snapshot, such as the current redshift, time, expansion factor, and other useful quantities such as the density parameters in baryons, matter and vacuum energy. We'll be using these later!

For further information on the datasets in the particle data, take a look at the [document that accompanied the public release of the EAGLE particle data](https://arxiv.org/pdf/1706.09899.pdf).


### Unit system

The quantities in the simulation snapshots are stored in what could be termed _"comoving h-full GADGET units"_. Let's break down what this means:

- _"comoving"_: The simulation is run entirely in comoving units. That way, we have a box of fixed size, with galaxy formation happening in it. In "reality" (physical units), however, the box is expanding like a balloon. To get our results in physical units, we must therefore multiply our data by the `ExpansionFactor` (found in the `Header`), raised to some power that depends on the quantity (`aexp-scale-exponent`, found in the quantity's attributes). For example, we would multiply distances by `a`, densities by `a**-3` and masses by `a**0=1`.

- _"h-full"_: It is commonplace in astronomy to sweep our ignorance of the true value of Hubble's constant under the rug and present values in terms of "little h", the dimensionless Hubble parameter, defined as `H_0 = 100 h km s^-1 Mpc^-1`. For more info, see [this paper](https://arxiv.org/pdf/1308.4150.pdf) by Darren Croton. We must also factor this out when we want results in physical units, by multiplying by h (`HubbleParam` in the `Header`) again raised to some power (`h-scale-exponent`).

- _"GADGET units"_: As we know, galaxy formation and cosmology tends to involve very extreme numbers; large distances, high energies and temperatures and low densities are all commonplace. To keep the numbers stored in the snapshots small and easy to deal with, the GADGET simulation code employs some slightly unusual units that you'll need to get the hang of. The most important ones to remember are:

  - Masses are expressed in units of 10^10 solar masses
  - Distances are in megaparsecs (Mpc)
  - Velocities are in km s^-1
  
  These are usually very easy to deal with; for example, you'll typically want mass in solar masses, so you can simply multiply what you load in by 10^10. Other times, the conversion is less straightforward - `Density` is in "10^10 solar masses per cubic megaparsec", which can be a bit awkward to convert into g cm^-3. Thankfully, as I mentioned earlier, every quantity has the attribute `CGSConversionFactor` - multiply your data by this to get it in cgs (centimetres, grams and seconds) units. Astronomy is a bit stuck in the past and hasn't quite discovered the SI system yet, but the benefit of using these conversion factors is that you immediately know what units you're in. The attributes of `Units` in the snapshot files give the GADGET units in cgs, which can also be helpful.

I'll give examples of how to do these conversions when we start experimenting with loading data in Python. Once you're familiar with it, it's quite straightforward to automate the unit conversion process, and you'll never need to worry about it.


### Group catalogues



## Working with the catalogues

### Loading in catalogues 

### Unit conversions

### Constructing samples of galaxies/subhaloes


## Working with particle data

### Initialising pyread_eagle

### Selecting a region and loading particle quantities

### Particle positions and periodic boxes






## Examples of common EAGLE tasks

### Using the catalogues I: Stellar mass - halo mass relation

### Using the catalogues II: Galaxy stellar mass function

### Loading particles within a spherical aperture

### Calculating a quantity for all haloes in a sample

### Making a radial profile

### Making pretty pictures with py-sphviewer

### Tracing galaxies through time





