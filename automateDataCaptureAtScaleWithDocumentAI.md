# Automate Data Capture at Scale with Document AI
## Create and Test a Document AI Processor [GSP924]
### Task 1. Enable the Cloud Document AI API
- Navigation menu > APIs & Services > Library
- Cloud Document AI API
- Enable
### Task 2. Create and test a general form processor
- Navigation menu > Document AI > Overview
- Explore processors > Create Processor [Form Parser]
- name: form-parser
- region: US
- Create
- [Download sample file](https://storage.googleapis.com/cloud-training/document-ai/generic/form.pdf)
- Upload Test Document
### Task 3. Set up the lab instance
- Navigation menu > Compute Engine > VM Instances
- SSH [document-ai-dev]
- Navigation menu > Document AI > Processors > ID
```shell
export PROCESSOR_ID=fe9dd870156df507
```
- Authenticate API requests
  - set environment
```shell
export PROJECT_ID=$(gcloud config get-value core/project)
```
  - create service account
```shell
export SA_NAME="document-ai-service-account"
gcloud iam service-accounts create $SA_NAME --display-name $SA_NAME
```
  - bind the service account to Document AI API user role
```shell
gcloud projects add-iam-policy-binding ${PROJECT_ID} --member="serviceAccount:$SA_NAME@${PROJECT_ID}.iam.gserviceaccount.com" --role="roles/documentai.apiUser"
```
  - create login credentials
```shell
gcloud iam service-accounts keys create key.json --iam-account  $SA_NAME@${PROJECT_ID}.iam.gserviceaccount.com
export GOOGLE_APPLICATION_CREDENTIALS="$PWD/key.json"
echo $GOOGLE_APPLICATION_CREDENTIALS
```
  - download sample from vm instance
```shell
gsutil cp gs://cloud-training/gsp924/health-intake-form.pdf .
```
  - create json request file
```shell
echo '{"inlineDocument": {"mimeType": "application/pdf","content": "' > temp.json
base64 health-intake-form.pdf >> temp.json
echo '"}}' >> temp.json
cat temp.json | tr -d \\n > request.json
```
### Task 4. Make a synchronous process document request using curl
- submit a form fro processing via curl
```shell
export LOCATION="us"
export PROJECT_ID=$(gcloud config get-value core/project)
curl -X POST -H "Authorization: Bearer "$(gcloud auth application-default print-access-token) -H "Content-Type: application/json; charset=utf-8" -d @request.json https://${LOCATION}-documentai.googleapis.com/v1beta3/projects/${PROJECT_ID}/locations/${LOCATION}/processors/${PROCESSOR_ID}:process > output.json
```
- extract the form entities
```shell
sudo apt-get update 
sudo apt-get install jq
cat output.json | jq -r ".document.text"
```
- extract the list of form fields detected by the processor
```shell
cat output.json | jq -r ".document.pages[].formFields"
```
### Task 5. Test a Document AI form processor using the Python client libraries
- configure your vm instance to use the document ai python client
```shell
gsutil cp gs://cloud-training/gsp924/synchronous_doc_ai.py .
```
- install python client libraries
```shell
sudo apt install python3-pip
python3 -m pip install --upgrade google-cloud-documentai google-cloud-storage prettytable
```
### Task 6. Run the Document AI Python code
- create environment variables
```shell
export PROJECT_ID=$(gcloud config get-value core/project)
export GOOGLE_APPLICATION_CREDENTIALS="$PWD/key.json"
```
- call the syncronouse_doc_ai.py
```shell
python3 synchronous_doc_ai.py --project_id=$PROJECT_ID --processor_id=$PROCESSOR_ID --location=us --file_name=health-intake-form.pdf | tee results.txt
```

## Process Documents with Python Using the Document AI API [GSP925]
### Task 1. Create and test a general form processor
- Enable the Cloud Document AI API
  - Navigation menu > APIs & services > Library > 
  - Cloud Document AI API > Enable
- Create a general form processor
  - Navigation menu > Document AI > Overview
  - Explore processors 
  - Create Processor [Form Parser]
  - region: US
  - Create
  - copy ID
### Task 2. Configure your Vertex AI Workbench instance to perform Document AI API calls
- Navigation > Vertex AI > Workbench > Open JupyterLab
- Terminal
```shell
gsutil cp notebook_files_path .
python -m pip install --upgrade google-cloud-core google-cloud-documentai google-cloud-storage prettytable 
gsutil cp form_path form.pdf
```
- open notebook > python 3
### Task 3. Make a synchronous process document request
- 

### Task 4. 
- set your processor id
- Run > Run selected Cell and All Below

### Task 5. Create a Document AI Document OCR Processor
- Navigation menu > Document AI > Overview
- Explore Processors > Create Processor [Document OCR]
- region: US
- Create
- Copy processor ID

### Prepare your environment for asynchronous Document AI API calls
- Terminal
```shell
export PROJECT_ID="$(gcloud config get-value core/project)"
export BUCKET="${PROJECT_ID}"_doc_ai_async
gsutil mb gs://${BUCKET}
gsutil -m cp async_files_path gs://${BUCKET}/input
```
- Open the JupyterLab > Python 3
### Task 7. Make an asynchronous process document request
- edit processor_id
- Run > Run Selected Cell and All Below
- File > Save Notebook

## Build and End-to-End Data Capture Pipeline using Document AI [GSP927]
### Task 1. Enable APIs and create an API key
```shell
gcloud services enable documentai.googleapis.com      
gcloud services enable cloudfunctions.googleapis.com  
gcloud services enable cloudbuild.googleapis.com    
gcloud services enable geocoding-backend.googleapis.com   
```
- Navigation menu > APIs & Serveices > Credentials
- Create credentials > API key
- three dots > Edit API key
- Restrict Key
- filter box: Geocoding API
- select Geocoding API > OK
- Save
### Task 2. Download the lab source code
```shell
mkdir ./documentai-pipeline-demo
gcloud storage cp -r \
  gs://spls/gsp927/documentai-pipeline-demo/* \
  ~/documentai-pipeline-demo/
```
### Task 3. create a form processor
- Navigation menu > Document AI > Explore Processors > Create Processor [Form Parser]
- region: US
- Create
- Copy ID
### Task 4. Create Cloud Storage buckets and a BigQuery dataset
- create cloud storage bucket
```shell
export PROJECT_ID=$(gcloud config get-value core/project)
export BUCKET_LOCATION="REGION"
gsutil mb -c standard -l ${BUCKET_LOCATION} -b on \
  gs://${PROJECT_ID}-input-invoices
gsutil mb -c standard -l ${BUCKET_LOCATION} -b on \
  gs://${PROJECT_ID}-output-invoices
gsutil mb -c standard -l ${BUCKET_LOCATION} -b on \
  gs://${PROJECT_ID}-archived-invoices
```
- create BigQuery dataset and tables
```shell
bq --location="US" mk  -d \
    --description "Form Parser Results" \
    ${PROJECT_ID}:invoice_parser_results
cd ~/documentai-pipeline-demo/scripts/table-schema/
bq mk --table \
  invoice_parser_results.doc_ai_extracted_entities \
  doc_ai_extracted_entities.json
bq mk --table \
  invoice_parser_results.geocode_details \
  geocode_details.json
```
- Create a Pub/Sub topic
```shell
export GEO_CODE_REQUEST_PUBSUB_TOPIC=geocode_request
gcloud pubsub topics \
  create ${GEO_CODE_REQUEST_PUBSUB_TOPIC}
```
### Task 5. Create Cloud Run functions
- create cloud run functions to process documents
```shell
gcloud storage service-agent --project=$PROJECT_ID

PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")

gcloud iam service-accounts create "service-$PROJECT_NUMBER" \
  --display-name "Cloud Storage Service Account" || true

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:service-$PROJECT_NUMBER@gs-project-accounts.iam.gserviceaccount.com" \
  --role="roles/pubsub.publisher"
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:service-$PROJECT_NUMBER@gs-project-accounts.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountTokenCreator"
```
- create the invoice processor cloud run function
```shell
cd ~/documentai-pipeline-demo/scripts
export CLOUD_FUNCTION_LOCATION="REGION"

gcloud functions deploy process-invoices \
--no-gen2 \
--region=${CLOUD_FUNCTION_LOCATION} \
--entry-point=process_invoice \
--runtime=python39 \
--source=cloud-functions/process-invoices \
--timeout=400 \
--env-vars-file=cloud-functions/process-invoices/.env.yaml \
--trigger-resource=gs://${PROJECT_ID}-input-invoices \
--trigger-event=google.storage.object.finalize
```
- create cloud run function to lookup geocode data from the addr
```shell
cd ~/documentai-pipeline-demo/scripts
gcloud functions deploy geocode-addresses \
--no-gen2 \
--region=${CLOUD_FUNCTION_LOCATION} \
--entry-point=process_address \
--runtime=python39 \
--source=cloud-functions/geocode-addresses \
--timeout=60 \
--env-vars-file=cloud-functions/geocode-addresses/.env.yaml \
--trigger-topic=${GEO_CODE_REQUEST_PUBSUB_TOPIC}
```
- Cloud Functions Sample Script Files
  - process-invoices-requirements.txt
```txt
# Function dependencies, for example:
# package>=version 
google-cloud-documentai>=0.3.0
google-cloud-storage>=1.30.0
google-cloud-bigquery>=1.26.1
google-cloud-pubsub>=1.7.0
```
  - process-invoices-main.py
```python
import base64
import re
import os
import json
from datetime import datetime
from google.cloud import bigquery
from google.cloud import documentai_v1beta3 as documentai
from google.cloud import storage
from google.cloud import pubsub_v1
 
#Reading environment variables
gcs_output_uri_prefix = os.environ.get('GCS_OUTPUT_URI_PREFIX')
project_id = os.environ.get('GCP_PROJECT')
location = os.environ.get('PARSER_LOCATION')
processor_id = os.environ.get('PROCESSOR_ID')
geocode_request_topicname = os.environ.get('GEOCODE_REQUEST_TOPICNAME')
timeout = int(os.environ.get('TIMEOUT'))
 
# An array of Future objects: environvery call to publish() returns an instance of Future
geocode_futures = []
# Setting variables
gcs_output_uri = f"gs://{project_id}-output-invoices"
gcs_archive_bucket_name = f"{project_id}-archived-invoices"
destination_uri = f"{gcs_output_uri}/{gcs_output_uri_prefix}/"
name = f"projects/{project_id}/locations/{location}/processors/{processor_id}"
dataset_name = 'invoice_parser_results'
table_name = 'doc_ai_extracted_entities'

# Create a dict to create the schema and to avoid BigQuery load job fails due to inknown fields
bq_schema={
    "input_file_name":"STRING",  
    "address":"STRING",
    "supplier":"STRING", 
    "invoice_number":"STRING", 
    "purchase_order":"STRING",
    "date":"STRING",
    "due_date":"STRING",
    "subtotal":"STRING", 
    "tax":"STRING",
    "total":"STRING"
}
bq_load_schema=[]
for key,value in bq_schema.items():
    bq_load_schema.append(bigquery.SchemaField(key,value))
 
docai_client = documentai.DocumentProcessorServiceClient()
storage_client = storage.Client()
bq_client = bigquery.Client()
pub_client = pubsub_v1.PublisherClient()
 
def write_to_bq(dataset_name, table_name, entities_extracted_dict):
 
    dataset_ref = bq_client.dataset(dataset_name)
    table_ref = dataset_ref.table(table_name)

    test_dict=entities_extracted_dict.copy()
    for key,value in test_dict.items():
      if key not in bq_schema:
          print ('Deleting key:' + key)
          del entities_extracted_dict[key]

    row_to_insert =[]
    row_to_insert.append(entities_extracted_dict)
 
    json_data = json.dumps(row_to_insert, sort_keys=False)
    #Convert to a JSON Object
    json_object = json.loads(json_data)
   
    job_config = bigquery.LoadJobConfig(schema=bq_load_schema)
    job_config.source_format = bigquery.SourceFormat.NEWLINE_DELIMITED_JSON
 
    job = bq_client.load_table_from_json(json_object, table_ref, job_config=job_config)
    error = job.result()  # Waits for table load to complete.
    print(error)

def get_text(doc_element: dict, document: dict):
    # Document AI identifies form fields by their offsets in document text. This function converts offsets to text snippets.
    response = ''
    # If a text segment spans several lines, it will be stored in different text segments.
    for segment in doc_element.text_anchor.text_segments:
        start_index = (
            int(segment.start_index)
            if segment in doc_element.text_anchor.text_segments
            else 0
        )
        end_index = int(segment.end_index)
        response += document.text[start_index:end_index]
    return response
 
def process_invoice(event, context):
    gcs_input_uri = 'gs://' + event['bucket'] + '/' + event['name']
    print('Printing the contentType: ' + event['contentType'])

    if(event['contentType'] == 'image/gif' or event['contentType'] == 'application/pdf' or event['contentType'] == 'image/tiff' ):
        input_config = documentai.types.document_processor_service.BatchProcessRequest.BatchInputConfig(gcs_source=gcs_input_uri, mime_type=event['contentType'])
        # Where to write results
        output_config = documentai.types.document_processor_service.BatchProcessRequest.BatchOutputConfig(gcs_destination=destination_uri)
 
        request = documentai.types.document_processor_service.BatchProcessRequest(
            name=name,
            input_configs=[input_config],
            output_config=output_config,
        )
 
        operation = docai_client.batch_process_documents(request)
 
        # Wait for the operation to finish
        operation.result(timeout=timeout)
 
        match = re.match(r"gs://([^/]+)/(.+)", destination_uri)
        output_bucket = match.group(1)
        prefix = match.group(2)
      
        #Get a pointer to the GCS bucket where the output will be placed
        bucket = storage_client.get_bucket(output_bucket)
      
        blob_list = list(bucket.list_blobs(prefix=prefix))
        print('Output files:')
 
        for i, blob in enumerate(blob_list):
            # Download the contents of this blob as a bytes object.
            if '.json' not in blob.name:
                print('blob name ' + blob.name)
                print(f"skipping non-supported file type {blob.name}")
            else:
                #Setting the output file name based on the input file name
                print('Fetching from ' + blob.name)
                start = blob.name.rfind("/") + 1
                end = blob.name.rfind(".") + 1           
                input_filename = blob.name[start:end:] + 'gif'
                print('input_filename ' + input_filename)
      
                # Getting ready to read the output of the parsed document - setting up "document"
                blob_as_bytes = blob.download_as_bytes()
                document = documentai.types.Document.from_json(blob_as_bytes)
      
                #Reading all entities into a dictionary to write into a BQ table
                entities_extracted_dict = {}
                entities_extracted_dict['input_file_name'] = input_filename
                for page in document.pages:
                    for form_field in page.form_fields:  
                        field_name = get_text(form_field.field_name,document)
                        field_value = get_text(form_field.field_value,document)
                        if field_name.strip().lower() == 'date:':
                            entities_extracted_dict['date'] = field_value
                        if field_name.strip().lower() == 'invoice number:':
                            entities_extracted_dict['invoice_number'] = field_value
                        if field_name.strip().lower() == 'purchase order:':
                            entities_extracted_dict['purchase_order'] = field_value
                        if field_name.strip().lower() == 'payment due by:':
                            entities_extracted_dict['due_date'] = field_value
                        if field_name.strip().lower() == 'subtotal':
                            entities_extracted_dict['subtotal'] = field_value
                        if field_name.strip().lower() == 'tax':
                            entities_extracted_dict['tax'] = field_value
                        if field_name.strip().lower() == 'total':
                            entities_extracted_dict['total'] = field_value

                        # # Creating and publishing a message via Pub Sub
                        if field_name.strip().lower() == 'address:':
                            message = {
                                "entity_type": field_name,
                                "entity_text" : field_value,
                                "input_file_name": input_filename,
                            }
                            message_data = json.dumps(message).encode("utf-8")
                            
                            entities_extracted_dict['address'] = field_value
                            geocode_topic_path = pub_client.topic_path(project_id,geocode_request_topicname)
                            geocode_future = pub_client.publish(geocode_topic_path, data = message_data)
                            geocode_futures.append(geocode_future)
                      
                print(entities_extracted_dict)
                print('Writing to BQ')
                #Write the entities to BQ
                write_to_bq(dataset_name, table_name, entities_extracted_dict)
                
        #print(blobs)
        #Deleting the intermediate files created by the Doc AI Parser
        blobs = bucket.list_blobs(prefix=gcs_output_uri_prefix)
        for blob in blobs:
            blob.delete()
        #Copy input file to archive bucket
        source_bucket = storage_client.bucket(event['bucket'])
        source_blob = source_bucket.blob(event['name'])
        destination_bucket = storage_client.bucket(gcs_archive_bucket_name)
        blob_copy = source_bucket.copy_blob(source_blob, destination_bucket, event['name'])
        #delete from the input folder
        source_blob.delete()
    else:
        print('Cannot parse the file type')


if __name__ == '__main__':
    print('Calling from main')
    testEvent={"bucket":project_id+"-input-invoices", "contentType": "application/pdf", "name":"invoice2.pdf"}
    testContext='test'
    process_invoice(testEvent,testContext)

```
  - process-invoices-requirements.txt
```txt
# Function dependencies, for example:
# package>=version
googlemaps==4.4.2
google-cloud-pubsub==1.7.0
requests==2.25.1
urllib3==1.26.2
google-cloud-bigquery==1.26.1
```
  - process-invoices-main.py
```python
import base64
import json
import os
import requests
from urllib.parse import urlencode
from google.cloud import bigquery

dataset_name = 'invoice_parser_results'
table_name = 'geocode_details'
my_schema = [
{
   "name": "input_file_name",
   "type": "STRING"
 },{
   "name": "entity_type",
   "type": "STRING"
 },{
   "name": "entity_text",
   "type": "STRING"
 },{
   "name": "place_id",
   "type": "STRING"
 },{
   "name": "formatted_address",
   "type": "STRING"
 },{
   "name": "lat",
   "type": "STRING"
 },{
   "name": "lng",
   "type": "STRING"
 }
]

bq_client = bigquery.Client()
 
def write_to_bq(dataset_name, table_name, geocode_response_dict, my_schema):
 
  dataset_ref = bq_client.dataset(dataset_name)
  table_ref = dataset_ref.table(table_name)
  row_to_insert =[]
  row_to_insert.append(geocode_response_dict)

  json_data = json.dumps(row_to_insert, sort_keys=False)
  #Convert to a JSON Object
  json_object = json.loads(json_data)
  print(json_object)
  job_config = bigquery.LoadJobConfig(schema = my_schema)
  job_config.source_format = bigquery.SourceFormat.NEWLINE_DELIMITED_JSON

  job = bq_client.load_table_from_json(json_object, table_ref, job_config=job_config)
  error = job.result()  # Waits for table load to complete.
  print(error)
  
 
def process_address(event, context):
  """Triggered from a message on a Cloud Pub/Sub topic.
  Args:
  event (dict): Event payload.
  context (google.cloud.functions.Context): Metadata for the event.
  """
  pubsub_message = base64.b64decode(event['data']).decode('utf-8')
  #print(type(pubsub_message))
  #print(pubsub_message)
  message_dict = json.loads(pubsub_message)
  query_address = message_dict.get('entity_text')
  geocode_dict = {} 
  geocode_dict["input_file_name"] = message_dict.get('input_file_name')
  geocode_dict["entity_type"] = message_dict.get('entity_type')
  geocode_dict["entity_text"] = query_address
  geocode_response_dict = extract_geocode_info(query_address)
  geocode_dict.update(geocode_response_dict)
  print(geocode_dict)

  write_to_bq(dataset_name, table_name, geocode_dict, my_schema)
  #print(geocode_response_dict)

# Using Geocoding API 
def extract_geocode_info(query_address,data_type='json'):
   geocode_response_dict = {} 
   endpoint =f"https://maps.googleapis.com/maps/api/geocode/{data_type}"
   API_key = os.environ.get('API_key')
   params ={"address": query_address, "key":API_key}
   url_params = urlencode(params)
   #print(url_params)
   url = f"{endpoint}?{url_params}"
   print(url)
   r = requests.get(url)
   print(r.status_code)
   if r.status_code not in range(200,299):
       print('status code not in range')
       return {}
   try:
        #print(message_dict.get('entity_type'))
        #print(message_dict.get('input_filename'))
        #print(query_address)
        #geocode_response_dict["entity_type"] = message_dict.get('entity_type')
        #geocode_response_dict["input_filename"] = message_dict.get('input_filename')
        #geocode_response_dict["entity_text"] = query_address
        
        geocode_response_dict["place_id"] = r.json()['results'][0]['place_id']
        geocode_response_dict["formatted_address"] = r.json()['results'][0]['formatted_address']
        geocode_response_dict["lat"] = str(r.json()['results'][0]['geometry']['location'].get("lat"))
        geocode_response_dict["lng"] = str(r.json()['results'][0]['geometry']['location'].get("lng"))
        print(geocode_response_dict)
   except:
       pass
   return geocode_response_dict

```
### Task 6. Edit environment variables for Cloud Run Functions
- Edit environment variables for the process-invoices Cloud Run function
  - search: Cloud Run functions
  - Go to Cloud Run functions 1st gen
  - process-invoices
  - Edit
  - Runtime, build, connections and security settings
  - Runtime environment variables: GCP_PROJECT; value: PROJECT_ID
  - Runtime environment variables: PROCESSOR_ID -> Invoice processor ID
  - Runtime environment variables: PARSER_LOCATION -> region of Invoice processor
  - Next
  - .evn.yaml
  - update PROCESSOR_ID, PARSER_LOCATION, GCP_PROJECT
  - Deploy
- Edit environment variables for the geocode-addresses Cloud Run function
  - geocode-addresses
  - Edit
  - Runtime, build, connections and security settings
  - Runtime environment variables: API_key
  - Next
  - .env.yaml
  - update API_key
  - Deploy
### Task 7. Test and validate the end-to-end solution
- upload sample forms
```shell
export PROJECT_ID=$(gcloud config get-value core/project)
gsutil cp gs://spls/gsp927/documentai-pipeline-demo/sample-files/* gs://${PROJECT_ID}-input-invoices/
```
  - search: Cloud Run Functions
  - process-invoices
  - Logs
- BigQuery
  - Navigation menu > BigQuery
  - expand invoice_parser_results
  - doc_ai_extracted_entities > Preview
  - geocode_details > Preview
## Automate Data Capture at Scale with Document AI: Challenge Lab [GSP367]
### Task 1. Enable the Cloud Document AI API and copy lab source files
- enable cloud document ai api
```shell
gcloud auth list

export PROJECT_ID=$(gcloud config get-value core/project)

gcloud services enable documentai.googleapis.com --project $DEVSHELL_PROJECT_ID
```
- copy lab source files
```
mkdir ./document-ai-challenge
gsutil -m cp -r gs://spls/gsp367/* \
  ~/document-ai-challenge/
```

### Task 2. Create a form processor
- Navigation menu > Document AI > Explore Processors > Create Processor [Form Parser]
- region: US
- Create
- Copy ID

### Task 3. Create Google Cloud resources
- Create input, output, and archive Cloud Storage buckets
```shell
export PROJECT_ID=$(gcloud config get-value core/project)
export BUCKET_LOCATION="us-west1"
gsutil mb -c standard -l ${BUCKET_LOCATION} -b on \
  gs://${PROJECT_ID}-input-invoices
gsutil mb -c standard -l ${BUCKET_LOCATION} -b on \
  gs://${PROJECT_ID}-output-invoices
gsutil mb -c standard -l ${BUCKET_LOCATION} -b on \
  gs://${PROJECT_ID}-archived-invoices
```
- Create a BigQuery dataset and tables
```shell
bq --location="US" mk  -d \
    --description "Form Parser Results" \
    ${PROJECT_ID}:invoice_parser_results
cd ~/document-ai-challenge/scripts/table-schema/
bq mk --table \
  invoice_parser_results.doc_ai_extracted_entities \
  doc_ai_extracted_entities.json
```

- Create a Pub/Sub topic
```shell
export GEO_CODE_REQUEST_PUBSUB_TOPIC=geocode_request
gcloud pubsub topics \
  create ${GEO_CODE_REQUEST_PUBSUB_TOPIC}
```

### Task 4. Deploy the document processing Cloud Run functions
```shell
cd ~/document-ai-challenge/scripts

PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects list --filter="project_id:$PROJECT_ID" --format='value(project_number)')

SERVICE_ACCOUNT=$(gcloud storage service-agent --project=$PROJECT_ID)

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member serviceAccount:$SERVICE_ACCOUNT \
  --role roles/pubsub.publisher

  export CLOUD_FUNCTION_LOCATION="REGION"
  gcloud functions deploy process-invoices \
  --gen2 \
  --region=${CLOUD_FUNCTION_LOCATION} \
  --entry-point=process_invoice \
  --runtime=python39 \
  --service-account=${PROJECT_ID}@appspot.gserviceaccount.com \
  --source=cloud-functions/process-invoices \
  --timeout=400 \
  --env-vars-file=cloud-functions/process-invoices/.env.yaml \
  --trigger-resource=gs://${PROJECT_ID}-input-invoices \
  --trigger-event=google.storage.object.finalize\
  --service-account $PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --allow-unauthenticated
```
- Edit environment variables for the process-invoices Cloud Run function
  - search: Cloud Run functions
  - if necessary[Go to Cloud Run functions 1st gen]
  - process-invoices
  - Edit
  - Variables & Secret
  - Runtime environment variables: GCP_PROJECT; value: PROJECT_ID
  - Runtime environment variables: PROCESSOR_ID -> Invoice processor ID
  - Runtime environment variables: PARSER_LOCATION -> region of Invoice processor

### Task 5. Test and validate the end-to-end solution
- upload sample forms
```shell
export PROJECT_ID=$(gcloud config get-value core/project)
gsutil cp ~/document-ai-challenge/invoices/* gs://${PROJECT_ID}-input-invoices/
```
  - search: Cloud Run Functions
  - process-invoices
  - Logs
- BigQuery
  - Navigation menu > BigQuery
  - expand invoice_parser_results
  - doc_ai_extracted_entities > Preview
  - geocode_details > Preview
