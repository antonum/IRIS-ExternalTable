# IRIS External Table

![CI](https://github.com/antonum/IRIS-ExternalTable/workflows/CI/badge.svg)

IRIS External Table allows you to access files in local filesystem and cloud BLOB storage such as AWS S3 and Azure BLOB Storage as regular IRIS SQL tables. The resulting tables can be queried like regular IRIS tables, can be JOINed with other tables, used in subselects, used for bulk loading the data into another place with `INSERT INTO localtable SELECT FROM external_table` etc.

Note that the 

![External Table](./images/ExternalTableDiagram.png)

External tables can be based on a single file or directory/bucket. Using tables, based on multiple files is ideal for situations like log processing, where data is constantly added as new files.

Let's say you have the following text file, located in S3 bucket at `s3://mybucket/myfile.csv`:
```sql
name,personid
anton,1
john,2
bill,3
```
You can execute the following SQL statements:
```sql
CREATE TABLE person.firstname (
    firstname VARCHAR(50), 
    personid INT
)

CALL EXT.ConvertToExternal(
    'person.firstname',
    '{
        "adapter":"EXT.AWSS3",
        "location":"s3://mybucket/myfile.csv",
        "delimiter": ",",
        "skipHeaders": 1
    }')
-- change "s3://mybucket/myfile.csv" to "/myfolder/myfile.csv" and "EXT.AWSS3" to "EXT.LocalFile" to use the local filesystem instead

SELECT * FROM person.firstname
firstname	personid
anton	    1
john	    2
bill	    3

3 Rows(s) Affected
```
Delimited files and JSON can be used as data sources. Local filesystem, AWS S3, Google Storage Bucket and Azure BLOB storage data sources implemented.

## Installation
```
git clone https://github.com/antonum/IRIS-ExternalTable.git
iris session iris
USER>set sc = ##class(%SYSTEM.OBJ).LoadDir("<path-to>/IRIS-ExternalTable/src", "ck",,1)
```
## IRIS Package Manager (ZPM) - based installation

IRIS External Table can be installed with ObjectScript Package Manager. https://openexchange.intersystems.com/package/ObjectScript-Package-Manager-2

```
USER> zpm
zpm: USER>install external-table
```

## Usage

Create table as usual:

```sql
CREATE TABLE test.table1
    (field1 VARCHAR(50), 
    int1 INT, 
    float1 DOUBLE, 
    field2 VARCHAR(50))
```
Then convert it to "External" table using `EXT.ConvertToExternal` stored procedure, specifying existing table name and table configuration in JSON format.

```sql
call EXT.ConvertToExternal(
    'test.table1',
    '{ 
        "adapter":"EXT.GoogleStorage",
        "location":"gs://iris-external-table/",
        "delimiter": ","
    }' 
)
```
Table configuration can also be specified by using reference to the file.
```sql
CALL EXT.ConvertToExternal('test.table1','<path-to>/multifile-gs.json')
```
Where `multifile-gs.json` is:
```json
{
    "adapter":"EXT.GoogleStorage",
    "location":"gs://iris-external-table/",
    "delimiter": ","
}
```

`adapter` and `location` are required parameters.

`location` can point to a single file or directry/bucket. For directory/bucket, make sure to include tailing slash `"location":"<path-to>/"`

Supported adapters:
- `EXT.LocalFile` - files on the local file system. Location format: `/<path-to>/<filename.csv>`
- `EXT.AWSS3` - AWS S3 Buckets. Location format: `s3://<bucketname>/<filename.csv>`
- `EXT.GoogleStorage` - Google Cloud Storage Buckets. Location format: `gs://<bucketname>/<filename.csv>`
- `EXT.Azure` - Azure BLOB Storage containers. Location format: `https://<bucketname>.blob.core.windows.net/<containername>/<filename.csv>`

## %PATH

For all the tables, the additional hidden `%PATH` field is created, containing the underlying filename. It can be useful for processing log data where file name itself contains data such as date/time. `%PATH` is not included in `SELECT * FROM TableName` query and must be explicitly specified.

```sql
SELECT %PATH, * FROM myExternalTable
```

## CSV files

You must specify `"delimiter": ","`

`"skipHeaders": 1` is optional and identifies number of lines to skip at the beginning og the file. Default is 0.

## Quoted CSV files

Quoted CSV allows for values to have comma inside the fields, as long as the entire field is enclosed in double quotes. For performance reasons you might prefer to use `"type": "csv"` where possible.
```
Year,Make,Model,Description,Price
1997,Ford,E350,"ac, abs, moon",3000.00
1999,Chevy,"Venture Extended Edition","",4900.00
1999,Chevy,"Venture Extended Edition, Very Large",,5000.00
1996,Jeep,Grand Cherokee,"MUST SELL! air, moon roof, loaded",4799.00
```
You must specify `"delimiter": ","` and `"type": "quoted_csv"`. 

`"skipHeaders": 1` is optional and identifies number of lines to skip at the beginning og the file. Default is 0.
```sql
 create table cars ( 
    year INT,
    make char(10),
    model varchar(50),
    description varchar(50),
    price float
)
call EXT.ConvertToExternal(
    'cars',
    '{ 
        "adapter":"EXT.LocalFile", 
        "location":"test/sql/cars-for-sales.csv",
        "type": "quoted_csv", 
        "delimiter": ",", 
        "skipHeaders": 1
    }' 
select * from cars
```
## JSON files

JSON Lines http://jsonlines.org/ format supported. File contains multiple lines, every line is a single JSON document, converted to the row.

JSON table configuration must specify `"type": "jsonlines"`

Fields in JSON data are matched to the field names in SQL table by default. Names are case sensitive.

For complex/non-flat JSON structures you can specify optional `"jsonParser"` section, where table field name is matched to the ObjectScript code, extracting field data from the `%jsonline` object. Use https://docs.intersystems.com/irislatest/csp/docbook/DocBook.UI.Page.cls?KEY=GJSON_create for the reference. Field names with underscores, dashes etc. must be enclosed in quoted double quotes.

```sql
 create table toronto.greenparking (
    address varchar(150),
    enable_streetview char(3),
    lat float,
    lng float,
    rate varchar(50),
    payment_options varchar(50),
    rate_details_periods varchar(150)
)

call EXT.ConvertToExternal(
    'toronto.greenparking',
    '{
        "adapter":"EXT.LocalFile",
        "location":"<path-to>/toronto-green-parking.json",
        "type": "jsonlines",
        "jsonParser": {
            "payment_options": "%jsonline.\"payment_options\"",
            "rate_details_periods": "%jsonline.\"rate_details\".periods"
        }
    }')
```
For the tables of `jsonlines` type additional hidden `%DOCUMENT` field is created, similar to `%PATH`. `%DOCUMENT` contains the entire JSON document that can be used for extracting data from documents in downstram processing.

```sql
SELECT %DOCUMENT, * FROM myExternalTable
```

## Data Migration

External table can be used to migrate data into internal table, using INSER INTO .. SELECT FROM syntax. 

```sql
INSERT INTO internal_table SELECT * FROM external_table

INSERT INTO internal_table (firstname, personid) SELECT firstname, personid*10 FROM external_table
```

## Cloud Support - Authentication

All cloud providers expect you to use authenticated requests to access data. IRIS-ExternalTable supports the following authentication methods for the cloud providers:

### AWS S3

AWS Uses Account Key/Secret authentication to sign requests. https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys

If you are running IRIS External Table on EC2 instance, the recommended way of dealing with authentication is using EC2 Instance Roles https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html IRIS External Table would be able to use permissions of that role. No extra setup required.

On a local/non EC2 instance you need to specify AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY by either specifying environment variables or installing and configuring AWS CLI client.
```bash
export AWS_ACCESS_KEY_ID=AKIAEXAMPLEKEY
export AWS_SECRET_ACCESS_KEY=111222333abcdefghigklmnopqrst
```

Make sure that environment variable is visible within your IRIS process. You can verify it by running:
```bash
USER>write $system.Util.GetEnviron("AWS_ACCESS_KEY_ID")
```
It should output the value of the key.

or install AWS CLI, following instruction here https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html and run:
```bash
aws configure
```

### Google Cloud

IRIS instance running on GCP Virtual Machine needs no additional steps to allow IRIS External Table to access Google Storage Buckets. Just make sure the following command on the instance returns valid token and instance itself has the permissions to read data from Google Storage.
```bash
gcloud auth print-access-token
```

If IRIS is running not on Google Cloud VM - you need to install Google Cloud SDK on the machine https://cloud.google.com/sdk/install , create Google Service Account https://cloud.google.com/iam/docs/service-accounts and download JSON key for that account. Activate the key on the instance:
```bash
gcloud auth activate-service-account --key-file=my-service-account-key.json
```

### Azure

Work in progress... Check back for updates.

Authentication is based on Azure AD. The following parameters need to be set: `azure-client-id`, `azure-client-secret`, `azure-tenant-id`. (See EXT.Azure for details. Subject to change!!!) Currently using `^EXT.Config(parameter-name)` global.

You can use the following commands to gather this information:
```bash
az login --use-device-code
az ad sp create-for-rbac --sdk-auth
```

## Configuration

^EXT.Config("LocalDir") global can be used to set common path part for table configuration files and EXT.LocalFile based locations.

For instance, if the full path to your table config is `/home/irisowner/config/mytable.json` and data is located in `/home/irisowner/data/mydata.csv`, then setting:
```
set ^EXT.Config("LocalDir")="/home/irisowner/" 
```
would allow you to refer to them as: 
```
call EXT.ConvertToExternal('person.firstname','config/mytable.json')

/home/irisowner/config/mytable.json
{
    "adapter":"EXT.LocalFile",
    "location":"data/mydata.csv",
    "delimiter": ","
}
```
## SQL

External table is designed to support full range of SELECT queries. They can be used along with other external and internal queries in the SELECT statements.

These tables are READ-ONLY and only support FULL-TABLE-SCAN execution plan.

DROP TABLE must be used with %NODELDATA flag
```sql
DROP TABLE external_table  %NODELDATA
```

## Using cloud adapters outside of ExternalTable
 
 Each of the cloud adapters (EXT.AWSS3, EXT.Azure and EXT.GoogleStorage) implements methods GetFileList() and GetFileHeader() that can be used to access files
 stored in the cloud.
 
 *GetFileList("s3://mybucket/",.fileList)* returns the list of files in the bucket/folder in the variable fileList. Note - vatiable must be passed by reference .variableName

*GetFileHeader("s3://mybucket/",.fileHeader)* returns the pointer to the %Stream object, containing the content of the file. You can perform all the usual %Stream operations with it, such as Read() ReadLine(), Rewind(), check for fileHeader.AtEnd etc. https://docs.intersystems.com/irislatest/csp/documatic/%25CSP.Documatic.cls 
```ObjectScript
 set sc=##class(EXT.AWSS3).GetFileList("s3://anton-fhir/data/",.fileList)
 /* list of file in the bucket is now at fileList
    zw fileList 
    fileList=18
    fileList(1)="s3://anton-fhir/data/Adina377_Corkery305_cb12851a-2ebd-4c15-88a9-5bee0f308afc.json"
    fileList(2)="s3://anton-fhir/data/Alfred550_Fadel536_155f7362-37da-4e09-a149-a2fec35e20fa.json"
    ...
    fileList(18)="s3://anton-fhir/data/practitionerInformation1585754149520.json"
 */

 for i=1:1:fileList {
    write !,"Loading:  ",fileList(i)
    set sc=##class(EXT.AWSS3).GetFileHeader(fileList(i),.fileHeader)
    //pointer to %Stream object is now in fileHeader
    //output file content to the console:
    write !,fileHeader.Read()
 }
 ```