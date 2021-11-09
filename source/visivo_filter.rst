VisIVO Filter
=============
VisIVO Filter is a component that converts data from a VisIVO Binary Table (VBT) into a new one or can create a Volume from a table.

To get a general help:

.. code-block:: console

    $ VisIVOFilter --help

To get a specific operation help:

.. code-block:: console

    $ VisIVOFilter --op <operation> --help

To run the operation:

.. code-block:: console

    $ VisIVOFilter --op <operation> <parameters> [--file] InputFile

.. note:: The InputFile must be a valid VBT. :file:`InputFile.bin` and :file:`InputFile.bin.head` must exist.


Global options
--------------
The following option can be given on any filter operation.

--memsizelimit <percentage>
    This option reduces the memory request of the percentage value given with this option.
    
    Depending on the specific filter request and on the system where VisIVOFilter runs, the allocated memory could exceed the available size and the application could be aborted or can use a significant portion of the system swap area, with a dramatic lost of performance.
    
    This parameter can be given to reduce the allocated space avoiding this effect.

    A Warning message will be given when this option is used. The allowed value is a float greater than 0. and lower than 95.0.
--history
    Create an XML file which contains the history of operations performed.
--historyfile
    Change output history file name. Default: :file:`hist.xml`.


Parameter file
--------------
Alternatively, to run the operation with options specified in the parameter file:

.. code-block:: console

    $ VisIVOFilter <parameterFile>

Lines starting with # are comments.

An example of this file is the following (for the randomizer filter):

.. code-block::

    op=randomizer
    #memsizelimit=30        This is a commented line
    perc=50.0
    iseed=1
    out=VBT_rand.bin
    file=VBT.bin


Operations
----------
The following operations are available:

.. contents::
    :local:


Randomizer
^^^^^^^^^^
Create a random subset from the original data table.

Usage:

.. code-block:: console

    $ VisIVOFilters --op randomizer --perc percentage [--field parameters] [--iseed iseed] [--out filename_out.bin] [--file] inputFile.bin


Options:

--perc
    Percentage (from 0.0 to 100.0) of the input file obtained in the output file.
--field
    Valid columns names of the input table. Default: all columns are included.
--iseed
    Specify the seed for the random generation. Default value 0.
--out
    Output table filename. Default name is given.
--file
    Input table filename.



Select Columns
^^^^^^^^^^^^^^
TBW.


Merge
^^^^^
TBW.


Append
^^^^^^
TBW.


Select Fields
^^^^^^^^^^^^^
TBW.


Math Operations
^^^^^^^^^^^^^^^
TBW.


Decimator
^^^^^^^^^
TBW.
