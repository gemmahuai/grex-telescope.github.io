# Data Processing

## Voltage Dumps

The voltage dumps are stored in [NetCDF](https://www.unidata.ucar.edu/software/netcdf/), a machine-independent
data format intended for array-oriented scientific data such as ours. This format is a subset of HDF5 and is
supported by all major programming languages. Not only that, but it is self describing! Our files contain axis
variables on each dimension to remove any ambiguity as to which coordinate each data point refers to.

Using the NetCDF binaries, we can quickly introspect a voltage dump to see how it is laid out

```shell
user@grex:/hdd/data/voltages$ ncdump -h grex_dump-20240122T055935.nc
netcdf grex_dump-20240122T055935 {
dimensions:
        time = 524288 ;
        pol = 2 ;
        freq = 2048 ;
        reim = 2 ;
variables:
        double time(time) ;
                time:units = "Days" ;
                time:long_name = "Dynamic Barycentric Time (TDB) since J2000" ;
        string pol(pol) ;
                pol:long_name = "Polarization" ;
        double freq(freq) ;
                freq:units = "Megahertz" ;
                freq:long_name = "Frequency" ;
        string reim(reim) ;
                reim:long_name = "Complex" ;
        byte voltages(time, pol, freq, reim) ;
                voltages:long_name = "Channelized Voltages" ;
                voltages:units = "Volts" ;
}
```

Part of the trickiness here is HDF5 (and NetCDF by extension) doesn't support complex numbers, so the real and imaginary components are stored independently as another dimension.

### Python Reading Example

In Python, an excellent library for dealing with dimensional data is [xarray](https://docs.xarray.dev/en/stable/getting-started-guide/index.html). This
library supports a large number of file formats, which are optional dependencies.
For us, we need the netCDF4 python package, but it's probably best just to
use the complete installation.

It might also help if you read the xarray [page on NetCDF](https://docs.xarray.dev/en/stable/user-guide/io.html).

Reading this file with

```python
xarray.open_dataset("grex_dump.nc")
```

will associate all the axes to the appropriate dimensions, we just have to construct the complex numbers manually.

A one-liner to do this would be

```python
voltages = ds["voltages"].sel(reim="real") + ds["voltages"].sel(reim="imaginary")*1j
```

This may take a while for large dumps though, and you may want to slice it up into chunks (not sure if there is an elegant way to do that here).

But now you can do eveything xarray can do!

A few examples:

#### Stokes I

Calculated by taking the time-average of the sum of magnitude squared of the voltages

```python
stokesi = (abs(voltages)**2).sum(dim="pol").mean(dim="time")
```

#### H1 Line

Get the Stokes intensity around the H1 line (+/- 1 MHz) as a function a time

```python
h1 = (abs(voltages)**2).sum(dim="pol").sel(freq=slice(1421,1419)).sum(dim="freq")
```