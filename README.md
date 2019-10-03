# DDL TOOLS

DDL tools is a set of Python libraries and applications designed to make creating and working with ThoughtSpot DDL 
easier. 
 
## Setup

This section will walk you through how to set up the environment to get started with the DDL tools.

### Environment

These tools have all been written in Python 3.7 and expect to be run in a 3.6 environment at a minimum.

You can either install directly into an existing Python environment or use a virtual environment. 
It's often better to run in an virtual environment to avoid conflicts with dependencies,  since ddl_tools relies on 
other packages to run.  Note that this installation process requires external access to install packages and 
get the ddl_tools code from GitHub.

To create a virtual environment, you can run the following from the command prompt:

`$ virtualenv -p python3 ./venv`

Note that the `venv` folder can be whatever name and location you like (preferably external to a code repository).

Next, you need to activate the environment with the command: 

`$ source ./venv/bin/activate`

Note that you will need to reactivate the environment whenever you want to use it.  

You should see your prompt change to (venv) plus whatever it was before.  To verify the python version run:

`$ python --version`  You want to be on version 3.6 or higher.

If you want to leave the virtual environment, simple enter `$ deactivate` or close the terminal you are using.

See https://virtualenv.pypa.io/en/latest/ for more details on using virtualenv.

## Downloading and installing the DDL tools

Now that you have an environment for installing into you can install directly from GitHub with the command:

`$ pip install git+https://github.com/thoughtspot/ddl_tools`.  

You should see output as the ddl_tools and dependencies are installed.  

If you want or need to update to a newer version of the DDL tools, use the command:

`$ pip install --upgrade git+https://github.com/thoughtspot/ddl_tools`.  

## Running the pre-built tools

All of the pre-built tools are run using the general format: 

`python -m ddltools.<tool-name>`

Note there is no `.py` at the end and you *must* use `python -m`.  So for example to run `convert_ddl` and see the 
options, you would enter `python -m ddl_tools.convert_ddl --help`  Try it now and verify your environment is all set.

The DDL tools currently consist of two scripts:
1. `convert_ddl`, converts between different formats including DDL for most major databases, TQL, and Excel.
2. `ddldiff` a tool to show the differences between two different schemas.  Note that this tool is currently beta.

## convert_ddl

~~~
usage: convert_ddl [-h] [--empty] [--from_ddl FROM_DDL] [--to_tql TO_TQL]
                   [--from_excel FROM_EXCEL] [--to_excel TO_EXCEL]
                   [-d DATABASE] [-s SCHEMA] [-c] [-l] [-u] [--camelcase]
                   [-v] [--debug]

optional arguments:
  -h, --help            show this help message and exit
  --empty               creates an empty modeling file.
  --from_ddl FROM_DDL   will attempt to convert DDL from the infile
  --to_tql TO_TQL       will convert to TQL to the outfile
  --from_excel FROM_EXCEL
                        convert from the given Excel file
  --to_excel TO_EXCEL   will convert to Excel and write to the outfile.
  -d DATABASE, --database DATABASE
                        name of ThoughtSpot database
  -s SCHEMA, --schema SCHEMA
                        name of ThoughtSpot schema
  -c, --create_db       generate create database and schema statements
  -l, --lowercase       create table and column names in lowercase
  -u, --uppercase       create table and column names in uppercase
  --camelcase           converts table names and columns names with _ to camel
                        case, e.g. my_table becomes MyTable.
  -v, --validate        validate the database
  --debug               Prints details of parsing.
~~~

## ddldiff

Compares two DDL files and can generate alters to make the first match the second.

~~~
usage: ddl_diff 
       [-h] [--ddl1 DDL1] [--ddl2 DDL2] [--database DATABASE]
       [--schema SCHEMA] [--alter1] [--alter2] [--ignore_case]

optional arguments:
  -h, --help           show this help message and exit
  --ddl1 DDL1          DDL file containing the schema that would be changed.
  --ddl2 DDL2          DDL file containing the new schema.
  --database DATABASE  name of database for generating alter statements
  --schema SCHEMA      name of schema for generating alter statements
  --alter1             Generates drop, create, alter, etc. statements that
                       would be needed to make the first DDL align with the
                       second.
  --alter2             Generates drop, create, alter, etc. statements that
                       would be needed to make the second DDL align with the
                       first.
  --ignore_case        Causes case of names to be ignored
  ~~~

### Sample of common workflow to convert DDL

The standard workflow that we use with new DDL that we want to connvert uses the following steps:
* Convert from DDL to Excel
* Review and enhance the model in Excel
* Validate the model
* Convert from Excel to TQL DDL

After that, you can create the database tables in TQL.

### Sample commands

To convert from some other database DDL to Excel:
```
convert_ddly --from_ddl <somefile> --to_excel <somefile> --database <db-name>
```
This will result in a new excel file.  Note that the .xlsx extension is option on the excel file name.

To validate a model from Excel:
```
convert_ddl --from_excel <somefile>.xlsx --validate
```
You will either get errors or a message that the model is valid.

To convert from Excel to TQL DDL:
```bazaar
convert_ddl --from_excel <somefile>.xlsx --to_tql <somefile> 
```
You will get an output file that contains (hopefully) valid TQL syntax.

### Cleanup in Excel
convert_ddl does it's best to parse DDL from a wide variety of sources, but there are some feature gaps and 
occasional things you'll need to clean up.

WARNING:  Do not delete any of the existing columns.  Adding columns is OK since they will be ignored.
 
#### Columns tab
The first tab has all of the columns.  You should look for data types of "UNKNOWN".  These are data types
in the original that the parser couldn't decide how to translate.  We are constantly finding new types and adding.

You should also review the columns to see if you like the type chosen.  In particular, we default to BIGINT and DOUBLE
when it's not obvious.  But you may not need these wider types.  Also, some DATE types should be mapped to DATETIME 
instead of DATE.  Finally, don't worry about the actual size in the VARCHAR.  We just ignore this in ThoughtSpot.

#### Tables tab

Most of this tab is automatically generated, but you will want to update three columns:  primary key, shard key, and 
number of shards.  The last two are only needed if you want to shard the table.  The # rows column is optional and 
doesn't impact TQL generation, but it helps with determining shard needs.  

The PKs to and from columns tell the number of foreign key relationships for each table.  This can be convenient for
determining if the table have relationships defined.

#### Foreign keys tab

The foreign keys tab allows you to enter FK relationships between the tables.  The columns should be obvious.  
Note that ThoughtSpot requires a foreign key to reference a primary key and the number and type of columns
must match.  The validation step will check for these conditions.  

You can use any name you like.  A common practice is FK_fromtable_to_totable_by_columna.  You can use the following
formula in the first column to generate this formula.  Note that the quotes don't always paste corrrecty and this 
formula assumes it's in the second column.  Update as needed.

`="fk_"&D2&"_to_"&F2&"_by_"&SUBSTITUTE(SUBSTITUTE(E2," ",""),",","_and_")`

#### Relationships tab

The relationships tab lets you enter relationships between tables.  In general, you should try to use foreign keys, but 
sometimes relationships are more appropriate.  The condition must be a valid condition between the tables.  This 
condition is not validated by the validator other than it exists.

