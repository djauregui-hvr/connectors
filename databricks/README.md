# hvrdatabricksagent.py - HVR Databricks connector
## Description
This python script can be used to merge CDC changes and/or refresh a Databricks Delta table.  The script uses
ODBC to connect to Databricks and apply the changes.  Databricks hosted on Azure and AWS are supported.   

To support this connector the target location is Azure Blob storage, Azure ADLS Gen2, or an S3 bucket.  A FileFormat
action is configured for Csv with /HeaderLine (default), or Avro, Parquet, Json.  The Databricks cluster that is the 
target for this connector must be configured with access to the Azure storage or S3 bucket.

Integrate/Refresh writes table changes to the integrate location (Blob/ADLS store or S3 bucket) as files in the
format defined by the FileFormat action.  This connector connects to the cluster via ODBC and using SQL
statements loads and/or merges the data from the integrate location into the target table.

The following options are available with this connector:
- CDC replication
- Soft Delete
- Timekey
- Create/recreate target table during refresh
  - Create the target as a managed (default) or unmanaged table

## Requirements
- Integrate running on a Windows or Linux machine
- The Simba Spark ODBC driver from Databricks downloaded and installed on the Integrate machine
- If Linux:
  - unixODBC downloaded and installed
- Python pyodbc package downloaded and installed
- A DSN configured to connect to Databricks (see Databricks documentation)
- For Databricks hosted by Azure, using Blob storage:
  - Python azure-storage-blob package downloaded and installed
  - An Azure Blob Storage account & container - Integrate will land files here
  - The Databricks cluster configured to be able to access the Azure Blob Storage
- For Databricks hosted by Azure, using ADLS Gen2:
  - Python azure-storage-file-datalake package downloaded and installed
  - An Azure ADLS Gen2 account & container - Integrate will land files here
  - The Databricks cluster configured to be able to access the ADLS Gen2 storage 
  - An Access Key (regardless of authentication method) so the connector can manage the files
- For Databricks hosted by AWS:
  - Python boto3 package downloaded and installed
  - An S3 bucket - Integrate will land files here
  - The Databricks cluster configured to be able to access the S3 bucket
- The Integrate action defined with /ReorderRows=SORT_COALESCE
- A FileFormat action with one of the following (the default is CSV)
  - /Csv /HeaderLine
  - /Avro
  - /Json
  - /Parquet

## Environment variables
The following environment variables are required by the the agent or optionally influence the behavior of
the connector. They should be supplied by channel Environment actions.

| Environment variable       | Mandatory | Description |
| --------------------       | --------- | ----------- |
| HVR_DBRK_DSN               |    Yes    | The DSN configured on the Integrate machine to connect to Databricks   |
| HVR_DBRK_FILESTORE_ID      | AWS - Yes | The AWS Access Key for connecting to the target location |
| HVR_DBRK_FILESTORE_KEY     |    Yes    | The Azure Access Key or the AWS Secret Key for connecting to the target location |
| HVR_DBRK_FILESTORE_REGION  | AWS - No  | The region where the S3 bucket resides - for connecting to the target location |
| HVR_DBRK_CONNECT_STRING    |     No    | Replaces HVR_DBRK_DSN if desired |
| HVR_DBRK_CONNECT_TIMEOUT   |     No    | If set, the connection to the cluster will have the specified timeout (seconds) |
| HVR_DBRK_DATABASE          |     No    | Specifies the target database, if not the default |
| HVR_DBRK_TIMEKEY           |     No    | Set to 'ON' if the target table is Timekey  |
| HVR_DBRK_FILEFORMAT        |     No    | The file format configured in the FileFormat action if not CSV |
| HVR_DBRK_FILE_EXPR         |     No    | The value of Integrate /RenameExpression if set |
| HVR_DBRK_DELIMITER         |     No    | The FieldSeparator in the FileFormat /Csv action if set |
| HVR_DBRK_LINE_SEPARATOR    |     No    | The LineSeparator in the FileFormat /Csv action if set |
| HVR_DBRK_EXTERNAL_LOC      |     No    | The location for the unmanaged target table if create-on-refresh is configured |
| HVR_DBRK_UNMANAGED_BURST   |     No    | Create the burst table unmanaged ('ON'), or managed ('OFF') |
| HVR_DBRK_LOAD_BURST_DELAY  |     No    | Delay, in seconds, after creating the burst table and before loading |
| HVR_DBRK_HVRCONNECT        |     No    | Provides the credentials for the script to connect to the repository |
| HVR_DBRK_TBLPROPERTIES     |     No    | If set, the connector will set these table properties during refresh |

## UserArgument options
The following options may be set in the /UserArguments of the AgentPlugin action for hvrdatabricksagent.py

