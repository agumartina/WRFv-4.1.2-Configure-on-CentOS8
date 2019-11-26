# WRFv-4.1.2 Configure on CentOS 8
Configuration of WRF 4.1 on CenOS
These are some installation notes taken in the process of installing WRF version 4.1.2 on a computer with CentOS 8.


## Install required software


```console
$ sudo yum groupinstall 'Development Tools'
$ sudo yum install csh gfortran m4
```

## System environment tests

First and foremost, it is very important to have a gfortran compiler, as well as gcc and cpp. If you have these installed, you should be given a path for the location of each.
```console
$ which gfortran
/user/bin/gfortran
$ which cpp
/user/bin/cpp
$ which gcc
/user/bin/gcc
```

Check your gcc version. It is recommend using version 4.4.0 or later.
```console
$ gcc --version
gcc (GCC) 8.2.1 20180905 (Red Hat 8.2.1-3)
Copyright (C) 2018 Free Software Foundation, Inc.
Esto es software libre; vea el código para las condiciones de copia.  NO hay
garantía; ni siquiera para MERCANTIBILIDAD o IDONEIDAD PARA UN PROPÓSITO EN
PARTICULAR
```

Create a new, clean directory called `source`, `wrf_io`, `wrfout`, `salidas_wrf`, `GFS025` and another one called `TESTS`.

There are a few simple tests that can be run to verify that the fortran compiler is built properly, and that it is compatible with the C compiler. Download the tar file that contains the tests into the `TESTS` directory and unpack the tar file.
```console
$ cd ~/TESTS
$ wget http://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/Fortran_C_tests.tar
$ tar -xvf Fortran_C_tests.tar
```

There are 7 tests available, so start at the top and run through them, one at a time.

* Test 1: Fixed Format Fortran Test.
```console
$ gfortran TEST_1_fortran_only_fixed.f
$ ./a.out
SUCCESS test 1 fortran only fixed format
```

* Test 2: Free Format Fortran.
```console
$ gfortran TEST_2_fortran_only_free.f90
$ ./a.out
Assume Fortran 2003: has FLUSH, ALLOCATABLE, derived type, and ISO C Binding
SUCCESS test 2 fortran only free format
```

* Test 3: C.
```console
$ gcc TEST_3_c_only.c
$ ./a.out
SUCCESS test 3 c only
```

* Test 4: Fortran Calling a C Function (our gcc and gfortran have different defaults, so we force both to always use 64 bit [-m64] when combining them).
```console
$ gcc -c -m64 TEST_4_fortran+c_c.c
$ gfortran -c -m64 TEST_4_fortran+c_f.f90
$ gfortran -m64 TEST_4_fortran+c_f.o TEST_4_fortran+c_c.o
$ ./a.out
C function called by Fortran Values are xx = 2.00 and ii = 1
SUCCESS test 4 fortran calling c
```

In addition to the compilers required to manufacture the WRF executables, the WRF build system has scripts as the top level for the user interface. The WRF scripting system uses, and therefore is necessary having csh, perl and sh.
To test whether these scripting languages are working properly on the system, there are 3 tests to run. These tests were included in the "Fortran and C Tests Tar File".

* Test 5: csh.
```console
$ csh ./TEST_csh.csh
SUCCESS csh test
```

* Test 6: perl.
```console
$ ./TEST_perl.pl
SUCCESS perl test
```

* Test 7: sh.
```console
$ ./TEST_sh.sh
SUCCESS sh test
```

## Build Librarties and dependences

**We need to set where to find sources for libraries**
```console
export LIBSRC=$HOME/source
mkdir -p $LIBSRC

export LIBBASE=$HOME/wrf_io
mkdir -p $LIBBASE

export NETCDF=$LIBBASE
export ZLIB=$LIBBASE
export HDF5=$LIBBASE
export JASPER=$LIBBASE
export PHDF5=$LIBBASE
```

