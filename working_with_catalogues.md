## Working with the catalogues

Let's have a go at loading in some data! We'll start with the catalogues, as we don't need `pyread_eagle` to load these in - just the python module `h5py`.

### Loading in catalogues

If you remember, the catalogues are split across many files. To load in a dataset from them, we must load it in from each file and append it to a numpy array in a loop. Here's how I would go about loading in `Group_M_Crit200` from the `FOF` table:

```python
# Import modules. This basic example requires only numpy and h5py
import numpy as np
import h5py as h5


# It's a good idea to create variables for the simulation details at the top
sim = 'L0025N0376'      # Initial conditions
model = 'REFERENCE'     # Model
tag = '019_z001p004'    # Snapshot identifier, or 'tag' as people often call it


# Establish the table we want to load in from, and what we want to load
table = 'FOF'
quantity = 'Group_M_Crit200'

# Remember the catalogues have filenames like 'eagle_subfind_tab_019_z001p004.0.hdf5'
# That number at the end is what we want to loop over
# Let's establish the majority of the file path as a string to start with
subfindfile = '/hpcdata0/simulations/EAGLE/' + sim + '/' + model + '/data/' + 'groups_'+tag+'/eagle_subfind_tab_' + tag + '.'

# I like to loop over an integer called file_ind in a while loop, incrementing it each time a file is loaded
file_ind = 0    # Initialise the file index

while True:

    try:
        # Use h5py to load in the file
        with h5.File(subfindfile+str(file_ind)+'.hdf5', 'r') as f:
            
            # h5py lets you access the contents like a dictionary. Load in the '/FOF/Group_M_Crit200' dataset
            data = f['/'+table+'/%s'%(quantity)]                        

            if file_ind == 0:
                # If this is the first file, convert the dataset into a numpy array
                data_arr = np.array(data)
            else:
                # If this isn't the first file, add the loaded data to the array
                data_arr = np.append(data_arr,np.array(data),axis=0)

    # When we run out of files to load in, an OS exception will be raised.
    # Catch it to finish loading in files and break from the loop.
    except OSError:
        print('Run out of files after loading ',file_ind)
        break

    # Increment the file index
    file_ind += 1
        
# Print the loaded data!
print(data_arr)
```


### Unit conversions

### A function for easy catalogue reading

### Constructing samples of galaxies/subhaloes
