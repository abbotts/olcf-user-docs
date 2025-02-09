
************************
Installing Parallel h5py
************************

This guide has been shortened and adapted from a challenge in OLCF's `Hands-On with Summit <https://github.com/olcf/hands-on-with-summit>`__ GitHub repository (`Python: Parallel HDF5 <https://github.com/olcf/hands-on-with-summit/tree/master/challenges/Python_Parallel_HDF5>`__).

.. warning::
   The guide is designed to be followed from start to finish, as certain steps must be completed in the correct order before some commands work properly.

Overview
========

This guide teaches you how to build a personal, parallel-enabled version of h5py and how to write an HDF5 file in parallel using mpi4py and h5py.
Although Summit is being used in this guide, all of the concepts still apply to other OLCF systems.

In this guide, you will: 

* Learn how to install mpi4py on Summit
* Learn how to install parallel h5py on Summit
* Test your build with Python scripts

Parallel HDF5
=============

Scientific simulations generate large amounts of data on Summit (about 100 Terabytes per day for some applications).
Because of how large some datafiles may be, it is important that writing and reading these files is done as fast as possible.
Less time spent doing input/output (I/O) leaves more time for advancing a simulation or analyzing data.

One of the most utilized file types is the Hierarchical Data Format (HDF), specifically the HDF5 format.
`HDF5 <https://www.hdfgroup.org/solutions/hdf5/>`__ is designed to manage large amounts of data and is built for fast I/O processing and storage.
An HDF5 file is a container for two kinds of objects: "datasets", which are array-like collections of data, and "groups", which are folder-like containers that hold datasets and other groups.

There are various tools that allow users to interact with HDF5 data, but we will be focusing on `h5py <https://docs.h5py.org/en/stable/>`__ -- a Python interface to the HDF5 library.
h5py provides a simple interface to exploring and manipulating HDF5 data as if they were Python dictionaries or NumPy arrays.
For example, you can extract specific variables through slicing, manipulate the shapes of datasets, and even write completely new datasets from external NumPy arrays.

Both HDF5 and h5py can be compiled with MPI support, which allows you to optimize your HDF5 I/O in parallel.
MPI support in Python is accomplished through the `mpi4py <https://mpi4py.readthedocs.io/en/stable/>`__ package, which provides complete Python bindings for MPI.
Building h5py against mpi4py allows you to write to an HDF5 file using multiple parallel processes, which can be helpful for users handling large datasets in Python.
h5Py is available after loading the default Python module on either Summit or Andes, but it has not been built with parallel support.

Setting up the environment
==========================

.. warning::
   Before setting up your conda environment, you must exit and log back in to Summit so that you have a fresh login shell.
   This is to ensure that no previously activated conda environments exist in your ``$PATH`` environment variable.

Building h5py from source is highly sensitive to the current environment variables set in your profile.
Because of this, it is extremely important that all the modules and conda environments you plan to load are done in the correct order, so that all the environment variables are set correctly.
First, unload all the current modules that you may have automatically loaded on Summit and then immediately load the default modules.

.. code-block:: bash

   $ module purge
   $ module load DefApps

Next, load the gnu compiler module (most Python packages assume GCC), hdf5 module (necessary for h5py), and the python module (allows you to create a new conda environment):

.. code-block:: bash

   $ module load gcc
   $ module load hdf5
   $ module load python

Loading the python module puts you in a "base" conda environment, but you need to create a new environment using the ``conda create`` command:

.. code-block:: bash

   $ conda create -p /ccs/proj/<project_id>/<user_id>/conda_envs/summit/h5pympi-summit python=3.8

.. note::
   As noted in the :doc:`/software/python/index` page, it is highly recommended to create new environments in the "Project Home" directory.

After following the prompts for creating your new environment, you can now activate it:

.. code-block:: bash

   $ source activate /ccs/proj/<project_id>/<user_id>/conda_envs/summit/h5pympi-summit

Installing mpi4py
=================

Now that you have a fresh conda environment, you will next install mpi4py from source into your new environment.
To make sure that you are building from source, and not a pre-compiled binary, use pip:

.. code-block:: bash

   $ MPICC="mpicc -shared" pip install --no-binary=mpi4py mpi4py

The ``MPICC`` flag ensures that you are using the correct C wrapper for MPI on the system.
Building from source typically takes longer than a simple ``conda install``, so the download and installation may take a couple minutes.
If everything goes well, you should see a "Successfully installed mpi4py" message.

Installing h5py
===============

Next, install h5py from source.
Because h5py depends on NumPy, install an optimized version of the NumPy package using ``conda install``:

.. code-block:: bash

   $ conda install -c defaults --override-channels numpy

The ``-c defaults --override-channels`` flags ensure that conda will search for NumPy only on the "defaults" channel.
Installing NumPy in this manner results in an optimized NumPy that is built against linear algebra libraries, which performs operations much faster.

Next, you are finally ready to install h5py from source:

.. code-block:: bash

   $ HDF5_MPI="ON" CC=mpicc pip install --no-binary=h5py h5py

