[![linux](https://github.com/hpc-io/vol-async/actions/workflows/linux.yml/badge.svg?branch=develop)](https://github.com/hpc-io/vol-async/actions/workflows/linux.yml)
# HDF5 Asynchronous I/O VOL Connector

[**Full documentation**](https://hdf5-vol-async.readthedocs.io)

## 1. Background

Asynchronous I/O is becoming increasingly popular with the large amount of data access required by scientific applications. They can take advantage of an asynchronous interface by scheduling I/O as early as possible and overlap computation or communication with I/O operations, which hides the cost associated with I/O and improves the overall performance. This work is part of the [ECP-ExaIO](https://www.exascaleproject.org/research-project/exaio) project.

[<img src="https://lh3.googleusercontent.com/pw/AM-JKLX033FP6RFe5CqYx7vQY_YF834O4SOfFr53xzUdB-TOGIVnG-jNn0fp-8aHbgqZtogRlgSNHJxQqI8gAG0sZo3HNOhmf3k8GZpFyvz2sCBEl2lekbOh8ne3TJyAjbP0XbVZ79JczoDe3pqSIjbfJa-M=w3090-h613-no?authuser=0">](overview)



Some configuration parameters used in the instructions:

    VOL_DIR :  directory of HDF5 Asynchronous I/O VOL connector repository
    ABT_DIR :  directory of Argobots source code
    HDF5_DIR:  directory of HDF5 source code
        
We have tested async VOL compiled with GNU(gcc 6.4+), Intel, and Cray compilers on Summit, Cori, and Theta supercomputers.

## 2. Installation
    
### 2.1 Installation with Spack
[Spack](https://spack.io) is a flexible package manager that supports multiple versions, configurations, platforms, and compilers. 
Async VOL and its dependent libraries (MPI, HDF5, Argobots) can all be installed with the following spack command:

    spack install hdf5-vol-async
    
### 2.2 Installation with Makefile

2.2.1 Download and Compile HDF5

Async VOL requires HDF5 1.13+, which can be downloaded from [THG](https://portal.hdfgroup.org/display/support/Downloads) or [GitHub](https://github.com/HDFGroup/hdf5.git)

    cd $HDF5_DIR
    ./autogen.sh  (may skip this step if the configure file exists)
    ./configure --prefix=$HDF5_DIR/install --enable-parallel --enable-threadsafe --enable-unsupported 
    # may need to add CC=cc or CC=mpicc to the above configure command
    make && make install

2.2.2 Download and Compile Argobots

The latest Argobots can also be downloaded from [GitHub](https://github.com/pmodels/argobots) or as a submodule from the async VOL git repo (see next step).
    
    cd $ABT_DIR
    ./autogen.sh  (may skip this step if the configure file exists)
    ./configure --prefix=$ABT_DIR/install
    # may need to add CC=cc or CC=mpicc to the above configure command
    make && make install
    # Note: using mpixlC on Summit may result in Argobots runtime error, use xlC or gcc instead.

2.2.3 Download and Compile Async VOL connector

    git clone --recursive https://github.com/hpc-io/vol-async.git
    cd $VOL_DIR/src
    # Edit "Makefile" or use a template Makefile for existing systems: e.g. "cp Makefile.summit Makefile"
    # Change the path of HDF5_DIR and ABT_DIR to $HDF5_DIR/install and $ABT_DIR/install
    # (Optional) update the compiler flag macros: DEBUG, CFLAGS, LIBS, ARFLAGS
    # (Optional) comment/uncomment the correct DYNLDFLAGS & DYNLIB macros
    make


## 3. Set Environment Variables

Will need to set the following environmental variable before running the asynchronous operation tests and your async application, e.g.:

for Linux:

    export LD_LIBRARY_PATH=$VOL_DIR/src:$HDF5_DIR/install/lib:$ABT_DIR/install/lib:$LD_LIBRARY_PATH
    export HDF5_PLUGIN_PATH="$VOL_DIR/src"
    export HDF5_VOL_CONNECTOR="async under_vol=0;under_info={}" 
    # (optional) Some systems like Cori@NERSC need the following to enable MPI_THREAD_MULTIPLE 
    export MPICH_MAX_THREAD_SAFETY=multiple # 

MacOS:

    export DYLD_LIBRARY_PATH=$VOL_DIR/src:$HDF5_DIR/install/lib:$ABT_DIR/install/lib:$DYLD_LIBRARY_PATH
    export HDF5_PLUGIN_PATH="$VOL_DIR/src"
    export HDF5_VOL_CONNECTOR="async under_vol=0;under_info={}" 

## 4. Test
    # Similar to 2.3
    cd $VOL_DIR/test
    # Edit "Makefile" or use a template Makefile for existing systems: e.g. "cp Makefile.summit Makefile"
    # Update HDF5_DIR, ABT_DIR and ASYNC_DIR to the correct paths of their installation directory
    # (Optional) update the compiler flag macros: DEBUG, CFLAGS, LIBS, ARFLAGS
    # (Optional) comment/uncomment the correct DYNLIB & LDFLAGS macro
    make
    
Running the automated tests requires Python3.
(Optional) If the system is not using mpirun to launch MPI tasks, edit mpirun_cmd in pytest.py with the corresponding MPI launch command.
    
Run both the serial and parallel tests

    make check

Run the serial tests only

    make check_serial

If any test fails, check async_vol_test.err in the $VOL_DIR/test directory for the error message. 
With certain file systems where file locking is not supported, an error of "file create failed" may occur and can be fixed with "export HDF5_USE_FILE_LOCKING=FALSE", which disables the HDF5 file locking.

## 5. Implicit mode

The implicit mode allows an application to enable asynchronous I/O through setting the following environemental variables and without any major code change. 
By default, the HDF5 metadata operations are executed asynchronously, and the dataset operations are executed synchronously. We recommend to use explicit mode (see next section) for best performance.

    # Set environment variables: HDF5_PLUGIN_PATH and HDF5_VOL_CONNECTOR
    Run your application

## 6. Explicit mode

6.1 Use MPI_THREAD_MULTIPLE

The asynchronous tasks often involve MPI collecive operations from the HDF5 library, and they may be executee concurrently with your application's MPI operations, 
thus we require to initialize MPI with MPI_THREAD_MULTIPLE support. Change MPI_Init(argc, argv) in your application's code to the following:

    MPI_Init_thread(argc, argv, MPI_THREAD_MULTIPLE, &provided);
        
6.2 Use event set and new async API to manage asynchronous I/O operations

    es_id = H5EScreate();                        // Create event set for tracking async operations
    fid = H5Fopen_async(.., es_id);              // Asynchronous, can start immediately
    gid = H5Gopen_async(fid, .., es_id);         // Asynchronous, starts when H5Fopen completes
    did = H5Dopen_async(gid, .., es_id);         // Asynchronous, starts when H5Gopen completes
    status = H5Dwrite_async(did, .., es_id);     // Asynchronous, starts when H5Dopen completes
    status = H5Dread_async(did, .., es_id);      // Asynchronous, starts when H5Dwrite completes
    H5ESwait(es_id, H5ES_WAIT_FOREVER, &num_in_progress, &op_failed); 
    # Wait for operations in event set to complete, buffers used for H5Dwrite must only be changed after
    H5ESclose(es_id);                            // Close the event set (must wait first)

6.3 Error handling with event set

    H5ESget_err_status(es_id, &es_err_status);   // Check if event set has failed operations
    H5ESget_err_count(es_id, &es_err_count);     // Retrieve the number of failed operations in this event set
    H5ESget_err_info(es_id, 1, &err_info, &es_err_cleared);   // Retrieve information about failed operations 
    H5free_memory(err_info.api_name);
    H5free_memory(...)

6.4 Run with async

    # Set environment variables: HDF5_PLUGIN_PATH and HDF5_VOL_CONNECTOR
    Run your application

## Best Practices
By default, async VOL detects whether the application is busy issuing HDF5 I/O calls or has moved on to computation. If it finds no HDF5 function is called within a short wait period (600 ms by default), it will start executing the asynchronous tasks. However, some applications may have larger time gaps between HDF5 calls than the default wait period, which could lead to effectively synchronous I/O. To avoid this, one can set the following environment variable to disable the "wait and check" mechnism and inform async VOL when to start the async execution, this is especially useful when writing checkpointing data, e.g. [AMReX HDF5 output](https://github.com/AMReX-Codes/amrex/tree/development/Tests/HDF5Benchmark).

```
// Start async execution at file close time
export HDF5_ASYNC_EXE_FCLOSE=1
// Start async execution at group close time
export HDF5_ASYNC_EXE_GCLOSE=1
// Start async execution at dataset close time
export HDF5_ASYNC_EXE_DCLOSE=1
```

## Limitation
Async VOL has additional overhead due to its internal management of asynchronous tasks and the background thread execution. If the application is metadata-intensive, e.g. create thousands of groups, datasets, or attributes, this overhead (~0.001s per operation) becomes comparable to the creation time, and could result in worse performance. There may also be additional overhead due to the ``wait and check'' mechnism (mentioned above) unless HDF5_ASYNC_EXE_* is set.

## Know Issues
When an application has a large number of HDF5 function calls, an error like the following may occur:

    *** Process received signal ***
    Signal: Segmentation fault: 11 (11)
    Signal code: (0)
    Failing at address: 0x0
    [ 0] 0 libsystem_platform.dylib 0x00007fff20428d7d _sigtramp + 29
    [ 1] 0 ??? 0x0000000000000000 0x0 + 0
    [ 2] 0 libabt.1.dylib 0x0000000105bdbdc0 ABT_thread_create + 128
    [ 3] 0 libh5async.dylib 0x00000001064bde1f push_task_to_abt_pool + 559
   
This is due to the default Argobots thread stack size being too small (16384), and can be resovled by setting the environement variable:

    export ABT_THREAD_STACKSIZE=100000

Setting the above environment variable could also fix an issue when an async application hangs.

When an application calls H5Dget_space_async, and uses the dataspace ID immediately, a deadlock may occur occationally. We have forced synchronous execution for H5Dget_space_async, to re-enable its asynchronous execution, set the following environement variable:

    export HDF5_ASYNC_DISABLE_DSET_GET=0
