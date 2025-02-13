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
  - /Json /RowFragments
  - /Parquet

## Environment variables
The following environment variables are required by the the agent or optionally influence the behavior of
the connector. They should be supplied by channel Environment actions.

| Environment variable       | Mandatory | Description |
| --------------------       | --------- | ----------- |
| HVR_DBRK_DSN               |    Yes    | The DSN configured on the Integrate machine to connect to Databricks   |
| HVR_DBRK_FILESTORE_ID      |   Maybe   | The AWS Access Key for connecting to the target location |
| HVR_DBRK_FILESTORE_KEY     |   Maybe   | The Azure Access Key or the AWS Secret Key for connecting to the target location |
| HVR_DBRK_FILESTORE_REGION  |     No    | The region where the S3 bucket resides - for connecting to the target location |
| HVR_DBRK_CONNECT_STRING    |     No    | Replaces HVR_DBRK_DSN if desired |
| HVR_DBRK_CONNECT_TIMEOUT   |     No    | If set, the connection to the cluster will have the specified timeout (seconds) |
| HVR_DBRK_DATABASE          |     No    | Specifies the target database, if not the default |
| HVR_DBRK_DELIMITER         |     No    | The FieldSeparator in the FileFormat /Csv action if set |
| HVR_DBRK_EXTERNAL_LOC      |     No    | The location for the unmanaged target table if create-on-refresh is configured |
| HVR_DBRK_FILEFORMAT        |     No    | The file format configured in the FileFormat action if not CSV |
| HVR_DBRK_FILESTORE_OPS     |     No    | Control which file operations are executed |
| HVR_DBRK_FILE_EXPR         |     No    | The value of Integrate /RenameExpression if set |
| HVR_DBRK_HVRCONNECT        |     No    | Provides the credentials for the script to connect to the repository |
| HVR_DBRK_LINE_SEPARATOR    |     No    | The LineSeparator in the FileFormat /Csv action if set |
| HVR_DBRK_LOAD_BURST_DELAY  |     No    | Delay, in seconds, after creating the burst table and before loading |
| HVR_DBRK_MULTIDELETE       |     No    | Handle the multi-delete change that is a result of SAPXform |
| HVR_DBRK_PARALLEL          |     No    | Number of parallel processes processing table changes |
| HVR_DBRK_PARTITION_table   |     No    | If set, target table is created with partitions columns |
| HVR_DBRK_SLICE_REFRESH_ID  |     No    | Should be set by hvrslicedrefresh.py.  If set connector runs sliced refresh logic |
| HVR_DBRK_TBLPROPERTIES     |     No    | If set, the connector will set these table properties during refresh |
| HVR_DBRK_TIMEKEY           |     No    | Set to 'ON' if the target table is Timekey  |
| HVR_DBRK_UNMANAGED_BURST   |     No    | Create the burst table unmanaged ('ON'), or managed ('OFF') |

## UserArgument options
The following options may be set in the /UserArguments of the AgentPlugin action for hvrdatabricksagent.py

| Option | Description |
| ------ | ----------- |
|   -c   | The context used to refresh.  Set this option if '-r' is set and a Context is used in the refresh |
|   -d   | Name of the SoftDelete column.  Default is 'is_deleted'.  Set this option if the SoftDelete column is configured <br>with a name other than 'is_deleted'. |
|   -D   | Name of the SoftDelete column.  Set this option if the SoftDelete column is in the target table. |
|   -o   | Name of the {hvr_op} column.  Default is ‘op_type’.  Set this option if the name of the Extra column populated by <br>{hvr_op} is different than ‘op_type’. |
|   -n   | If set, the connector will apply inserts using INSERT SQL instead of MERGE |
|   -O   | Name of the {hvr_op} column.   Set this option if the target table includes the Extra column. |
|   -p   | Set this option on a refresh of a TimeKey target if it is desired that the target is not truncated before the refresh. |
|   -r   | Set this option to instruct the script to create/recreate the target table during Refresh |
|   -t   | Set this option if the target is TimeKey.   Same as using the Environment action HVR_DBRK_TIMEKEY=ON |
|   -w   | Specify files in ADLS G2 to databricks using 'wasbs://' instead of 'abfss://' |
|   -y   | If set the script will NOT delete the files on S3 or Azure store after they have been applied to the target |

## Authentication
The connector needs authentication to connect to the file store and delete the files processed during the cycle.  Set Environment actions to the following values to provide this authorization to the conector.

