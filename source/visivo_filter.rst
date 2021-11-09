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
--historyfile <file.xml>
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
This operation creates a random subset from the original data table.

Usage:

.. code-block:: console

    $ VisIVOFilter --op randomizer --perc percentage [--field parameters] [--iseed iseed] [--out filename_out.bin] [--file] inputFile.bin

Options:

--perc
    Percentage (from 0.0 to 100.0) of the input file obtained in the output file.

    .. note:: Only the first decimal place is considered.

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
This operation creates e new table using (or excluding) one or more fields of a data table. The default case produces the output table including only listed fields.

Usage:

.. code-block:: console

    $ VisIVOFilter --op selcolumns --field parameters [--delete] [--out filename_out.bin] [--file] inputFile.bin

Options:

--field
    Valid columns names of the input table. Default: all columns are included.
--delete
    Produce output table excluding only field listed in the --field option.
--out
    Output table filename. Default name is given.
--file
    Input table filename.


Merge
^^^^^
This operation creates a new table from two or more existing data tables. Up to 100 tables can be merged. Volumes can be merged but they must have the same geometry.

Usage:

.. code-block:: console

    $ VisIVOFilter --op merge  [--size HUGE/SMALLEST] [--pad value] [--out filename_out.bin] [--filelist] tab_selection_file.txt

Options:

--size
    Produce a new table having the size of the smallest (or larger) table. Default option: SMALLEST.
--pad
    Pad the table rows of smaller table with the given value if HUGE size is used. Default value: 0.
--out
    Output table filename. Default name is given.
--filelist
    Input filename containing the table list.

The :file:`tab_selection_file.txt` is a file that contain a list of tables and valid columns. Wildcard "*" means all the columns of the given table. An example file is the following:

.. code-block::

    tab1.bin Col_1
    tab1.bin Col_2
    tab5.bin Col_x
    tab4.bin *

This file a new table having columns Col_1 and Col_2 from :file:`tab1.bin`, Col_x from :file:`tab5.bin` and all the columns of :file:`tab4.bin`.


Append
^^^^^^
This operation creates a new table appending data from a list of existing tables. Append Filter can append up to 100 tables with the same number of Columns.

Usage:

.. code-block:: console

    $ VisIVOFilter --op append [--out filename_out.bin] [--filelist] table_list.txt

Options:

--out
    Output table filename. Default name is given.
--filelist
    Input filename containing the table list.

:file:`table_list.txt` is a file that contains a list of valid table names. The ".bin" extension is automatically added if the listed filename does not contain it.

.. code-block::

    tab1
    tab2
    tab3

.. note:: The column names are copied from the first table. An error is given if tables contain different numbers of columns.


Select Fields
^^^^^^^^^^^^^
This operation creates a new table setting limits on one or more fields of a data table. Optionally it creates a list of elements satisfying the requested condition.

Usage:

.. code-block:: console

    $ VisIVOFilter --op selfield  --limits limitsfile.txt [--operator AND/OR] [--outlist list_filename] [--format uns/int/ascii] [--out filename_out.bin] [--file] inputFile.bin

Options:

--limits
    A file that has three columns: a valid column name and an interval indicating the extraction limits.
--operator
    Limits on all fields listed in --limits option file are combined by default with logic AND operator. If this option is given with OR value the field limits are combined with logic OR operator.
--outlist
    Output list filename containing the number of the elements satisfying the requested condition. Default name is given.
--format
    Data format in the outlist filename. Default value unsigned long long int.
--out
    Output table filename. Default name is given.
--file
    Input table filename.

The :file:`limitsfile.txt` file must have the following structure. A valid column name and an interval indicating the extraction limits:

.. code-block::

    X 20.0 30.0
    Y 10.0 20.0
    Z 0.0 10.0

This file produces a new table that contains all the data points of the input table (all columns will be reported) where :math:`X \in [20.0, 30.0]` AND :math:`Y \in [10.0, 20.0]` AND :math:`Z \in [0.0, 10.0]`.

.. note:: The unlimited word can be used to indicate the infinite value.


Math Operations
^^^^^^^^^^^^^^^
TBW.


Decimator
^^^^^^^^^
TBW.
