VisIVO Importer
===============
VisIVO Importer converts user-supplied datasets into a VBT. VBTs are used by VisIVO Filter for data processing and VisIVO Viewer for visualization. The conversion is independent of data dimensionality.

General VisIVO Importer Syntax:

.. code-block:: console

    $ VisIVOImporter --flag1 --flag2 ... --flagn FileName

VisIVO Importer General Options:

--fformat
    Specifies the format of input datasets.
--out <name>
    (Optional) Output FileName. The path specified with this flag is the VisIVO Importer output directory.
    
    If no path is specified the VisIVO Importer output directory is the current directory, and the default name for OutputFileName is :file:`VisIVOServerBinary.bin`.
--volume
    (Optional) Used to create volumes.
--compx <value>, --compy <value>, --compz <value>
    (Optional) Used for entering geometry for volumetric datasets.
    
    If data size fits in a cubic mesh, values are computed automatically.
--sizex <value>, --sizey <value>, --sizez <value>
    (Optional) Used for specifying volumetric cell dimensions; the default values are 1.0, 1.0 and 1.0 respectively.
--userpwd <username:password>
    (Optional) Used to prescribe a username and password for accessing remote files.
    
    Ambiguous characters for the shell (i.e. $ >, < etc..) must be given with escape character “\”. For example ``guest$09`` must be given as ``guest\$09``.

    .. note:: Escape characters MUST NOT BE GIVEN using the parameterFile.
--binaryheader <headerfilename>
    (Optional) Used to specify the file name of the header of VBTs; this flag is ignored in case of other formats.
--missingvalue <value>
    (Optional) Used to set the missing data to a fixed value. If not present the default value is: -1.0918273645e+23.
--textvalue <value>
    (Optional) Used to set the textual data to a fixed value. If not present a default value is given: -1.4536271809e+15.
--bigendian
    (Optional) Used only for big endian Gadget and FLY input files, otherwise by default this flag is set to little endian.
--double
    (Optional) Used only for the double data type of FLY input files, otherwise by default this flag is set to float.
--npoints <value>
    (Optional) Used only for FLY input files to specify the number of data points.
--history
    (Optional) Create an XML file which contains the history of operations performed.
--historyfile <filename>
    (Optional) Change output history file name. Default: :file:`hist.xml`.

The FileName is the local (or remote) file containing the data to be converted into a VBT. If FileName starts with ``http://``, ``ftp://`` or ``sftp://`` the remote file is downloaded automatically. However if the --userpwd option is specified, the prescribed username and password are employed for remote access. The sftp syntax requires the remote directory where data are located.

.. note:: sftp is not allowed if VisIVO is compiled with the LIGHT option. The server must have libcurl with ssl support to enable the sftp functionality and VisIVO must be compiled without the LIGHT option.

All downloaded files are copied temporarily into the directory given in --out option and are deleted automatically at the end of the import process. Under the current file directory, the file :file:`DownLoadedFilename_VisIVO_List` contains information on all download operations. If the fformat option is binary VisIVOImporter will attempt to download two remote files, a binary table (given as remote filename) and its associated header file (same name +``.head`` extension).


Parameter file
--------------
Alternatively, to run the importer with options specified in the parameter file:

.. code-block:: console

    $ VisIVOImporter <parameterFile>

Lines starting with # are comments.

Options with one or more parameters must be all specified after the “=” sign in the same line (e.g. field=X Y Z).

Options that do not require parameters must be given with “true” keyword (e.g. double=true, bigendian=true).

The input filename has the keyword file (ex. file=myInputFilename).

Examples of parameter files are the following:

.. code-block::

    fformat=ascii
    out=outFilename.bin
    file=asciinputFile

.. code-block::

    fformat=votablefast
    out=/home/user/dataNewTable.bin
    missingvalue=0.0
    file=myVOTable.xml

.. code-block::

    fformat=fly
    out=FlyData.bin
    double=true
    npoints=1000
    bigendian=true
    userpwd=myusername:mypassword
    file=http://remotehosts.domain.eu/directory/InputDataFile


Formats
-------
The following file formats are supported:

.. contents::
    :local:


ASCII
^^^^^
ASCII files are expected to be in tabular form. An ASCII file may contain values for N variables organized in columns. The columns are typically separated by whitespace characters, e.g. spaces or tabs.

The first row of an ASCII file lists the N variables names explicitly. As an example, the command below produces :file:`NewTable.bin`, :file:`NewTable.bin.head` from :file:`ASCIIUserFileName.txt`.

.. code-block:: console

    $ VisIVOImporter --fformat ascii --out /home/user/data/NewTable ASCIIUserFileName.txt


CSV
^^^
CSV is a delimited data format that has fields/columns separated by the comma character and records/rows separated by newlines.

Fields that contain a special character (such as comma, newline, or double quote) must be enclosed in double quotes. However, if a line contains a single entry that happens to be the empty string, it may be enclosed in double quotes. If a field's value is a double quote character, this is dealt with by placing another double quote character next to it.

The CSV file format does not require a specific character encoding, byte order, or line terminator format. 

As an example, the command below produces the files :file:`NewTable.bin` and :file:`NewTable.bin.head` from the user-supplied CSV file :file:`CSVUserFileName.txt`.

.. code-block:: console

    $ VisIVOImporter --fformat csv --out /home/user/data/NewTable CSVUserFileName.txt

.. note:: The Importer automatically skips all the lines starting with # character. If the first line contains column names starting with #, this character will be removed, and the columns names will be given without it.