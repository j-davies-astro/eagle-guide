## Introduction

With this brief guide, I hope to get you up and running with the EAGLE simulation data at the Astrophysics Research Institute. I'll explain how to install useful python modules for reading simulation data, how to use them to extract information about dark matter haloes, galaxies and simulated particles, and give examples of some basic data reduction and analysis you'll likely need to do when working with EAGLE. 

### Prerequisites

- We will be working in the Ubuntu (Linux) operating system, which runs on all ARI _starpc_ machines. If you're working remotely or using a personal computer, you can access the ARI systems via the machines _external_ and _external2_. This can be done two different ways:

  - **Command line:** Type `ssh <your_username>@external2.astro.ljmu.ac.uk>` and type in your password when prompted. Unfortunately, X11 forwarding is blocked, so you won't be able to use any graphical interfaces through this method. A better method is to use...
  - **NoMachine:** You can access a full Linux environment remotely using NoMachine. Please see [the ARI Docs pages](https://www.astro.ljmu.ac.uk/docs/) for information on how to set this up.
  
- All code will be run from a Linux terminal/command line. By default, the ARI systems use `csh`. I recommend you get familiar with some basic Linux commands if you aren't already - commands like `cd`, `ls`, `top`, `grep`, `more` and `tail` will be invaluable to you.

- This guide is for working with EAGLE in Python 3, and will primarily make use of the standard packages `numpy`, `matplotlib`, and `h5py`. The Python environment and these packages should already be set up for you on the ARI systems. Other very useful packages include:
  
  - Other standard python modules such as `sys`, `os`, and `copy`
  - `astropy` for doing, among other things, cosmological calculations
  - `tqdm` for showing progress bars while your code runs


```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/j-davies-ari/eagle-guide/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
