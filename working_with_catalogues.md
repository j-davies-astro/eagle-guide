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
        # Use h5py to initialise the catalogue file
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

This code is totally general - you could set `table` to `'Subhalo'` instead and load something in from the SUBFIND catalogue. Try running it for yourself!

A couple of notes:

- Remember to run this code on one of the cluster machines and **not** on `external1/2`.

- We're working with the `Ref-L0025N0376` simulation here. It's always best to prototype and test your code on the smaller simulation volumes first, before going for the big `Ref-L0100N1504` box. The 100 Mpc volume is 16 times larger than the 25 Mpc volumes, so things will take at least 16 times as long to run - likely even longer as things probably won't scale perfectly!

- I recommend using `h5py` in conjunction with a `with` statement, as shown above, for speed.

- You need to do that step of converting the loaded data into a numpy array. `data` here is an HDF5 object, which can be sliced and indexed, but can't be fully manipulated as a regular numpy array.


### Unit conversions

If you successfully ran the code above, you'll notice that the data we loaded doesn't look like it corresponds to halo mass. We need to convert the units - right now we're in h-less comoving GADGET units. Conveniently, all the conversion factors we need can be loaded as attributes from our hdf5 file, as I discussed in the previous section. Lets modify the above example so it does unit corrections automatically:

```python
import numpy as np
import h5py as h5

sim = 'L0025N0376'
model = 'REFERENCE'
tag = '019_z001p004'

table = 'FOF'
quantity = 'Group_M_Crit200'

subfindfile = '/hpcdata0/simulations/EAGLE/' + sim + '/' + model + '/data/' + 'groups_'+tag+'/eagle_subfind_tab_' + tag + '.'

file_ind = 0

while True:

    try:
        with h5.File(subfindfile+str(file_ind)+'.hdf5', 'r') as f:
            
            data = f['/'+table+'/%s'%(quantity)]                        

            if file_ind == 0:

                # We can grab all our conversion factors from the first file...
                h_scale_exponent = data.attrs['h-scale-exponent']
                a_scale_exponent = data.attrs['aexp-scale-exponent']
                cgs_conversion_factor = data.attrs['CGSConversionFactor']

                # ... and get the expansion factor and little h from the header
                h = f['Header'].attrs['HubbleParam']
                a = f['Header'].attrs['ExpansionFactor']

                print('Loading ',quantity)
                print('h exponent = ',h_scale_exponent)
                print('a exponent = ',a_scale_exponent)
                print('cgs conversion factor = ',cgs_conversion_factor)
                print('h = ',h)
                print('a = ',a)

                data_arr = np.array(data)

            else:
                data_arr = np.append(data_arr,np.array(data),axis=0)

    except OSError:
        print('Run out of files after loading ',file_ind)
        break

    file_ind += 1
        
# Multiply by h and a, raised to the correct powers, to get the answer in physical GADGET units
M200_gadgetunits = data_arr * np.power(h,h_scale_exponent) * np.power(a,a_scale_exponent)

# Multiply that by the cgs conversion factor to get the answer in grams
M200_cgs = M200_gadgetunits * cgs_conversion_factor

# Alternatively, recall that the unit mass is 10^10 solar masses, so to get M200 in solar masses
M200_sol = M200_gadgetunits * 1e10

print(M200_gadgetunits)
print(M200_cgs)
print(M200_sol)
```
Since we're at redshift 1, you'll see in the output that the expansion factor `a` is about 0.5, however since we're loading in a mass, no correction was needed for expansion factor and `a_scale_exponent` was 0. We did, however, need to multiply by `h^-1` to get a physical mass.

### A function for easy catalogue reading

Clearly, it would be a total pain to do all the above every time we wanted to load something from the catalogues. Since all the appropriate conversion factors can be easily obtained, however, it's dead easy to wrap this up in a function that automates the process. It would look something like this:

```python
def catalogue_read(table,quantity,
                    sim = 'L0025N0376',
                    model = 'REFERENCE',
                    tag = '028_z000p000',
                    data_location = '/hpcdata0/simulations/EAGLE/',
                    phys_units=True,
                    cgs_units=False,
                    verbose=False):

    # Do a quick check to make sure a valid table has been specified
    assert table in ['FOF','Subhalo'],'table must be either FOF or Subhalo'

    subfindfile = data_location + sim + '/' + model + '/data/' + 'groups_'+tag+'/eagle_subfind_tab_' + tag + '.'

    file_ind = 0
    while True:

        try:
            with h5.File(subfindfile+str(file_ind)+'.hdf5', 'r') as f:
                                    
                data = f['/'+table+'/'+quantity]

                if file_ind == 0:

                    h_scale_exponent = data.attrs['h-scale-exponent']
                    a_scale_exponent = data.attrs['aexp-scale-exponent']
                    cgs_conversion_factor = data.attrs['CGSConversionFactor']
                    h = f['Header'].attrs['HubbleParam']
                    a = f['Header'].attrs['ExpansionFactor']

                    # Let's only print this stuff out if the user wants it!
                    if verbose:
                        print('Loading ',quantity)
                        print('h exponent = ',h_scale_exponent)
                        print('a exponent = ',a_scale_exponent)
                        print('cgs conversion factor = ',cgs_conversion_factor)
                        print('h = ',h)
                        print('a = ',a)

                    # Lets be a bit more cautious and make sure we cast our HDF5 dataset into an array of the correct type
                    dt = data.dtype
                    data_arr = np.array(data,dtype=dt)
            
                else:
                    data_arr = np.append(data_arr,np.array(data,dtype=dt),axis=0)

        except OSError:
            print('Run out of files after loading ',file_ind)
            break

        file_ind += 1
    
    # If the data we're loading is integer-type, no corrections will be needed
    if np.issubdtype(dt,np.integer):
        return data_arr

    # Otherwise, do unit corrections
    else:

        if phys_units:
            data_arr *= np.power(h,h_scale_exponent) * np.power(a,a_scale_exponent)

        if cgs_units:

            # cgs numbers can be huge and overflow np.float32
            # Recast the data to float64 to be safe
            data_arr = np.array(data_arr,dtype=np.float64)

            data_arr *= cgs_conversion_factor

        return data_arr
```

A few things to point out here:

- As you can see, the user need only specify the table (`FOF` or `Subhalo`) and the quantity they want to load, and make sure they specify the correct simulation details. I've set the present-day Ref-L0025N0376 snapshot as the default.

- By default, the data will be loaded in physical GADGET units. `cgs_units=True` can be set to return the answer in CGS.

- Note that I now obtain the correct data type from the HDF5 dataset and make sure to cast the data into an array of the correct type. This is important for loading quantities that are integers (such as `GroupNumber`), as simply using `np.array()` will convert these into floats.

- When converting to cgs units, I recast the data to `np.float64` to be safe. Asking for the mass of a DM halo in grams will return an extremely large number which will overflow a 32-bit float and return `inf`, so it's essential to do this.

With this function in hand, it's now easy to load whatever you like from the catalogues. For example, the following script will produce the same output as we got in the previous section:

```python
sim = 'L0025N0376'
model = 'REFERENCE'
tag = '019_z001p004'

M200_gadgetunits = catalogue_read('FOF','Group_M_Crit200',sim=sim,model=model,tag=tag)
M200_cgs = catalogue_read('FOF','Group_M_Crit200',sim=sim,model=model,tag=tag,cgs_units=True)
M200_sol = M200_gadgetunits * 1e10

print(M200_gadgetunits)
print(M200_cgs)
print(M200_sol)
```


### Aperture measurements

### Constructing samples of galaxies/subhaloes
