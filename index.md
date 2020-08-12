## Introduction

With this brief guide, I hope to get you up and running with the EAGLE simulation data at the Astrophysics Research Institute. I'll explain how to install useful python modules for reading simulation data, how to use them to extract information about dark matter haloes, galaxies and simulated particles, and give examples of some basic data reduction and analysis you'll likely need to do when working with EAGLE. 

### Prerequisites

- I'll assume that you've read the original EAGLE papers: [Schaye et al. (2015)](https://academic.oup.com/mnras/article/446/1/521/1316115) and [Crain et al. (2015)](https://academic.oup.com/mnras/article/450/2/1937/984366). They introduce the simulation nomenclature and both explain and justify the physics in the EAGLE model.

- We will be working in the Ubuntu (Linux) operating system, which runs on all ARI _starpc_ machines. If you're working remotely or using a personal computer, you can access the ARI systems via the machines _external1_ and _external2_. This can be done two different ways:

  - **Command line:** Type `ssh <your_username>@external2.astro.ljmu.ac.uk>` and type in your password when prompted. Unfortunately, X11 forwarding is blocked, so you won't be able to use any graphical interfaces through this method. A better method is to use...
  - **NoMachine:** You can access a full Linux environment remotely using NoMachine. Please see [the ARI Docs pages](https://www.astro.ljmu.ac.uk/docs/) for information on how to set this up.
  
  
- All code will be run from a Linux terminal/command line. By default, the ARI systems use `csh`. I recommend you get familiar with some basic Linux commands if you aren't already - commands like `cd`, `ls`, `top`, `grep`, `more` and `tail` will be invaluable to you.

- This guide is for working with EAGLE in Python 3, and will primarily make use of the standard packages `numpy`, `matplotlib`, and `h5py`. The Python environment and these packages should already be set up for you on the ARI systems. Other very useful packages include:
  
  - Other standard python modules such as `sys`, `os`, and `copy`
  - `astropy` for doing, among other things, cosmological calculations
  - `tqdm` for showing progress bars while your code runs


### Running your code

You should **not** run your code on either a _starpc_ or _external1/2_. When it comes to analysing cosmological simulations, machines like these are simply not up to the task! The EAGLE simulations follow the evolution of billions of particles and consume enormous amounts of memory when loaded in from disk - for example, simply loading the co-ordinates of all gas particles in the Ref-L100N1504 EAGLE simulation will use ~80 GB of RAM.

To work with EAGLE, first get yourself on _starpc_ or _external1/2_, open a terminal window. There are then two ways you can run code:

- The more straightforward way is to directly run code on one of the ARI's many cluster computers. These are listed [here](https://www.astro.ljmu.ac.uk/docs/knowledge-base/slurm-partitions/) - those working on HPC projects should primarily use one of `cyclops`, `phoenix`, the "four horsemen" (`war`, `famine`, `pestilence` and `death`) or `mclovin`. In a terminal on _starpc_ or _external1/2_, type `ssh -Y cyclops` or similar, type in your password, and you'll be ready to run code on that machine.

- The alternative method is to use the SLURM resource manager, which will submit your code to a machine with computing resources available. For information on this, see the ARI docs [here](https://www.astro.ljmu.ac.uk/docs/knowledge-base/slurm-introduction-2/).


## Installing software

Let's install two key bits of software for working with EAGLE. The first is essential, and the second is strongly recommended.

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
You should now have a directory named `pyread_eagle`. `cd` into it and type
```
pip install --user -e .
```
to install the package into your `$PYTHONPATH` via symlink (this means that when you update your copy of the repository, the installed version of the module also updates). Check that the module installed by importing it into python:
```
python3.6
import pyread_eagle
```

This module is Kyle Oman's pure-python port of `read_eagle`, by John Helly. The original module is written in C, with a wrapper for use in Python. `pyread_eagle` is, in Kyle's words, between "roughly 2x faster and 5x slower than the C version" depending on what you're loading in. The disadvantage with `read_eagle`, however, is that it's very difficult to install on the ARI systems; you need to have HDF5 configured and installed in a specific way and your installations of Python, HDF5 and the module must have been compiled with the same compilers. You can find `read_eagle` [here](https://github.com/jchelly/read_eagle), but I don't recommend trying it at the ARI - I made it work at the start of my PhD but have been unable to replicate that success for others since!

### GUI for examining .hdf5 files

HDFView