The ``HDF5_MPI`` flag is the key to telling pip to build h5py with parallel support, while the ``CC`` flag makes sure that you are using the correct C wrapper for MPI.
This installation will take much longer than both the mpi4py and NumPy installations (5+ minutes if the system is slow).
When the installation finishes, you will see a "Successfully installed h5py" message.

Testing parallel h5py
=====================

Test your build by trying to write an HDF5 file in parallel using 42 MPI tasks.

First, change directories to your GPFS scratch area:

.. code-block:: bash

   $ cd $MEMBERWORK/<YOUR_PROJECT_ID>
   $ mkdir h5py_test
   $ cd h5py_test

Let's test that mpi4py is working properly first by executing the example Python script "hello_mpi.py":

.. code-block:: python

   # hello_mpi.py
   from mpi4py import MPI

   comm = MPI.COMM_WORLD      # Use the world communicator
   mpi_rank = comm.Get_rank() # The process ID (integer 0-41 for a 42-process job)

   print('Hello from MPI rank %s !' %(mpi_rank))

To do so, submit a job to the batch queue with "submit_hello.lsf":

.. code-block:: bash

   $ bsub -L $SHELL submit_hello.lsf

Example "submit_hello.lsf" batch script:

.. code-block:: bash

   #!/bin/bash
   #BSUB -P <PROJECT_ID>
   #BSUB -W 00:05
   #BSUB -nnodes 1
   #BSUB -J mpi4py
   #BSUB -o mpi4py.%J.out
   #BSUB -e mpi4py.%J.err

   cd $LSB_OUTDIR
   date

   module load gcc
   module load hdf5
   module load python

   source activate /ccs/proj/<project_id>/<user_id>/conda_envs/summit/h5pympi-summit

   jsrun -n1 -r1 -a42 -c42 python3 hello_mpi.py

If mpi4py is working properly, in ``mpi4py.<JOB_ID>.out`` you should see output similar to:

.. code-block::

   Hello from MPI rank 21 !
   Hello from MPI rank 23 !
   Hello from MPI rank 28 !
   Hello from MPI rank 40 !
   Hello from MPI rank 0 !
   Hello from MPI rank 1 !
   Hello from MPI rank 32 !
   .
   .
   .

If you see this, great, it means that mpi4py was built successfully in your environment.

Finally, let's see if you can get these tasks to write to an HDF5 file in parallel using the "hdf5_parallel.py" script:

.. code-block:: python

   # hdf5_parallel.py
   from mpi4py import MPI
   import h5py

   comm = MPI.COMM_WORLD      # Use the world communicator
   mpi_rank = comm.Get_rank() # The process ID (integer 0-41 for a 42-process job)
   mpi_size = comm.Get_size() # Total amount of ranks

   with h5py.File('output.h5', 'w', driver='mpio', comm=MPI.COMM_WORLD) as f:
       dset = f.create_dataset('test', (42,), dtype='i')
       dset[mpi_rank] = mpi_rank

   comm.Barrier()

   if (mpi_rank == 0):
       print('42 MPI ranks have finished writing!')

The MPI tasks are going to write to a file named "output.h5", which contains a dataset called "test" that is of size 42 (assigned to the "dset" variable in Python).
Each MPI task is going to assign their rank value to the "dset" array in Python, so you should end up with a dataset that contains 0-41 in ascending order.

Time to execute "hdf5_parallel.py" by submitting "submit_h5py.lsf" to the batch queue:

.. code-block:: bash

   $ bsub -L $SHELL submit_h5py.lsf

Example "submit_h5py.lsf" batch script:

.. code-block:: bash

   #!/bin/bash
   #BSUB -P <PROJECT_ID>
   #BSUB -W 00:05
   #BSUB -nnodes 1
   #BSUB -J h5py
   #BSUB -o h5py.%J.out
   #BSUB -e h5py.%J.err

   cd $LSB_OUTDIR
   date

   module load gcc
   module load hdf5
   module load python

   source activate /ccs/proj/<project_id>/<user_id>/conda_envs/summit/h5pympi-summit

   jsrun -n1 -r1 -a42 -c42 python3 hdf5_parallel.py

Provided there are no errors, you should see "42 MPI ranks have finished writing!" in the ``h5py.<JOB_ID>.out`` output file, and there should be a new file called "output.h5" in your directory.
To see explicitly that the MPI tasks did their job, you can use the ``h5dump`` command to view the dataset named "test" in output.h5:

.. code-block:: bash

   $ h5dump output.h5

   HDF5 "output.h5" {
   GROUP "/" {
      DATASET "test" {
         DATATYPE  H5T_STD_I32LE
         DATASPACE  SIMPLE { ( 42 ) / ( 42 ) }
         DATA {
         (0): 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18,
         (19): 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34,
         (35): 35, 36, 37, 38, 39, 40, 41
         }
      }
   }
   }

If you see the above output, then the build was a success!

Additional Resources
====================

* `h5py Documentation <https://docs.h5py.org/en/stable/>`__
* `mpi4py Documentation <https://mpi4py.readthedocs.io/en/stable/>`__
* `HDF5 Support Page <https://portal.hdfgroup.org/display/HDF5/HDF5>`__