**First, whe need MPI libraries to compile all**
```console
$ sudo yum install openmpi-devel
$ tar xfz mpich.tar.gz
$ gunzip -c mpich.tar.gz | tar xf -
$ mkdir $HOME/mpich-install
$ cd $HOME/mpich-install
$ ./configure \-prefix=/home/you/mpich-install |& tee c.txt
$ make 2>&1 | tee m.txt
$ make install
$ PATH=/home/you/mpich-install/bin:$PATH
$ export PATH
$ mpicc --version
gcc (GCC) 8.2.1 20180905 (Red Hat 8.2.1-3)
Copyright (C) 2018 Free Software Foundation, Inc.
Esto es software libre; vea el código para las condiciones de copia.  NO hay
garantía; ni siquiera para MERCANTIBILIDAD o IDONEIDAD PARA UN PROPÓSITO EN
PARTICULAR
```

**It is important to note that these libraries must all be installed with the same compilers as will be used to install WRFV4 and WPS. So wee need to set a few enviroment variabñles to tell the conpires which compilers and flags use**

```console
export SERIAL_FC=gfortran
export SERIAL_F77=gfortran
export SERIAL_CC=gcc
export SERIAL_CXX=g++
export MPI_FC=mpifort
export MPI_F77=mpif77
export MPI_CC=mpicc
export MPI_CXX=mpicc
```

**ZLib install**
Configuring zlib: This is a compression library necessary for compiling WPS (specifically ungrib) with GRIB2 capability.
Assuming all the **export** commands from the NetCDF install are already set, you can move on to the commands to install zlib.

```console
$ wget -nc https://zlib.net/zlib-1.2.11.tar.gz
$ tar xzvf zlib-1.2.11.tar.gz
$ cd zlib-1.2.11
$ ./configure --prefix=${LIBBASE}
$ make
$ make install
$ cd ..
```

**HDF5 parallel install**
```console
$ wget -nc https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.10/hdf5-1.10.5/src/hdf5-1.10.5.tar.gz
$ tar xvzf hdf5-1.10.5.tar.gz
$ cd hdf5-1.10.5
$ export FC=$MPI_FC
$ export CC=$MPI_CC
$ export CXX=$MPI_CXX

$ ./configure --prefix=${HDF5} --enable-parallel --with-zlib=${ZLIB} --enable-fortran --enable-shared
$ make
$ make check
$ make install
$ cd ..
```

**netCDF (C library)**
```console
$ wget -nc https://github.com/Unidata/netcdf-c/archive/v4.7.2.tar.gz
$ tar xzvf v4.7.2.tar.gz
$ cd netcdf-c-4.7.2
$ CC=mpicc CPPFLAGS="-I${HDF5}/include" LDFLAGS="-L${HDF5}/lib" ./configure --enable-shared --enable-parallel-tests --enable-netcdf4 --disable-filter-testing --disable-dap --prefix=${NETCDF}
$ make check
$ make install
$ cd ..
```

**netCDF (Fortran interface library)**

```console
$ wget -nc https://github.com/Unidata/netcdf-fortran/archive/v4.5.2.tar.gz
$ tar xzvf v4.5.2.tar.gz
$ cd netcdf-fortran-4.5.2
$ export FC=$MPI_FC
$ export F77=$MPI_F77
$ CPPFLAGS="-I${NETCDF}/include" LDFLAGS="-L${NETCDF}/lib" LD_LIBRARY_PATH=${NETCDF}/lib LIBS="-lnetcdf -lhdf5_hl -lhdf5 -lz" ./configure --enable-parallel-tests --enable-shared --prefix=${NETCDF}
$ make check
$ make install
$ cd ..
```


**JASPER**
Configuring JasPer: This is a compression library necessary for compiling WPS (specifically ungrib) with GRIB2 capability.
Assuming all the **export** commands from the NetCDF install are already set, you can move on to the commands to install jasper.