| Option | Description |
| ------ | ----------- |
|   -c   | The context used to refresh.  Set this option if '-r' is set and a Context is used int he refresh |
|   -d   | Name of the SoftDelete column.  Default is 'is_deleted'.  Set this option if the SoftDelete column is configured <br>with a name other than 'is_deleted'. |
|   -D   | Name of the SoftDelete column.  Set this option if the SoftDelete column is in the target table. |
|   -o   | Name of the {hvr_op} column.  Default is ‘op_type’.  Set this option if the name of the Extra column populated by <br>{hvr_op} is different than ‘op_type’. |
|   -O   | Name of the {hvr_op} column.   Set this option if the target table includes the Extra column. |
|   -p   | Set this option on a refresh of a TimeKey target if it is desired that the target is not truncated before the refresh. |
|   -r   | Set this option to instruct the script to create/recreate the target table during Refresh |
|   -t   | Set this option if the target is TimeKey.   Same as using the Environment action HVR_DBRK_TIMEKEY=ON |
|   -y   | If set the script will NOT delete the files on S3 or Azure store after they have been applied to the target |

## File Format
By default the script works with CSV files that have a header line.  Since there is no schema component to a CSV file, and 
since all values are written as strings, the datatypes are defined by the data type of the table being loaded and as long 
as the values in the CSV file correspond, there are no conversion issues.

However, it is not that uncommon to find the default field separator, the comma, in the data and this causes parsing issues.
If, because of the characters in the data, the FileFormat action is modified to set the LineSeparator and/or FieldSeparator 
character, there are Environment actions to communicate these settings to the connector:
      HVR_DBRK_DELIMITER
      HVR_DBRK_LINE_SEPARATOR

The connector also supports PARQUET, AVRO, and JSON files.   If the FileFormat action is set to one of these, use 
HVR_DBRK_FILEFORMAT to communicate that setting to the connector.   

If the FileFormat is AVRO or PARQUET, note that PARQUET and AVRO files have schema definitions in them.   These can a data 
type inconsistency error to be thrown.   For instance, HVR sets the precision of an Oracle NUMBER column to 1000 if the DDL 
for the column does not specify a precision.   When DataBricks loads data from one of these files (with precision of at least 
one column’s datatype set to 1000), Databricks will throw an error.   To prevent the error a ColumnProperties action should 
be added to define a precision lower than or equal to 38.

If the FileFormat is JSON, the JsonMode should be set to RowFragments.

## Names of files
The connector associates the files produced by integrate or refresh with the different tables by parsing the file
name.  If the Integrate RenameExpression is set, the value of RenameExpression needs to be passed to the connector in
HVR_DBRK_FILE_EXPR.

## Burst table
The script tries to create the burst table as an unmanaged table. That is, with syntax such as:

    CREATE TABLE <tablename>__bur (<column list>) USING <fileformat> LOCATION <path to files>

where the location is a path to the files written by the current integrate cycle.   The alternative method of creating 
and loading the burst table is more time consuming:

    CREATE TABLE <tablename>__bur USING DELTA AS SELECT * FROM <tablename> WHERE 1=0
    ALTER TABLE <tablename>__bur ADD COLUMN …. (to add the op_type and is_deleted columns if needed)
    COPY INTO <tablename>__bur FROM (SELECT <columnlist> FROM <file list>)

To create the burst table as an unmanaged table, the script must be able to point to a location that has only those 
files written by integrate in this cycle for this table.  The script checks this by 1) checking if the table name is 
in the path, and 2) checking that the files in that location are only the ones that it expects to find there.   If 
these conditions hold, the connector will create the burst table as an unmanaged table.  If not it will create the 
burst table, alter it, and load it.

An Environment action, HVR_DBRK_UNMANAGED_BURST, can be used to force this logic one way or the other.  However, if 
HVR_DBRK_UNMANAGED_BURST=ON, and there are more files in the location than in the integrate cycle for the table, the 
script will revert to the managed table logic.

## Target is a replication copy
To maintain a replication copy, the connector requires that the following ColumnProperties actions are defined. Note 
that the connector will not apply these columns to the Databricks target table.

    ColumnProperties /Name=op_type /Extra /IntegrateExpression={hvr_op} /Datatype=integer /Context=!refresh
    ColumnProperties /Name=is_deleted /Extra /SoftDelete /Datatype=integer /Context=!refresh

