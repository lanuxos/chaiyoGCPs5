# Prepare Data for ML APIs on Google Cloud
## Dataprep: Qwik Start [GSP105]
### Task 1. Create a Cloud Storage bucket in your project
- Navigation menu > Cloud Storage > Buckets
- Create Bucket
- Enforce public access prevention on this bucket
- Create
### Task 2. Initialize Cloud  Dataprep
```shell
gcloud beta services identity create --service=dataprep.googleapis.com
```
- Navigation Menu > Dataprep
- Accept > Autherize > Agree and Continue > Allow > Accept > Continue
### Task 3. Create a flow
- Flows > Create > Blank Flow
- name untitled flow: FEC-2016
- OK
### Task 4. Import datasets
- Add Datasets > Importr Datasets
- Cloud Storage [gs://spls/gsp105]
- choose file or folder
- Go
- us-fec/
- + cn-2016.txt
- name: Candidate Master 2016
- + itcont-2016-orig.txt
- name: Campaign Contributions 2016
- Import & Add to Flow
### Task 5. Prep the candidate file
- Edit Recipe
- suggestions: Keep rows
- Add
- column6 > flag icon > change to STRING
- column7 > P column > Add
### Task 6. Wrangle the Contributions file and join it to the Candidates file
- FEC-2016
- click to select the grayed out Campaign Contributions 2016
- Add > Recipe > Edit Recipe
- Add new step
```
replacepatterns col: * with: '' on: `{start}"|"{end}` global: true
```
- Add
- New Step > Join datasets > Candidate Master 2016
- Accept
- on the right side, click on pencil icon [Edit icon]
- column2 = column11
- Save and Continue
- Next
- Check all columns
- Review 
- Add to Recipe
### Task 7. Summary of data
- New Step > Transformation
```
pivot value:sum(column16),average(column16),countif(column16 > 0) group: column2,column24,column8
```
- Add
### Task 8. Rename columns
- New Step
```
rename type: manual mapping: [column24,'Candidate_Name'], [column2,'Candidate_ID'],[column8,'Party_Affiliation'], [sum_column16,'Total_Contribution_Sum'], [average_column16,'Average_Contribution_Sum'], [countif,'Number_of_Contributions']
```
- Add
- New Step
```
set col: Average_Contribution_Sum value: round(Average_Contribution_Sum)
```
- Add
## Dataflow: Qwik Start - Templates [GSP192]
### Task 1. Ensure that the Dataflow API is successfully re-enabled
- Dataflow API
- Manage
- Disable API
- Enable
### Task 2. Create a BigQuery dataset, BigQuery table, and Cloud Storage bucket using Cloud Shell
```shell
bq mk taxirides

bq mk \
--time_partitioning_field timestamp \
--schema ride_id:string,point_idx:integer,latitude:float,longitude:float,\
timestamp:timestamp,meter_reading:float,meter_increment:float,ride_status:string,\
passenger_count:integer -t taxirides.realtime

export BUCKET_NAME=

gsutil mb gs://$BUCKET_NAME/
```
### Task 3. Create a BigQuery dataset, BigQuery table, and Cloud Storage bucket using the Google Cloud console
- BigQuery
- Done
- three dots: Create dataset [taxirides]
- us (multiple regions in United States)
- CREATE DATASET
- three dots: Create Table [realtime]
- Schema: Edit as text
```
ride_id:string,point_idx:integer,latitude:float,longitude:float,timestamp:timestamp,
meter_reading:float,meter_increment:float,ride_status:string,passenger_count:integer
```
- Create table
- cloud Storage > Buckets > Create bucket
- name & Create
### Task 4. Run the pipeline
```shell
gcloud dataflow jobs run iotflow \
    --gcs-location gs://dataflow-templates-"Region"/latest/PubSub_to_BigQuery \
    --region "Region" \
    --worker-machine-type e2-medium \
    --staging-location gs://"Bucket Name"/temp \
    --parameters inputTopic=projects/pubsub-public-data/topics/taxirides-realtime,outputTableSpec="Table Name":taxirides.realtime
```
- Navigation menu
- Dataflow > Jobs
### Task 5. Submit a query
```sql
SELECT * FROM `"Bucket Name".taxirides.realtime` LIMIT 1000
```
### Task 6. Test your understanding
- True
- Pub/Sub to BigQuery

## Dataflow: Qwik Start - Python [GSP207]
### Task 0. environment setup & API enable
```shell
gcloud config set compute/region "REGION"
```
- Dataflow API > Manage > Disable API > Enable
### Task 1. Create a Cloud Storage bucket
- Navigation menu > Cloud Storage > Buckets
- Create bucket [us-multi-region]
- Create
- Confirm
### Task 2. Install the Apache Beam SDK for Python
```shell
docker run -it -e DEVSHELL_PROJECT_ID=$DEVSHELL_PROJECT_ID python:3.9 /bin/bash

pip install 'apache-beam[gcp]'==2.42.0

python -m apache_beam.examples.wordcount --output OUTPUT_FILE
```
### Task 3. Run an example Dataflow pipeline remotely
```shell
BUCKET=gs://<bucket name provided earlier>
python -m apache_beam.examples.wordcount --project $DEVSHELL_PROJECT_ID \
  --runner DataflowRunner \
  --staging_location $BUCKET/staging \
  --temp_location $BUCKET/temp \
  --output $BUCKET/results/output \
  --region "filled in at lab start"
```
### Task 4. Check that your Dataflow job succeeded
- Dataflow 
- Status: Success
### Task 5. Test your understanding
- True

## Dataproc: Qwik Start - Command Line [GSP103]
### Task 0. environment setup
- Cloud Dataproc API > Enable
- Permission to Service Account
  - Navigation menu > IAM & Admin IAM
  - edit [compute@developer.gserviceaccount.com]
  - + ADD ANOTHER ROLE 
  - Dataproc Administrator, Dataproc Editor, Dataproc Resource Manager Node Service Agent, Dataproc Worker, Editor, Owner, Storage Admin
  - Save
### Task 1. Create a cluster
- Navigation menu > Dataproc > Clusters > Create cluster
- Create for Cluster on Compute Engine
- follow details
- Create
### Task 2. Submit a job
- Jobs
- follow detail
- Submit
### Task 3. View the job output
- Jobs 
- LINE WRAP > ON
### Task 4. Update a cluster to modify the number of workers
- Cluster
- example-cluster
- Configuration
- Edit
- Worker nodes: 4
- Save
- Jobs
- SUBMIT
- follow details
- Submit
### Task 5. Test your understanding
- 

## Cloud Natural Language API: Qwik Start [GSSP104]
### Task 1. Create a cluster
```shell
gcloud config set dataproc/region Region

PROJECT_ID=$(gcloud config get-value project) && \
gcloud config set project $PROJECT_ID

PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --role=roles/storage.admin

gcloud compute networks subnets update default --region=REGION  --enable-private-ip-google-access

gcloud dataproc clusters create example-cluster --worker-boot-disk-size 500 --worker-machine-type=e2-standard-4 --master-machine-type=e2-standard-4
```
### Task 2. Submit a job
```shell
gcloud dataproc jobs submit spark --cluster example-cluster \
  --class org.apache.spark.examples.SparkPi \
  --jars file:///usr/lib/spark/examples/jars/spark-examples.jar -- 1000
```
### Task 3. Update a cluster
```shell
gcloud dataproc clusters update example-cluster --num-workers 4

gcloud dataproc clusters update example-cluster --num-workers 2
```
### Task 4. Test your understanding
- True

## Speech-to-Text API: Qwik Start [GSP119]
### Task 1. Create an API key
- Navigation menu > APIs & services > Credentials > API key
- Navigation menu > Compute Engine > SSH
```shell
export API_KEY=
```
### Task 2. Create your Speech-to-Text API request
```shell
cat > request.json << EOF
{
  "config": {
      "encoding":"FLAC",
      "languageCode": "en-US"
  },
  "audio": {
      "uri":"gs://cloud-samples-tests/speech/brooklyn.flac"
  }
}
EOF
```
### Task 3. Call the Speech-to-Text API
```shell
curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json \
"https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}"

curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json \
"https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" > result.json
```
## Video Intelligence: Qwik Start [GSP154]
### Task 1. Set up authorization
- Create service account named: quickstart
```shell
gcloud iam service-accounts create quickstart
```
- Create a service account key file
```shell
gcloud iam service-accounts keys create key.json --iam-account quickstart@<your-project-123>.iam.gserviceaccount.com
```
- authenticate your service account
```shell
gcloud auth activate-service-account --key-file key.json
```
- obtain an authorization token
```shell
gcloud auth print-access-token
```
### Task 2. Make an annotate video request
- Create request.json
```shell
cat > request.json <<EOF
{
   "inputUri":"gs://spls/gsp154/video/train.mp4",
   "features": [
       "LABEL_DETECTION"
   ]
}
EOF
```
- Make videos: annotate
```shell
curl -s -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$(gcloud auth print-access-token)'' \
    'https://videointelligence.googleapis.com/v1/videos:annotate' \
    -d @request.json
```
- 
```shell
export PROJECTS=
export LOCATIONS=
export OPERATION_NAME=
curl -s -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$(gcloud auth print-access-token)'' \
    'https://videointelligence.googleapis.com/v1/projects/${PROJECTS}/locations/${LOCATIONS}/operations/${OPERATION_NAME}'
```

## Prepare Data for ML APIs on Google Cloud: Challenge Lab [GSP323]
### Task 1. Run a simple Dataflow job
- environment setup
```shell
export DATASET=lab_567
export BUCKET=qwiklabs-gcp-03-075bdb566914-marking
export TABLE=customers_946
export REGION=$(gcloud compute project-info describe \
export BUCKET_URL_1=gs://qwiklabs-gcp-03-075bdb566914-marking/task3-gcs-119.result
export BUCKET_URL_2=gs://qwiklabs-gcp-03-075bdb566914-marking/task4-cnl-793.result
gcloud services enable apikeys.googleapis.com
gcloud alpha services api-keys create --display-name="awesome" 
KEY_NAME=$(gcloud alpha services api-keys list --format="value(name)" --filter "displayName=awesome")
API_KEY=$(gcloud alpha services api-keys get-key-string $KEY_NAME --format="value(keyString)")
--format="value(commonInstanceMetadata.items[google-compute-default-region])")
PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format="json" | jq -r '.projectNumber')

bq mk $DATASET

gsutil mb gs://$BUCKET

gsutil cp gs://cloud-training/gsp323/lab.csv  .
gsutil cp gs://cloud-training/gsp323/lab.schema .
cat lab.schema

echo '[
    {"type":"STRING","name":"guid"},
    {"type":"BOOLEAN","name":"isActive"},
    {"type":"STRING","name":"firstname"},
    {"type":"STRING","name":"surname"},
    {"type":"STRING","name":"company"},
    {"type":"STRING","name":"email"},
    {"type":"STRING","name":"phone"},
    {"type":"STRING","name":"address"},
    {"type":"STRING","name":"about"},
    {"type":"TIMESTAMP","name":"registered"},
    {"type":"FLOAT","name":"latitude"},
    {"type":"FLOAT","name":"longitude"}
]' > lab.schema

bq mk --table $DATASET.$TABLE lab.schema

gcloud dataflow jobs run awesome-jobs --gcs-location gs://dataflow-templates-$REGION/latest/GCS_Text_to_BigQuery --region $REGION --worker-machine-type e2-standard-2 --staging-location gs://$DEVSHELL_PROJECT_ID-marking/temp --parameters inputFilePattern=gs://cloud-training/gsp323/lab.csv,JSONPath=gs://cloud-training/gsp323/lab.schema,outputTable=$DEVSHELL_PROJECT_ID:$DATASET.$TABLE,bigQueryLoadingTemporaryDirectory=gs://$DEVSHELL_PROJECT_ID-marking/bigquery_temp,javascriptTextTransformGcsPath=gs://cloud-training/gsp323/lab.js,javascriptTextTransformFunctionName=transform

gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
    --member "serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
    --role "roles/storage.admin"

gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
  --member=user:$USER_EMAIL \
  --role=roles/dataproc.editor

gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
  --member=user:$USER_EMAIL \
  --role=roles/storage.objectViewer

gcloud compute networks subnets update default \
    --region $REGION \
    --enable-private-ip-google-access

gcloud iam service-accounts create awesome \
  --display-name "my natural language service account"

sleep 15

gcloud iam service-accounts keys create ~/key.json \
  --iam-account awesome@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com

export GOOGLE_APPLICATION_CREDENTIALS="/home/$USER/key.json"

sleep 30

gcloud auth activate-service-account awesome@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com --key-file=$GOOGLE_APPLICATION_CREDENTIALS

gcloud ml language analyze-entities --content="Old Norse texts portray Odin as one-eyed and long-bearded, frequently wielding a spear named Gungnir and wearing a cloak and a broad hat." > result.json

gcloud auth login --no-launch-browser --quiet

gsutil cp result.json $BUCKET_URL_2

gcloud auth login --no-launch-browser --quiet

gcloud dataproc clusters create awesome --enable-component-gateway --region $REGION --master-machine-type e2-standard-2 --master-boot-disk-type pd-balanced --master-boot-disk-size 100 --num-workers 2 --worker-machine-type e2-standard-2 --worker-boot-disk-type pd-balanced --worker-boot-disk-size 100 --image-version 2.2-debian12 --project $DEVSHELL_PROJECT_ID

VM_NAME=$(gcloud compute instances list --project="$DEVSHELL_PROJECT_ID" --format=json | jq -r '.[0].name')
export ZONE=$(gcloud compute instances list $VM_NAME --format 'csv[no-heading](zone)')

gcloud compute ssh --zone "$ZONE" "$VM_NAME" --project "$DEVSHELL_PROJECT_ID" --quiet --command="hdfs dfs -cp gs://cloud-training/gsp323/data.txt /data.txt"

gcloud compute ssh --zone "$ZONE" "$VM_NAME" --project "$DEVSHELL_PROJECT_ID" --quiet --command="gsutil cp gs://cloud-training/gsp323/data.txt /data.txt"

gcloud dataproc jobs submit spark \
  --cluster=awesome \
  --region=$REGION \
  --class=org.apache.spark.examples.SparkPageRank \
  --jars=file:///usr/lib/spark/examples/jars/spark-examples.jar \
  --project=$DEVSHELL_PROJECT_ID \
  -- /data.txt

cat > request.json <<EOF
{
  "config": {
      "encoding":"FLAC",
      "languageCode": "en-US"
  },
  "audio": {
      "uri":"gs://cloud-training/gsp323/task3.flac"
  }
}
EOF

curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json \
"https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" > result.json

gsutil cp result.json gs://qwiklabs-gcp-03-075bdb566914-marking/task3-gcs-119.result

gcloud iam service-accounts create quickstart

gcloud iam service-accounts keys create key.json --iam-account quickstart@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com

gcloud auth activate-service-account --key-file key.json

cat > request.json <<EOF 
{
   "inputUri":"gs://spls/gsp154/video/train.mp4",
   "features": [
       "TEXT_DETECTION"
   ]
}
EOF

curl -s -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$(gcloud auth print-access-token)'' \
    'https://videointelligence.googleapis.com/v1/videos:annotate' \
    -d @request.json

curl -s -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$(gcloud auth print-access-token)'' \
    'https://videointelligence.googleapis.com/v1/projects/qwiklabs-gcp-03-075bdb566914/locations/us-east1/operations/projects/988562689243/locations/asia-east1/operations/13895930082462156116'

```

### Task 2. Run a simple Dataproc job

### Task 3. Use the Google Cloud Speech-to-Text API

### Task 4. Use the Cloud Natural Language API


bg dataset
lab_519

bucket
qwiklabs-gcp-04-889267ad7a39-marking

table
customers_615

result
gs://qwiklabs-gcp-04-889267ad7a39-marking/task3-gcs-408.result

gs://qwiklabs-gcp-04-889267ad7a39-marking/task4-cnl-626.result