#### AWS
    HVR_DBRK_FILESTORE_ID - Set to the AWS access key ID
    HVR_DBRK_FILESTORE_KEY - Set to the AWS secret access key
    HVR_DBRK_FILESTORE_REGION - Set tot he region where the S3 bucket is located

#### Azure Blob store
    HVR_DBRK_FILESTORE_KEY - Set to the Azure access key

#### Azure ADLS gen2
The ADLSg2 file store can be authorized either through an access key, or OAuth credentials.  If HVR_DBRK_FILESTORE_KEY is set, the connector will use it to connect to the file store.

    HVR_DBRK_FILESTORE_KEY - Set to the Azure access key

If HVR_DBRK_FILESTORE_KEY is not set, the connector will authenticate with DefaultAzureCredential as described in [How to authorize Python apps on Azure](https://docs.microsoft.com/en-us/azure/developer/python/azure-sdk-authenticate). The DefaultAzureCredential automatically uses the app's managed identity (MSI) in the cloud, and automatically loads a local service principal from environment variables when running locally. The following environment variables should be set in the HVR user's environment when not running in the cloud:

    AZURE_TENANT_ID - Set to the directory (tenant) ID
    AZURE_CLIENT_ID - Set to the application (client) ID
    AZURE_CLIENT_SECRET - Set to the client secret

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

## Files written by HVR and processed by the connector
The connector associates the files produced by integrate or refresh with the different tables by parsing the file
name.  If the Integrate RenameExpression is set, the value of RenameExpression needs to be passed to the connector in
HVR_DBRK_FILE_EXPR.

When issuing the SQL that moves the data from those files to a Delta table, the pathname is generated differently depending on the target.  For AWS the path starts with 's3://'.   For Azure Blob store the path starts with 'wasbs://'.   For ADLS Gen2 the path starts with 'abfss://'.

The connector processes one table at a time.   As soon as the data for a table has been merged, the connector will delete the files for that table.  If the connector is interrupted (this could happen when Integrate is suspended), then when HVR or Integrate start back up, the connector will be called again with the same environment variable settings. For this reason the connector always checks that the files exist before processing a table.   If they do not exist the connector will skip that table as it has already been processed.

The connector also checks to see if the files written by Integrate in the last cycle are the only files in the folder.  If they are, then the connector can create the burst table as an unmanaged table passing the folder as the location.  See 'Burst table' below.

The various file operations performed by the connector can be disabled using HVR_DBRK_FILESTORE_OPS.  Its valid values are:

    'check'
    'delete'
    '+cleanup'
    'none'

If set to 'none' the connector will not check that the files exist, nor will it delete the files after the table has been processed.

If set to '+cleanup' the connector will delete any files it finds in the folder during the 'check' phase that are not part of this integrate cycle.  Note that to enable '+cleanup', HVR_DBRK_FILE_EXPR must be set to the value of Integrate /RenameExpression and the pathname specified by HVR_DBRK_FILE_EXPR must include {hvr_tbl_name} as an element in a folder, not just the file name.

## Burst table
The script creates the burst table as a managed table and then uses COPY INTO to load the burst table.  The connector can also be configured to create the burst table as an unmanaged table.  That is, the create table defines metadata but the data are the files uploaded by Integrate. When the burst table is created as an unmanaged table, the script skips the COPY INTO sql to load the burst table.

To create the burst table as an unmanaged table, the script must be able to point to a location that has only those 
files written by integrate in this cycle for this table.  The script checks this by 1) checking if the table name is 
in the path, and 2) checking that the files in that location are only the ones that it expects to find there.   If 
these conditions hold, the connector can create the burst table as an unmanaged table.  If not it will create the 
burst table and load it.

By default, the script will create the burst table as an unmanaged table if it can (that is, if the above conditions hold), otherwise the script will create the burst table as a managed table.

An Environment action, HVR_DBRK_UNMANAGED_BURST, can be used to force this logic one way or the other.  However, if 
HVR_DBRK_UNMANAGED_BURST=ON, and there are more files in the location than in the integrate cycle for the table, the 
script will revert to the managed table logic.

## Target is a replication copy
To maintain a replication copy, the connector requires that the following ColumnProperties actions are defined. Note 
that the connector will not apply these columns to the Databricks target table.
:w

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
- The '-r' option is set in the AgentPlugin /UserArgument
- The repository connection string is provided to the Agent Plugin via the HVR_DBRK_HVRCONNECT Environment action.

To get the value for the HVR_DBRK_HVRCONNECT Environment action:
1. In the GUI run Initialize to get the connection string.  For example, my MySQL hub's connection is:  '-uhvr/!{6mATRwyh}!' -h mysql '142.171.34.118~3308~mysql'
2. Convert to base64 in any web converter.  