A simple example of the configuration for a replicate copy follows:

    AgentPlugin /Command=hvrdatabricksagent.py
    ColumnProperties /Name=op_type /Extra /IntegrateExpression={hvr_op} /Datatype=integer /Context=!refresh
    ColumnProperties /Name=is_deleted /Extra /SoftDelete /Datatype=integer /Context=!refresh
    Environment /Name=HVR_DBRK_FILESTORE_KEY /Value=“ldkfjljfdgljdfgljdflgj”
    Environment /Name=HVR_DBRK_DSN /Value=azuredbrk
    FileFormat /Csv /HeaderLine
    Integrate /ReorderRows=SORT_COALESCE

## Target is TimeKey
If the connector is configured for a TimeKey target (HVR_DBRK_TIMEKEY=ON or AgentPlugin /UserArguments="-t"),
then the connector will load all the changes in the files directly to the target table.  Any /Extra column defined 
for the target exist in the target table.

## Target is SoftDelete
If the target is SoftDelete, as opposed to a replicate copy, the connector must be configured to preserve the
SoftDelete column.  The SoftDelete column can have any name.  To indicate to the connector the name of the 
SoftDelete column, and that it should be preserved, use the "-D" option in the AgentPlugin /UserArguments.
A sample configuration for SoftDelete follows:

    AgentPlugin /Command=hvrdatabricksagent.py /UserArgument="-D is_deleted"
    ColumnProperties /Name=op_type /Extra /IntegrateExpression={hvr_op} /Datatype=integer /Context=!refresh
    ColumnProperties /Name=is_deleted /Extra /SoftDelete /Datatype=integer
    Environment /Name=HVR_DBRK_FILESTORE_KEY /Value=“jkhkgkjhkjh="
    Environment /Name=HVR_DBRK_DSN /Value=azuredbrk
    FileFormat /Csv /HeaderLine
    Integrate /ReorderRows=SORT_COALESCE

## Refresh Create/Recreate target table
The connector can be configured to create/recreate the target table when refresh is run.  The requirements are:
- Integrate runs on the hub
- The connector is running under Python 3
- The AgentPlugin action that defines the Databricks connector has “-r” in the UserArguments. For example:  
       AgentPlugin /Command=hvrdatabricksagent.py /UserArgument=”-r” /Context=”refresh”
- The repository connection string is provided to the Agent Plugin via the HVR_DBRK_HVRCONNECT Environment action.

To get the value for the HVR_DBRK_HVRCONNECT Environment action:
1. In the GUI run Initialize to get the connection string.  For example, my MySQL hub's connection is:  '-uhvr/!{6mATRwyh}!' -h mysql '142.171.34.118~3308~mysql'
2. Convert to base64 in any web converter.  

The script can create the target table as a managed, or an unmanged table.   By default the table is created managed.   
To create an unmanaged table, specify the location of the table using the HVR_DBRK_EXTERNAL_LOC environment action.  
Note that the pathname specified by HVR_DBRK_EXTERNAL_LOC may contain {hvr_tbl_name} and, if it does, the script will 
perform the substitution.  For example:
       /Name=HVR_DBRK_EXTERNAL_LOC /Value="/mnt/delta/{hvr_tbl_name}"

If there are ColumnProperties actions tied to a Context, and that Context is used with the refresh, the "-c" Context should be passed to the connector using the "-c" option.

## Sliced Refresh
If the environment variable HVR_DBRK_SLICE_REFRESH_ID is set on a refresh, the connector will use locking and control functionality to ensure that the target table is truncated or created only at the beginning, and that only one slice job at a time accesses the target table.  This logic is in conjunction with the hvrslicedrefresh.py script.

## Table Properties
By default the connector will set the following properties on the table when it is refreshed:

    delta.autoOptimize.optimizeWrite = true, delta.autoOptimize.autoCompact = true

If the Environment variable HVR_DBRK_TBLPROPERTIES is set, the connector will replace the default table properties
with the value in the Environment variable.  If HVR_DBRK_TBLPROPERTIES is set to '' or "", the connector will
not set table properties during refresh.

## Change Log
| Version | Date     | Description |
| ------- | -------- | ----------- |
| 1.0     | 06/18/21 | First version - includes all changes up to this time as defined in the change log in hvrdatabricksagent.py |
| 1.1     | 06/30/21 | Fix table_file_name_map for non-default /RenameExpression |
| 1.2     | 07/01/21 | Escape quote all column names to support column name like class# |
| 1.3     | 07/02/21 | Issue plutiple COPY INTO commands if # files > 1000 |
| 1.4     | 07/09/21 | Fixed a bug in create table processing ColumnProperties DatatypeMatch where it would only apply to <br>first column that matched |
| 1.5     | 07/09/21 | Fixed create table column ordering - respect source column order |
| 1.6     | 07/09/21 | Provide an Environment variable for customizing table properties |
| 1.7     | 07/14/21 | Added support for /DatatypeMatch="number[prec=0 && scale=0]" only |


