## Working with the catalogues

Let's have a go at loading in some data! We'll start with the catalogues, as we don't need `pyread_eagle` to load these in - just the python module `h5py`.

### Loading in catalogues

If you remember, the catalogues are split across many files. To load in a dataset from them, we must load it in from each file and append it to a numpy array in a loop. Here's how I would go about loading in `Group_M_Crit200` from the `FOF` table:

```python
import numpy as np
import h5py as h5
import sys
from sys import exit
from copy import deepcopy

##############################
sim = 'L0025N0376'
model = 'REFERENCE'
tag = '019_z001p004'
##############################

##############################
table = 'FOF'
quantity = 'Group_M_Crit200'
##############################


subfindfile = '/hpcdata0/simulations/EAGLE/' + sim + '/' + model + '/data/' + 'groups_'+tag+'/eagle_subfind_tab_' + tag + '.'


file_ind = 0
while True:

    try:
        with h5.File(subfindfile+str(file_ind)+'.hdf5', 'r') as f:
                
            data = f['/'+table+'/%s'%(quantity)]

            if file_ind == 0:
                
                data_arr = np.array(data)
        
            else:
                data_arr = np.append(data_arr,np.array(data),axis=0)

    except OSError:
        print('Run out of files after loading ',file_ind)
        break

    file_ind += 1
        

print(data_arr)
```


### Unit conversions

### A function for easy catalogue reading

### Constructing samples of galaxies/subhaloes
