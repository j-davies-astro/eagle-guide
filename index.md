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

- Navigate to the EAGLE simulations, which can be found at `/hpcdata0/simulations/EAGLE/`

- Here, among a few other folders, you'll find some folders with names `LxxxxNxxxx`. These identifiers specify the co-moving simulation 'box size' (e.g. L0100 is the flagship volume with 100 cMpc on a side), and the number of particles in the 'box' which is defined by the number along one edge (e.g. the flagship volume, N1504, contains 1504^3 dark matter particles and, initially, 1504^3 gas particles). These numbers are set by the **initial conditions** of the simulation. For simulations of the same resolution, these numbers scale as you would expect:

  - `L0100N1504`, `L0050N0752`, `L0025N0376` and `L0012N0188` all have the same, standard, EAGLE resolution
  - `L0025N0752` and `L0034N1034` are examples of high-resolution simulations
  - **N.B.** Since EAGLE uses smoothed-particle hydrodynamics (SPH), which is a Lagrangian scheme, the resolution of the simulation is in fact determined by the _masses_ of the simulated particles. See Schaye et al. (2015) for more info on this.

- Head into the `L0025N0376` directory. Here you'll see a whole bunch of different folders; their names describe the **model variation** used in that simulation. The standard EAGLE model is called `REFERENCE`, and we'll focus on that for now. 

- Head into that `REFERENCE`, and then into `data`. Here you'll see three types of folder, corresponding to the three types of output that the simulation produces. 

  - Folders starting with `snapshot_` contain the raw simulation output. These contain information on the state of every gas, dark matter, star and black hole particle in the simulation.
  
  - Folders starting with `groups_` contain information on the structures in the simulation that are identified as 'bound', according to the SUBFIND structure finder. I'll sometimes refer to these as "the catalogues" - more about this later.
  
  - Folders starting with `particledata_` contain similar 'raw' data to the `snapshot_` folders, but only for particles that are identified as bound to those structures mentioned above. These files contain additional info that only means anything in relation to the structure the particle is bound to.
 
  The remainder of the folder names specify which **snapshot** output is in the folder. The simulation is 'dumped' to file 29 times over the course of its evolution, starting at redshift _z=20_ and ending at _z=0_. The first number specifies the snapshot number, with `000` being the first output and `028` being the last. The somewhat cryptic remainder of the folder name specifies the redshift of the snapshot - for example, `_z001p004` means _z=1.004_.

  Some simulations will also show "snipshot" outputs here. The simulations only dump 29 full outputs because of the immense amount of storage space required, however reduced outputs, called snipshots, can also be dumped out roughly 400 times. These contain only the 'bare essentials' such as the positions, velocities and densities of particles.






