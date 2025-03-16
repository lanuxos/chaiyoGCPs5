# Secure BigLake Data
## Cloud IAM: Qwik Start [GSP064]
### Task 1. Explore the IAM console and project level roles
- 
### Task 2. Prepare a Cloud Storage bucket for access testing 
- 
### Task 3. Remove project access
- 
### Task 4. Add Cloud Storage permission
- 
## Data Catalog: Qwik Start [GSP729]
### 
- 
## BigLake: Qwik Start [GSP1040]
### Create and view a connection resource.
- BigQuery > BigQuery Studio > Done
- +ADD > Connections to external data sources
- Vertex AI remote models, remote functions and BigLake (Cloud Resource)
- ID: my-connection
- Multi-region > US (multiple regions in United States)
- Create connection
- copy Service Account ID

### Set up access to a Cloud Storage data lake.
- IAM & Admin > IAM
- +GRANT ACCESS
- New principals: SERVICE_ACCOUNT_ID
- Select a role: Storage Object Viewer
- Save

### Create a BigLake table.
- BigQuery > Studio > Create dataset
- Dataset ID: demo_dataset
- Multi-region: US(multiple regions in United States)
- Create Dataset
- three dots: Create table
- Create table from: Google Cloud Storage
- Browse: BUCKET_NAME, customer.csv
- Select
- Destination: demo_dataset
- table name: biglake_table
- table type: External Table
- Create a BigLake table using a Cloud Resource connection
- us.my-connection
- Schema: Edit as text
```json
[
{
    "name": "customer_id",
    "type": "INTEGER",
    "mode": "REQUIRED"
  },
  {
    "name": "first_name",
    "type": "STRING",
    "mode": "REQUIRED"
  },
  {
    "name": "last_name",
    "type": "STRING",
    "mode": "REQUIRED"
  },
  {
    "name": "company",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "address",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "city",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "state",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "country",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "postal_code",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "phone",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "fax",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "email",
    "type": "STRING",
    "mode": "REQUIRED"
  },
  {
    "name": "support_rep_id",
    "type": "INTEGER",
    "mode": "NULLABLE"
  }
]
```
- Create Table

### Query a BigLake table through BigQuery.
- from biglake_table > Query
```sql
SELECT * FROM `Project ID.demo_dataset.biglake_table`
```
- Run

### Set up access control policies.
- add policy tags to columns
  - BigQuery > Studio
  - demo_dataset > biglake_table
  - Edit Schema
  - checked [address, postal_code, phone]
  - Add policy tag
  - TAXONOMY_NAME > biglake_policy
  - select
  - Save
- verify the column level security
  - biglake_table
```sql
SELECT * FROM `Project ID.demo_dataset.biglake_table`
```
  - Run
```sql
SELECT *  EXCEPT(address, phone, postal_code)
FROM `Project ID.demo_dataset.biglake_table`
```
### Upgrade external tables to BigLake tables.
- Create the external table
  - three dots of demo_dataset > Create table
  - Create table from, Google Cloud Storage
  - Browse (BUCKET_NAME, invoice.csv)
  - Destination: demo_dataset
  - table name: external_table
  - table type: External Table
  - Schema, Edit as text
```json
[
{
    "name": "invoice_id",
    "type": "INTEGER",
    "mode": "REQUIRED"
  },
  {
    "name": "customer_id",
    "type": "INTEGER",
    "mode": "REQUIRED"
  },
  {
    "name": "invoice_date",
    "type": "TIMESTAMP",
    "mode": "REQUIRED"
  },
  {
    "name": "billing_address",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "billing_city",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "billing_state",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "billing_country",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "billing_postal_code",
    "type": "STRING",
    "mode": "NULLABLE"
  },
  {
    "name": "total",
    "type": "NUMERIC",
    "mode": "REQUIRED"
  }
]
```
  - Create table
- update external table to Biglake table
```shell
export PROJECT_ID=$(gcloud config get-value project)
bq mkdef \
--autodetect \
--connection_id=$PROJECT_ID.US.my-connection \
--source_format=CSV \
"gs://$PROJECT_ID/invoice.csv" > /tmp/tabledef.json
```
  - verify table definition has been created
```shell
cat /tmp/tabledef.json
```
  - get the schema from table
```shell
bq show --schema --format=prettyjson  demo_dataset.external_table > /tmp/schema
```
  - update the table using the new external table definition
```shell
bq update --external_table_definition=/tmp/tabledef.json --schema=/tmp/schema demo_dataset.external_table
```

## Secure BigLake Data: Challenge Lab [ARC129]
### Your challenge
You are asked to help a newly formed development team with some of their initial work on a new project. Specifically, they need a new BigLake table from a Cloud Storage file with the appropriate permissions to limit access to sensitive data columns; you receive the following request to complete the following tasks:

Create a BigLake table from an existing file on Cloud Storage.
Apply and verify policy tags to restrict access to columns containing sensitive data.
Remove direct IAM permissions to Cloud Storage for other users (after policy tags have been applied to protect the data).
Some standards you should follow:

Ensure that any needed APIs (such as Data Catalog and BigQuery Connection API) are successfully enabled and that necessary service accounts have the appropriate permissions.
Create all resources in the multiple regions in United States, unless otherwise directed.
### Task 1. Create a BigLake table using a Cloud Resource connection
- check API enabled [Data Catalog, BigQuery Connection API]
- create cloud resource connection
  - BigQuery > BigQuery Studio > Done
  - +ADD > Connections to external data sources
  - Vertex AI remote models, remote functions and BigLake (Cloud Resource)
  - ID: user_data_connection
  - Multi-region > US (multiple regions in United States)
  - Create connection
  - copy Service Account ID
- Set up access to a Cloud Storage data lake.
  - IAM & Admin > IAM
  - +GRANT ACCESS
  - New principals: SERVICE_ACCOUNT_ID
  - Select a role: Storage Object Viewer
  - Save
- create a Biglake table [user-online-sessions.csv]
  - BigQuery > Studio > Create dataset
  - Dataset ID: online_shop
  - Multi-region: US(multiple regions in United States)
  - Create Dataset
  - three dots: Create table
  - Create table from: Google Cloud Storage
  - Browse: BUCKET_NAME, customer.csv
  - Select
  - Destination: online_shop
  - table name: user_online_sessions
  - table type: External Table
  - Create a BigLake table using a Cloud Resource connection
  - us.user_data_connection
  - Schema: Auto detect
  - Create Table
### Task 2. Apply and verify policy tags on columns containing sensitive data
- add policy tags to columns
  - BigQuery > Studio
  - online_shop > user_online_sessions
  - Edit Schema
  - checked [zip, latitude, ip_address, longitude]
  - Add policy tag
  - TAXONOMY_NAME > biglake_policy
  - select
  - Save
```sql
SELECT * EXCEPT(zip, latitude, ip_address, longitude) FROM `qwiklabs-gcp-03-884c88381110.online_shop.user_online_sessions`
```
### Task 3. Remove IAM permissions to Cloud Storage for other users
- Leave the IAM role for project viewer.
- Remove only the IAM role for Cloud Storage.