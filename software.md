# Installing software

Let's install two key bits of software for working with EAGLE.

## Python read routines

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

## GUI for examining HDF5 files

EAGLE simulation data is stored in the [HDF5](https://www.hdfgroup.org/solutions/hdf5/) file format, which allows for fast I/O and storage of very large datasets. These files act more like Python dictionaries than simple text files, and so it's a little more difficult to have a direct look at what's actually _in_ the files. 

To make this easier, download the HDFView software, which can be found [here](https://www.hdfgroup.org/downloads/hdfview/#download), onto your space on the /home drive. Inside this package should be a file named `hdfview.sh`. Run this to open the application.

## Creating aliases (command-line shortcuts)

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
