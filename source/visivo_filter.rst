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


Parameter file
--------------
Alternatively, to run the operation with options specified in the parameter file:

.. code-block:: console

    $ VisIVOFilter <parameterFile>


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

    $ VisIVOFilters --op randomizer --perc percentage [--field parameters] [--iseed iseed] [--out filename_out.bin] [--history] [--historyfile filename.xml] [--file] inputFile.bin


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
--history
    (Optional) Create an XML file which contains the history of operations performed.
--historyfile
    (Optional) Change output history file name. Default: :file:`hist.xml`.


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