```console
$ wget -nc http://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/jasper-1.900.1.tar.gz
$ tar xvzf jasper-1.900.1.tar.gz
$ cd jasper-1.900.1
$ ./configure --prefix=$LIBBASE
$ make
$ make install
$ export JASPERLIB=$LIBBASE/lib
$ export JASPERLIB=$LIBBASE/include
```

**LIBPNG**
This is a compression library necessary for compiling WPS (specifically ungrib) with GRIB2 capability.
Assuming all the **export** commands from the NetCDF install are already set, you can move on to the commands to install libpng.
```console
$ wget -nc http://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/libpng-1.2.50.tar.gz
$ tar xvzf libpng-1.2.50.tar.gz 
$ cd libpng-1.2.50/
$ ./configure --prefix=$LIBBASEm
$ make
$ make install
$ cd ..
```

## Libraries compatibility tests

Once the target machine is able to make small Fortran and C executables (what was verified in the System Environment Tests section), and after the NetCDF and MPI libraries are constructed (two of the libraries from the Building Libraries section), to emulate the WRF code's behavior, two additional small tests are required. We need to verify that the libraries are able to work with the compilers that are to be used for the WPS and WRF builds.

Move to `TESTS` directory, download the tar file that contans these tests and unpack it.
```console
$ cd ../TESTS
$ wget http://www2.mmm.ucar.edu/wrf/OnLineTutorial/compile_tutorial/tar_files/Fortran_C_NETCDF_MPI_tests.tar
$ tar -xvf Fortran_C_NETCDF_MPI_tests.tar
$ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${LIBBASE}/lib
```

There are 2 tests.

* Test 1: Fortran + C + NetCDF

The NetCDF-only test requires the include file from the NETCDF package be in this directory. Copy the NetCDF include here and compile the Fortran and C codes for the purpose of this test (the -c option says to not try to build an executable).
```console
$ cp ${NETCDF}/include/netcdf.inc .
$ gfortran -c 01_fortran+c+netcdf_f.f
$ gcc -c 01_fortran+c+netcdf_c.c
$ gfortran 01_fortran+c+netcdf_f.o 01_fortran+c+netcdf_c.o -L${NETCDF}/lib -lnetcdff -lnetcdf
$ ./a.out
```

The following should be displayed on your screen.
```console
C function called by Fortran
Values are xx = 2.00 and ii = 1
SUCCESS test 1 fortran + c + netcdf
```

* Test 2: Fortran + C + NetCDF + MPI

The NetCDF+MPI test requires include files from both of these packages be in this directory, but the MPI scripts automatically make the `mpif.h` file available without assistance, so no need to copy that one. Copy the NetCDF include file here and note that the MPI executables `mpif90` and `mpicc` are used below when compiling. Issue the following commands.
```console
$ cp ${NETCDF}/include/netcdf.inc .
$ mpif90 -c 02_fortran+c+netcdf+mpi_f.f
$ mpicc -c 02_fortran+c+netcdf+mpi_c.c
$ mpif90 02_fortran+c+netcdf+mpi_f.o 02_fortran+c+netcdf+mpi_c.o -L${NETCDF}/lib -lnetcdff -lnetcdf
$ mpirun ./a.out
```

The following should be displayed on your screen.
```console
C function called by Fortran
Values are xx = 2.00 and ii = 1
status = 2
SUCCESS test 2 fortran + c + netcdf + mpi
```

## Building WRFV4

First of all, we need to download the source code from
```console
$ wget https://github.com/wrf-model/WRF/archive/v4.1.2.tar.gz 
$ tar xvzf v4.1.2.tar.gz 
$ cd WRF-4.1.2/
```
Also, if you are runnin over an Intel machine, you should add
```console
$ export WRF_EM_CORE=1
$ export PHDF5=$HDF5
```
and edit the default config to eneable Jasper

```console
$ nano arch/Config.pl
$ $I_really_want_to_output_grib2_from_WRF = "TRUE" ;
```
Now we are able to run the confgure 

