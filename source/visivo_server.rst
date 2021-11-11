VisIVO Server
=============
TBW.


VisIVO Binary Table
-------------------
A VisIVO Binary Table (VBT) is a highly-efficient data representation used by VisIVO Server internally. A VBT is realized through a header file (extension ``.bin.head``) containing all necessary metadata, and a raw data file (extension ``.bin``) storing actual data values.

For example, the header may contain information regarding the overall number of fields and number of points for each field (for point datasets) or the number of cells and relevant mesh sizes (for volume datasets). The raw data file is typically a sequence of values, e.g. all X followed by all Y values.


Header
^^^^^^
The header file contains the following fields:

.. code-block:: none

    float | double
    n1
    n2 [ GeoX GeoY GeoZ DX DY DZ ]
    little | big
    X
    Y
    Z
    Vx
    Vy
    Vz

Where:

- float | double
    It is the data type of the storage variables used.
- n1
    It denotes the number of columns (fields) in the VBT (e.g. 6).
- n2
    It denotes the number of rows in the VBT (e.g. 262144).
- GeoX GeoY GeoZ DX DY DZ
    They are employed only if the VBT represents volumetric datasets.
    
    In that case GeoX, GeoY and GeoZ represent the mesh geometry, while DX, DY and DZ represent the x, y and z size of volumetric cells (e.g. 64 64 64 1.0 1.0 1.0).
- little | big
    It denotes the endianism employed in the VBT. After this field there exist n1 rows that indicate the VBT columns as positions (X, Y, Z) and velocities (Vx, Vy, Vz).