The connector retrieves the table description and all the ColumnProperites actions so that it can generate the same column description as HVR would.  If there are Contexts on the ColumnProperties actions, and if the Context is used for a refresh, the '-c <context>' option should be added to the UserArguments so that the connector knows which ColumnProperties actions to apply.

The script can create the target table as a managed, or an unmanaged table.   By default the table is created managed.
To create an unmanaged table, specify the location of the table using the HVR_DBRK_EXTERNAL_LOC environment action.  
Note that the pathname specified by HVR_DBRK_EXTERNAL_LOC may contain {hvr_tbl_name} and, if it does, the script will 
perform the substitution.  For example:

       /Name=HVR_DBRK_EXTERNAL_LOC /Value="/mnt/delta/{hvr_tbl_name}"

If there are ColumnProperties actions tied to a Context, and that Context is used with the refresh, the "-c" Context should be passed to the connector using the "-c" option.

The table can be configured so that partitioning is defined upon create with HVR_DBRK_PARTITION_table.   Set "table" to the HVR table name and set the Value of the Environment action to a comma separated list of columns.  For example:

       /Name=HVR_DBRK_PARTITION_kc4col /Value=c2,c1

## A note on wildcards
The connector will associate a ColumnProperties action with a table if ColumnProperties /Table is defined with 1) the actual table name, 2) a wildcard matching all tablenames ('\*'), 3) a specification matching the beginning of the name ('asdf\\*'), or 4) a specification matching the end of a name ('\*asdf').   The connector cannot match any other wildcarded name specification.   This is also true for the "table" used in HVR_DBRK_PARTITION_table.  The Environment Variable can be used to define partitioning where "table" is the actual table name, "table" = '\*', "table" = 'asdf\*', or "table" = '\*asdf'.

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
| 1.8     | 07/20/21 | Added support for sliced refresh when generated by hvrslicedrefresh.py -s' |
| 1.9     | 07/21/21 | Add UserArguments option to pass in the refresh Context - used to filter the ColumnProperties actions |
| 1.10    | 07/22/21 | Use 'wasbs://' instead of 'abfss://' when passing file path into databricks |
| 1.11    | 07/23/21 | Fixed throwing "F_JX0D03: list assignment index out of range" checking Python version |
| 1.12    | 07/23/21 | Use OAuth authentication to list and access files in ADLS gen 2 - use access key if set |
| 1.13    | 07/27/21 | Fixed throwing "F_JX0D03: list assignment index out of range" processing target columns |
| 1.14    | 07/27/21 | Fixed throwing 'F_JX0D03: delete_file() takes 2 positional arguments but # were given' |
| 1.15    | 07/28/21 | Process ColumnProperties and TableProperties where chn_name='*' as well as chn_name=<channel> |
| 1.16    | 07/30/21 | Fixed resilience of merge command - only insert ot update if hvr_op != 0 |
| 1.17    | 07/30/21 | Added -E & -i options for refreshing two targets with the same job |
| 1.18    | 08/04/21 | Fixed (re)create of target table appending rows |
| 1.19    | 08/06/21 | Fixed regression from v1.18 where create table failed on a managed target table |
|         |          | Added finer controls over what file operations are executed |
|         |          | Reduce the number of files returned by azstore_service.get_file_system_client.get_paths |
| 1.20    | 08/24/21 | Added a '+cleanup' option for HBVR_DBRK_FILESTORE_OPS to cleanup files |
| 1.21    | 08/25/21 | Only cleanup during integrate, not refresh |
| 1.22    | 08/27/21 | Create burst table explicilty instead of as select from target |
| 1.23    | 09/01/21 | Added an option (-n) to apply inserts using INSERT sql instead of MERGE |
| 1.24    | 09/02/21 | Added support for partitioning |
| 1.25    | 09/02/21 | Added support for parallel processing |
| 1.26    | 09/03/21 | Refactored the MERGE SQL, INSERT SQL |
| 1.27    | 09/09/21 | Added support for wildcards in partitioning spec |
| 1.28    | 09/10/21 | Use target column ordering for select clause of INSERT SQL |
| 1.29    | 09/21/21 | Re-introduced logic that removes non-burst columns if refresh |
| 1.30    | 09/22/21 | Fixed a couple of bugs building table map |
| 1.31    | 09/30/21 | Fixed order of columns in target table when created |
| 1.32    | 09/30/21 | Added way to set a delay between loading the burst and merge |