```console
$ ./configure
```
You will see various options. Choose the option that lists the compiler you are using and the way you wish to build WRFV3 (i.e., serially or in parallel). Although there are 3 different types of parallel (smpar, dmpar, and dm+sm), it is recommend choosing dmpar option.

```console
checking for perl5... no
checking for perl... found /usr/bin/perl (perl)
Will use NETCDF in dir: /home/wrf/wrf_io
HDF5 not set in environment. Will configure WRF for use without.
Will use PHDF5 in dir: /home/wrf/wrf_io
Will use 'time' to report timing information
Configuring to use jasper library to build Grib2 I/O...
  $JASPERLIB = /home/wrf/wrf_io/lib
  $JASPERINC = /home/wrf/wrf_io/include
------------------------------------------------------------------------
Please select from among the following Linux x86_64 options:

  1. (serial)   2. (smpar)   3. (dmpar)   4. (dm+sm)   PGI (pgf90/gcc)
  5. (serial)   6. (smpar)   7. (dmpar)   8. (dm+sm)   PGI (pgf90/pgcc): SGI MPT
  9. (serial)  10. (smpar)  11. (dmpar)  12. (dm+sm)   PGI (pgf90/gcc): PGI accelerator
 13. (serial)  14. (smpar)  15. (dmpar)  16. (dm+sm)   INTEL (ifort/icc)
                                         17. (dm+sm)   INTEL (ifort/icc): Xeon Phi (MIC architecture)
 18. (serial)  19. (smpar)  20. (dmpar)  21. (dm+sm)   INTEL (ifort/icc): Xeon (SNB with AVX mods)
 22. (serial)  23. (smpar)  24. (dmpar)  25. (dm+sm)   INTEL (ifort/icc): SGI MPT
 26. (serial)  27. (smpar)  28. (dmpar)  29. (dm+sm)   INTEL (ifort/icc): IBM POE
 30. (serial)               31. (dmpar)                PATHSCALE (pathf90/pathcc)
 32. (serial)  33. (smpar)  34. (dmpar)  35. (dm+sm)   GNU (gfortran/gcc)
 36. (serial)  37. (smpar)  38. (dmpar)  39. (dm+sm)   IBM (xlf90_r/cc_r)
 40. (serial)  41. (smpar)  42. (dmpar)  43. (dm+sm)   PGI (ftn/gcc): Cray XC CLE
 44. (serial)  45. (smpar)  46. (dmpar)  47. (dm+sm)   CRAY CCE (ftn $(NOOMP)/cc): Cray XE and XC
 48. (serial)  49. (smpar)  50. (dmpar)  51. (dm+sm)   INTEL (ftn/icc): Cray XC
 52. (serial)  53. (smpar)  54. (dmpar)  55. (dm+sm)   PGI (pgf90/pgcc)
 56. (serial)  57. (smpar)  58. (dmpar)  59. (dm+sm)   PGI (pgf90/gcc): -f90=pgf90
 60. (serial)  61. (smpar)  62. (dmpar)  63. (dm+sm)   PGI (pgf90/pgcc): -f90=pgf90
 64. (serial)  65. (smpar)  66. (dmpar)  67. (dm+sm)   INTEL (ifort/icc): HSW/BDW
 68. (serial)  69. (smpar)  70. (dmpar)  71. (dm+sm)   INTEL (ifort/icc): KNL MIC
 72. (serial)  73. (smpar)  74. (dmpar)  75. (dm+sm)   FUJITSU (frtpx/fccpx): FX10/FX100 SPARC64 IXfx/Xlfx

Enter selection [1-75] : 34
------------------------------------------------------------------------
Compile for nesting? (1=basic, 2=preset moves, 3=vortex following) [default 1]: 1

Configuration successful! 
------------------------------------------------------------------------
testing for fseeko and fseeko64
fseeko64 is supported
------------------------------------------------------------------------
Settings listed above are written to configure.wrf.
If you wish to change settings, please edit that file.
If you wish to change the default options, edit the file:
     arch/configure.defaults
NetCDF users note:
 This installation of NetCDF supports large file support.  To DISABLE large file
 support in NetCDF, set the environment variable WRFIO_NCD_NO_LARGE_FILE_SUPPORT
 to 1 and run configure again. Set to any other value to avoid this message.
  

Testing for NetCDF, C and Fortran compiler

This installation of NetCDF is 64-bit
                 C compiler is 64-bit
           Fortran compiler is 64-bit
              It will build in 64-bit

*****************************************************************************
This build of WRF will use classic (non-compressed) NETCDF format
*****************************************************************************

```

Once your configuration is complete, you should have a `configure.wrf` file, and you are ready to compile. To compile WRFV4.1.2, you will need to decide which type of case you wish to compile. The options are listed below.
```console
em_real (3d real case)
em_quarter_ss (3d ideal case)
em_b_wave (3d ideal case)
em_les (3d ideal case)
em_heldsuarez (3d ideal case)
em_tropical_cyclone (3d ideal case)
em_hill2d_x (2d ideal case)
em_squall2d_x (2d ideal case)
em_squall2d_y (2d ideal case)
em_grav2d_x (2d ideal case)
em_seabreeze2d_x (2d ideal case)
em_scm_xy (1d ideal case)
```

For this purpose we are going to compile WRF for real cases. Compilation should take about 20-30 minutes. The ongoing compilation can be checked.
```console
$ ./compile em_real >& compile.log &
$ tail -f compile.log
```

If we see this message, you done it right ;)

```console
==========================================================================
build started:   lun nov 18 21:48:48 -03 2019
build completed: lun nov 18 21:56:41 -03 2019
 
--->                  Executables successfully built                  <---
 
-rwxrwxr-x. 1 wrf wrf 57432880 nov 18 21:56 main/ndown.exe
-rwxrwxr-x. 1 wrf wrf 57309864 nov 18 21:56 main/real.exe
-rwxrwxr-x. 1 wrf wrf 56831120 nov 18 21:56 main/tc.exe
-rwxrwxr-x. 1 wrf wrf 61189576 nov 18 21:56 main/wrf.exe
 
==========================================================================
```

Once the compilation completes, to check whether it was successful, you need to look for executables in the `WRFV3/main` directory.
```console
$ ls -las main/*.exe
ndown.exe (one-way nesting)
real.exe (real data initialization)
tc.exe (for tc bogusing--serial only)
wrf.exe (model executable)
```

These executables are linked to 2 different directories. You can choose to run WRF from either directory.
```console
WRF-4.1.2/run
WRF-4.1.2//test/em_real
```
export WRF_DIR=$HOME/WRF-4.1.2

Now we need to download and compile WPS


## Building WPS

NOTE: If you choosed in WRF to run on shared memory architecture (smpar), you need to add the flag of OpenMP (-lgomp) to the WRF_LIB variable in file configure.wps (just append it after -lnetcdf).

After the WRF model is built, the next step is building the WPS program (if you plan to run real cases, as opposed to idealized cases). The WRF model MUST be properly built prior to trying to build the WPS programs. If you do not already have the WPS source code, move to your `Build_WRF` directory, download that file and unpack it. Then go into the WPS directory and make sure the WPS directory is clean.

```console
$ cd {path_to_dir}/Build_WRF
$ wget http://www2.mmm.ucar.edu/wrf/src/WPSV3.8.TAR.gz
$ tar -zxvf WPSV3.8.TAR.gz
$ cd {path_to_dir}/Build_WRF/WPS
$ ./clean
```


wget https://github.com/wrf-model/WPS/archive/v4.1.tar.gz
tar xvzf v4.1.tar.gz
cd WPS-4.1
./clean
 export WRF_DIR=$HOME/WRF-4.1.2

The next step is to configure WPS, however, you first need to set some paths for the ungrib libraries and then you can configure.
./configure

we need to choose 3
