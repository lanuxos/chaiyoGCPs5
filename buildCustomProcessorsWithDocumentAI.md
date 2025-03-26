# Build Custom Processors with Document AI
## Optical Character Recognition (OCR) with Document AI (Python) [GSP1138]
### Task 1. Enable the Document AI API
```shell
gcloud services enable documentai.googleapis.com
gcloud services enable storage.googleapis.com
```
### Task 2. Create and test a processor
- Navigation menu > View all Products > Artificial Intelligence > Document AI
- Explore Processors
- Create Processor [Document OCR]
- name: lab-ocr
- Create
- Copy processor ID
- [Download PDF](https://storage.googleapis.com/cloud-samples-data/documentai/codelabs/ocr/Winnie_the_Pooh_3_Pages.pdf)
- UPload Test Document
### Task 3. Authentication API requests
```shell
LOCATION=us
PROCESSOR_ID=b49197a878119290

export PROJECT_ID=$(gcloud config get-value core/project)

gcloud iam service-accounts create my-docai-sa \
  --display-name "my-docai-service-account"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:my-docai-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/documentai.admin"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:my-docai-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/storage.admin"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:my-docai-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/serviceusage.serviceUsageConsumer"

gcloud iam service-accounts keys create ~/key.json \
  --iam-account  my-docai-sa@${PROJECT_ID}.iam.gserviceaccount.com

export GOOGLE_APPLICATION_CREDENTIALS=$(realpath key.json)
```
### Task 4. Install the client library
```shell
pip3 install --upgrade google-cloud-documentai
pip3 install --upgrade google-cloud-storage
```
- three dots > Upload
- File > Choose Files
- Upload
- OR JUST USING CMD
```shell
gcloud storage cp gs://cloud-samples-data/documentai/codelabs/ocr/Winnie_the_Pooh_3_Pages.pdf .
```
### Task 5. Make an online processing request
```shell
cat > online_processing.py << OEF
from google.api_core.client_options import ClientOptions
from google.cloud import documentai_v1 as documentai

# TODO:
PROJECT_ID = "qwiklabs-gcp-00-a948d4899db7"
LOCATION = "us"  # Format is 'us' or 'eu'
PROCESSOR_ID = "b49197a878119290"  # Create processor in Cloud Console

# The local file in your current working directory
FILE_PATH = "Winnie_the_Pooh_3_Pages.pdf"
# Refer to https://cloud.google.com/document-ai/docs/file-types
# for supported file types
MIME_TYPE = "application/pdf"

# Instantiates a client
docai_client = documentai.DocumentProcessorServiceClient(
    client_options=ClientOptions(api_endpoint=f"{LOCATION}-documentai.googleapis.com")
)

# The full resource name of the processor, e.g.:
# projects/project-id/locations/location/processor/processor-id
# You must create new processors in the Cloud Console first
RESOURCE_NAME = docai_client.processor_path(PROJECT_ID, LOCATION, PROCESSOR_ID)

# Read the file into memory
with open(FILE_PATH, "rb") as image:
    image_content = image.read()

# Load Binary Data into Document AI RawDocument Object
raw_document = documentai.RawDocument(content=image_content, mime_type=MIME_TYPE)

# Configure the process request
request = documentai.ProcessRequest(name=RESOURCE_NAME, raw_document=raw_document)

# Use the Document AI client to process the sample form
result = docai_client.process_document(request=request)

document_object = result.document
print("Document processing complete.")
print(f"Text: {document_object.text}")
OEF

python3 online_processing.py
```
### Task 6. Make a batch processing request
```shell
gcloud storage buckets create gs://$PROJECT_ID
gcloud storage cp gs://cloud-samples-data/documentai/codelabs/ocr/Winnie_the_Pooh.pdf gs://$PROJECT_ID/

cat > batch_processing.py << EOF
import re
from typing import List

from google.api_core.client_options import ClientOptions
from google.cloud import documentai_v1 as documentai
from google.cloud import storage

#TODO:
PROJECT_ID = "qwiklabs-gcp-00-a948d4899db7"
LOCATION = "us"  # Format is 'us' or 'eu'
PROCESSOR_ID = "b49197a878119290"  # Create processor in Cloud Console

# Format 'gs://input_bucket/directory/file.pdf'
GCS_INPUT_URI = "gs://cloud-samples-data/documentai/codelabs/ocr/Winnie_the_Pooh.pdf"
INPUT_MIME_TYPE = "application/pdf"

# Format 'gs://output_bucket/directory'
GCS_OUTPUT_URI = "gs://qwiklabs-gcp-00-a948d4899db7"

# Instantiates a client
docai_client = documentai.DocumentProcessorServiceClient(
    client_options=ClientOptions(api_endpoint=f"{LOCATION}-documentai.googleapis.com")
)

# The full resource name of the processor, e.g.:
# projects/project-id/locations/location/processor/processor-id
# You must create new processors in the Cloud Console first
RESOURCE_NAME = docai_client.processor_path(PROJECT_ID, LOCATION, PROCESSOR_ID)

# Cloud Storage URI for the Input Document
input_document = documentai.GcsDocument(
    gcs_uri=GCS_INPUT_URI, mime_type=INPUT_MIME_TYPE
)

# Load GCS Input URI into a List of document files
input_config = documentai.BatchDocumentsInputConfig(
    gcs_documents=documentai.GcsDocuments(documents=[input_document])
)

# Cloud Storage URI for Output directory
gcs_output_config = documentai.DocumentOutputConfig.GcsOutputConfig(
    gcs_uri=GCS_OUTPUT_URI
)

# Load GCS Output URI into OutputConfig object
output_config = documentai.DocumentOutputConfig(gcs_output_config=gcs_output_config)

# Configure Process Request
request = documentai.BatchProcessRequest(
    name=RESOURCE_NAME,
    input_documents=input_config,
    document_output_config=output_config,
)

# Batch Process returns a Long Running Operation (LRO)
operation = docai_client.batch_process_documents(request)

# Continually polls the operation until it is complete.
# This could take some time for larger files
# Format: projects/PROJECT_NUMBER/locations/LOCATION/operations/OPERATION_ID
print(f"Waiting for operation {operation.operation.name} to complete...")
operation.result()

# NOTE: Can also use callbacks for asynchronous processing
#
# def my_callback(future):
#   result = future.result()
#
# operation.add_done_callback(my_callback)

print("Document processing complete.")

# Once the operation is complete,
# get output document information from operation metadata
metadata = documentai.BatchProcessMetadata(operation.metadata)

if metadata.state != documentai.BatchProcessMetadata.State.SUCCEEDED:
    raise ValueError(f"Batch Process Failed: {metadata.state_message}")

documents: List[documentai.Document] = []

# Storage Client to retrieve the output files from GCS
storage_client = storage.Client()

# One process per Input Document
for process in metadata.individual_process_statuses:

    # output_gcs_destination format: gs://BUCKET/PREFIX/OPERATION_NUMBER/0
    # The GCS API requires the bucket name and URI prefix separately
    output_bucket, output_prefix = re.match(
        r"gs://(.*?)/(.*)", process.output_gcs_destination
    ).groups()

    # Get List of Document Objects from the Output Bucket
    output_blobs = storage_client.list_blobs(output_bucket, prefix=output_prefix)

    # DocAI may output multiple JSON files per source file
    for blob in output_blobs:
        # Document AI should only output JSON files to GCS
        if ".json" not in blob.name:
            print(f"Skipping non-supported file type {blob.name}")
            continue

        print(f"Fetching {blob.name}")

        # Download JSON File and Convert to Document Object
        document = documentai.Document.from_json(
            blob.download_as_bytes(), ignore_unknown_fields=True
        )

        documents.append(document)

# Print Text from all documents
# Truncated at 100 characters for brevity
for document in documents:
    print(document.text[:100])
EOF

python3 batch_processing.py
```
### Task 7. Make a batch processing request for a directory
```shell
gsutil -m cp -r gs://cloud-samples-data/documentai/codelabs/ocr/multi-document/* gs://$PROJECT_ID/multi-document/

cat > batch_processing_directory.py << EOF
import re
from typing import List

from google.api_core.client_options import ClientOptions
from google.cloud import documentai_v1 as documentai
from google.cloud import storage

# TODO:
PROJECT_ID = "qwiklabs-gcp-00-a948d4899db7"
LOCATION = "us"  # Format is 'us' or 'eu'
PROCESSOR_ID = "b49197a878119290"  # Create processor in Cloud Console

# Format 'gs://input_bucket/directory'
GCS_INPUT_PREFIX = "gs://cloud-samples-data/documentai/codelabs/ocr/multi-document"

# Format 'gs://output_bucket/directory'
GCS_OUTPUT_URI = "gs://qwiklabs-gcp-00-a948d4899db7"

# Instantiates a client
docai_client = documentai.DocumentProcessorServiceClient(
    client_options=ClientOptions(api_endpoint=f"{LOCATION}-documentai.googleapis.com")
)

# The full resource name of the processor, e.g.:
# projects/project-id/locations/location/processor/processor-id
# You must create new processors in the Cloud Console first
RESOURCE_NAME = docai_client.processor_path(PROJECT_ID, LOCATION, PROCESSOR_ID)

# Cloud Storage URI for the Input Directory
gcs_prefix = documentai.GcsPrefix(gcs_uri_prefix=GCS_INPUT_PREFIX)

# Load GCS Input URI into Batch Input Config
input_config = documentai.BatchDocumentsInputConfig(gcs_prefix=gcs_prefix)

# Cloud Storage URI for Output directory
gcs_output_config = documentai.DocumentOutputConfig.GcsOutputConfig(
    gcs_uri=GCS_OUTPUT_URI
)

# Load GCS Output URI into OutputConfig object
output_config = documentai.DocumentOutputConfig(gcs_output_config=gcs_output_config)

# Configure Process Request
request = documentai.BatchProcessRequest(
    name=RESOURCE_NAME,
    input_documents=input_config,
    document_output_config=output_config,
)

# Batch Process returns a Long Running Operation (LRO)
operation = docai_client.batch_process_documents(request)

# Continually polls the operation until it is complete.
# This could take some time for larger files
# Format: projects/PROJECT_NUMBER/locations/LOCATION/operations/OPERATION_ID
print(f"Waiting for operation {operation.operation.name} to complete...")
operation.result()

# NOTE: Can also use callbacks for asynchronous processing
#
# def my_callback(future):
#   result = future.result()
#
# operation.add_done_callback(my_callback)

print("Document processing complete.")

# Once the operation is complete,
# get output document information from operation metadata
metadata = documentai.BatchProcessMetadata(operation.metadata)

if metadata.state != documentai.BatchProcessMetadata.State.SUCCEEDED:
    raise ValueError(f"Batch Process Failed: {metadata.state_message}")

documents: List[documentai.Document] = []

# Storage Client to retrieve the output files from GCS
storage_client = storage.Client()

# One process per Input Document
for process in metadata.individual_process_statuses:

    # output_gcs_destination format: gs://BUCKET/PREFIX/OPERATION_NUMBER/0
    # The GCS API requires the bucket name and URI prefix separately
    output_bucket, output_prefix = re.match(
        r"gs://(.*?)/(.*)", process.output_gcs_destination
    ).groups()

    # Get List of Document Objects from the Output Bucket
    output_blobs = storage_client.list_blobs(output_bucket, prefix=output_prefix)

    # DocAI may output multiple JSON files per source file
    for blob in output_blobs:
        # Document AI should only output JSON files to GCS
        if ".json" not in blob.name:
            print(f"Skipping non-supported file type {blob.name}")
            continue

        print(f"Fetching {blob.name}")

        # Download JSON File and Convert to Document Object
        document = documentai.Document.from_json(
            blob.download_as_bytes(), ignore_unknown_fields=True
        )

        documents.append(document)

# Print Text from all documents
# Truncated at 100 characters for brevity
for document in documents:
    print(document.text[:100])
EOF

python3 batch_processing_directory.py
```

## From Parsing with Document AI (Python) [GSP1139]
### Task 1. Enable the Document AI API
```shell
gcloud services enable documentai.googleapis.com

pip3 install --upgrade pandas

pip3 install --upgrade google-cloud-documentai
```
### Task 2. Create a Form Parser processor
- Navigation menu > View All Products > Artificial Intelligence > Document AI
- Explore Processors
- Create Processor [Form Parser]
- name: lab-form-parser
- Create
- Copy Processor ID
- Upload Test Document
### Task 3. Download the sample form
```shell
gcloud storage cp gs://cloud-samples-data/documentai/codelabs/form-parser/intake-form.pdf .
ls intake-form.pdf
```
### Task 4. Extract form key/value pairs
```shell
cat > form_parser.py << EOF
import pandas as pd
from google.cloud import documentai_v1 as documentai


def online_process(
    project_id: str,
    location: str,
    processor_id: str,
    file_path: str,
    mime_type: str,
) -> documentai.Document:
    """
    Processes a document using the Document AI Online Processing API.
    """

    opts = {"api_endpoint": f"{location}-documentai.googleapis.com"}

    # Instantiates a client
    documentai_client = documentai.DocumentProcessorServiceClient(client_options=opts)

    # The full resource name of the processor, e.g.:
    # projects/project-id/locations/location/processor/processor-id
    # You must create new processors in the Cloud Console first
    resource_name = documentai_client.processor_path(project_id, location, processor_id)

    # Read the file into memory
    with open(file_path, "rb") as image:
        image_content = image.read()

        # Load Binary Data into Document AI RawDocument Object
        raw_document = documentai.RawDocument(
            content=image_content, mime_type=mime_type
        )

        # Configure the process request
        request = documentai.ProcessRequest(
            name=resource_name, raw_document=raw_document
        )

        # Use the Document AI client to process the sample form
        result = documentai_client.process_document(request=request)

        return result.document


def trim_text(text: str):
    """
    Remove extra space characters from text (blank, newline, tab, etc.)
    """
    return text.strip().replace("\n", " ")


PROJECT_ID = "qwiklabs-gcp-01-eeba988577f4"
LOCATION = "us"  # Format is 'us' or 'eu'
PROCESSOR_ID = "23abff0f8f00e5ff"  # Create processor in Cloud Console

# The local file in your current working directory
FILE_PATH = "intake-form.pdf"
# Refer to https://cloud.google.com/document-ai/docs/processors-list
# for supported file types
MIME_TYPE = "application/pdf"

document = online_process(
    project_id=PROJECT_ID,
    location=LOCATION,
    processor_id=PROCESSOR_ID,
    file_path=FILE_PATH,
    mime_type=MIME_TYPE,
)

names = []
name_confidence = []
values = []
value_confidence = []

for page in document.pages:
    for field in page.form_fields:
        # Get the extracted field names
        names.append(trim_text(field.field_name.text_anchor.content))
        # Confidence - How "sure" the Model is that the text is correct
        name_confidence.append(field.field_name.confidence)

        values.append(trim_text(field.field_value.text_anchor.content))
        value_confidence.append(field.field_value.confidence)

# Create a Pandas Dataframe to print the values in tabular format.
df = pd.DataFrame(
    {
        "Field Name": names,
        "Field Name Confidence": name_confidence,
        "Field Value": values,
        "Field Value Confidence": value_confidence,
    }
)

print(df)
EOF

python3 form_parser.py
```
### Task 5. Parse tables
```shell
gcloud storage cp gs://cloud-samples-data/documentai/codelabs/form-parser/form_with_tables.pdf .

ls form_with_tables.pdf

cat > table_parsing.py << EOF
# type: ignore[1]
"""
Uses Document AI online processing to call a form parser processor
Extracts the tables and data in the document.
"""
from os.path import splitext
from typing import List, Sequence

import pandas as pd
from google.cloud import documentai


def online_process(
    project_id: str,
    location: str,
    processor_id: str,
    file_path: str,
    mime_type: str,
) -> documentai.Document:
    """
    Processes a document using the Document AI Online Processing API.
    """

    opts = {"api_endpoint": f"{location}-documentai.googleapis.com"}

    # Instantiates a client
    documentai_client = documentai.DocumentProcessorServiceClient(client_options=opts)

    # The full resource name of the processor, e.g.:
    # projects/project-id/locations/location/processor/processor-id
    # You must create new processors in the Cloud Console first
    resource_name = documentai_client.processor_path(project_id, location, processor_id)

    # Read the file into memory
    with open(file_path, "rb") as image:
        image_content = image.read()

        # Load Binary Data into Document AI RawDocument Object
        raw_document = documentai.RawDocument(
            content=image_content, mime_type=mime_type
        )

        # Configure the process request
        request = documentai.ProcessRequest(
            name=resource_name, raw_document=raw_document
        )

        # Use the Document AI client to process the sample form
        result = documentai_client.process_document(request=request)

        return result.document


def get_table_data(
    rows: Sequence[documentai.Document.Page.Table.TableRow], text: str
) -> List[List[str]]:
    """
    Get Text data from table rows
    """
    all_values: List[List[str]] = []
    for row in rows:
        current_row_values: List[str] = []
        for cell in row.cells:
            current_row_values.append(
                text_anchor_to_text(cell.layout.text_anchor, text)
            )
        all_values.append(current_row_values)
    return all_values


def text_anchor_to_text(text_anchor: documentai.Document.TextAnchor, text: str) -> str:
    """
    Document AI identifies table data by their offsets in the entirety of the
    document's text. This function converts offsets to a string.
    """
    response = ""
    # If a text segment spans several lines, it will
    # be stored in different text segments.
    for segment in text_anchor.text_segments:
        start_index = int(segment.start_index)
        end_index = int(segment.end_index)
        response += text[start_index:end_index]
    return response.strip().replace("\n", " ")


PROJECT_ID = "YOUR_PROJECT_ID"
LOCATION = "YOUR_PROJECT_LOCATION"  # Format is 'us' or 'eu'
PROCESSOR_ID = "FORM_PARSER_ID"  # Create processor before running sample

# The local file in your current working directory
FILE_PATH = "form_with_tables.pdf"
# Refer to https://cloud.google.com/document-ai/docs/file-types
# for supported file types
MIME_TYPE = "application/pdf"

document = online_process(
    project_id=PROJECT_ID,
    location=LOCATION,
    processor_id=PROCESSOR_ID,
    file_path=FILE_PATH,
    mime_type=MIME_TYPE,
)

header_row_values: List[List[str]] = []
body_row_values: List[List[str]] = []

# Input Filename without extension
output_file_prefix = splitext(FILE_PATH)[0]

for page in document.pages:
    for index, table in enumerate(page.tables):
        header_row_values = get_table_data(table.header_rows, document.text)
        body_row_values = get_table_data(table.body_rows, document.text)

        # Create a Pandas Dataframe to print the values in tabular format.
        df = pd.DataFrame(
            data=body_row_values,
            columns=pd.MultiIndex.from_arrays(header_row_values),
        )

        print(f"Page {page.page_number} - Table {index}")
        print(df)

        # Save each table as a CSV file
        output_filename = f"{output_file_prefix}_pg{page.page_number}_tb{index}.csv"
        df.to_csv(output_filename, index=False)
EOF

python3 table_parsing.py
```

## Using Specialized Processors with Document AI (Python) [GSP1140]
### Task 1. Enable the Document AI API
```shell
gcloud services enable documentai.googleapis.com

pip3 install --upgrade pandas

pip3 install --upgrade google-cloud-documentai
```
### Task 2. Create a Form Parser processor
- Navigation menu > VIEW ALL PRODUCTS > Artificial Intelligence > Document AI
- Explore Processor
- Specialized
- Invoice Parser
- Create Processor
- name: lab-invoice-parser
- Create
- Copy Processor ID
```shell
gcloud storage cp gs://cloud-samples-data/documentai/codelabs/specialized-processors/procurement_multi_document.pdf .

gcloud storage cp gs://cloud-samples-data/documentai/codelabs/specialized-processors/google_invoice.pdf .
```
### Task 3. Extract the entities
- Invoice Parser
```shell
cat > extraction.py << EOF
import pandas as pd
from google.cloud import documentai_v1 as documentai


def online_process(
    project_id: str,
    location: str,
    processor_id: str,
    file_path: str,
    mime_type: str,
) -> documentai.Document:
    """
    Processes a document using the Document AI Online Processing API.
    """

    opts = {"api_endpoint": f"{location}-documentai.googleapis.com"}

    # Instantiates a client
    documentai_client = documentai.DocumentProcessorServiceClient(client_options=opts)

    # The full resource name of the processor, e.g.:
    # projects/project-id/locations/location/processor/processor-id
    # You must create new processors in the Cloud Console first
    resource_name = documentai_client.processor_path(project_id, location, processor_id)

    # Read the file into memory
    with open(file_path, "rb") as file:
        file_content = file.read()

    # Load Binary Data into Document AI RawDocument Object
    raw_document = documentai.RawDocument(content=file_content, mime_type=mime_type)

    # Configure the process request
    request = documentai.ProcessRequest(name=resource_name, raw_document=raw_document)

    # Use the Document AI client to process the sample form
    result = documentai_client.process_document(request=request)

    return result.document


PROJECT_ID = "qwiklabs-gcp-04-6ac538ff2d11"
LOCATION = "us"  # Format is 'us' or 'eu'
PROCESSOR_ID = "86008372dce04ba3"  # Create processor in Cloud Console

# The local file in your current working directory
FILE_PATH = "google_invoice.pdf"
# Refer to https://cloud.google.com/document-ai/docs/processors-list
# for supported file types
MIME_TYPE = "application/pdf"

document = online_process(
    project_id=PROJECT_ID,
    location=LOCATION,
    processor_id=PROCESSOR_ID,
    file_path=FILE_PATH,
    mime_type=MIME_TYPE,
)

types = []
raw_values = []
normalized_values = []
confidence = []

# Grab each key/value pair and their corresponding confidence scores.
for entity in document.entities:
    types.append(entity.type_)
    raw_values.append(entity.mention_text)
    normalized_values.append(entity.normalized_value.text)
    confidence.append(f"{entity.confidence:.0%}")

    # Get Properties (Sub-Entities) with confidence scores
    for prop in entity.properties:
        types.append(prop.type_)
        raw_values.append(prop.mention_text)
        normalized_values.append(prop.normalized_value.text)
        confidence.append(f"{prop.confidence:.0%}")

# Create a Pandas Dataframe to print the values in tabular format.
df = pd.DataFrame(
    {
        "Type": types,
        "Raw Value": raw_values,
        "Normalized Value": normalized_values,
        "Confidence": confidence,
    }
)

print(df)
EOF

python3 extraction.py

# Create a bucket
export PROJECT_ID=$(gcloud config get-value project)
gsutil mb gs://$PROJECT_ID-docai

# Create and upload the file
python3 extraction.py > docai_outputs.txt
gsutil cp docai_outputs.txt gs://$PROJECT_ID-docai
```

## Uptraining with Document AI Workbench [GSP1141]
### Task 1. Enable the Document AI API
```shell
gcloud services enable documentai.googleapis.com

pip3 install --upgrade google-cloud-documentai
```
### Task 2. Create a processor
- Navigation menu > Artificial Intelligence > Document AI
- Explore Processors
- Specialized 
- Invoice Parser
- Create Processor
- name: lab-invoice-uptraining [us]
- Create
### Task 3. Create a dataset
```shell
export PROJECT_ID=$(gcloud config get-value project)
gcloud storage buckets create "gs://${PROJECT_ID}-uptraining-lab" --location=Region
```
- Processor Details
- Dataset
- Configure Your Dataset
- Show Advanced Options
- I'll specify my own storage location
- paste bucket name create earlier
- Continue
- Create Dataset
### Task 4. Import a test document
- Train
- Import Documents
- Source path: cloud-samples-data/documentai/codelabs/uptraining/pdfs
- Import
### Task 5. Label the test document
- Select Text tool
- highlight: MacWilliam Piping International
- assign label: supplier_name
- highlight: 14368 Pipeline Ave Chino, CA 91710
- assign label: supplier_address
- highlight: 10001
- assign label: invoice_id
- highlight: 2020-01-20
- assign label: due_date
- Switch to Bounding Box tool
- highlight: Knuckle Couplers
- assign label:line_item/description
- highlight: 9
- assign label: line_item/quantity
- assign it to: line_item (Knuckle Couplers)
- highlight: 74.43
- assign label: line_item/unit_price
- assign it to: line_item (Knuckle Couplers)
- highlight: 669.87
- assign label: label_item/amount
- assign it to: line_item (Knuckle Couplers)
- on the left side menu: Add more rows
- Use the Bounding Box tool to highlight the entire second row
- repeat for the third row
- Highlight the text 1,419.57
- assign label: net_amount
- hightlight the text   113.57
- assign label: total_tax_amount
- hightlight the text   1,533.14
- assign label: total_amount
- hightlight the $
- assign label: currency
### Task 6. Assign document to training set
- select document
- Assign to Set
- Training
### Task 7. Import pre-labeled data
- in the training management console
- Import Documents
- Source path: cloud-samples-data/documentai/Custom/Invoices/JSON
- Auto-split
- Import
### Task 8. Edit labels
- Edit Schema
- Enable: 
  - invoice_date 
  - line_item > amount
  - line_item > description
  - receiver_address
  - receiver_name
  - supplier_address
  - supplier_name
  - total_amount
- Save
- Save
### Task 9. Auto-label newly imported documents
- Import Document
```shell
cloud-samples-data/documentai/Custom/Invoices/PDF_Unlabeled
```
- Auto-labeling: Import with auto-labeling
- Import
- verify/Mark as labeled
### Task 10. Uptrain the Model
- Uptrain New Version
- name: lab-uptraining-test-1
- Start Training

## Custom Document Extraction with Document AI Workbench [GSP1142]
### Task 1. Enable the Document AI API
```shell
gcloud services enable documentai.googleapis.com
pip3 install --upgrade google-cloud-documentai
```
### Task 2. Create a processor
- Navigation menu > View All Products > Artificial Intelligence > Document AI
- Create Custom Processor
- Custom Extractor
- Create Processor
- name: lab-custom-extractor [us]
- create
### Task 3. Define processor fields
- Get started 
- Fields
- Create New Field
- Name: Data type: Occurrence > Create
  - control_number	                    Number	Optional multiple
  - employees_social_security_number	Number	Required multiple
  - employer_identification_number	    Number	Required multiple
  - employers_name_address_and_zip_code	Address	Required multiple
  - federal_income_tax_withheld	        Money	Required multiple
  - social_security_tax_withheld	    Money	Required multiple
  - social_security_wages	            Money	Required multiple
  - wages_tips_other_compensation	    Money	Required multiple

### Task 4. Upload a sample document
- Upload Sample Document
- Import documents from Google Cloud Storage
```
cloud-samples-data/documentai/Custom/W2/PDF/W2_XL_input_clean_2950.pdf
```
- Import
### Task 5. Label a document

### Task 6. Build processor version using foundation model
ຶ້- Build
- Call foundation model
- Create New Version
- name: w2-foundation-model
- Create
- Deploy & Use
### Task 7. Use generative AI to auto-label documents
- Build
- Import Documents
- Import documents from Google Cloud Storage
- Source path: cloud-samples-data/documentai/Custom/W2/AutoLabel
- Data split: Auto-split
- Auto-labeling: Import with auto-labeling
- Import
- Start Labeling
- Mark as Labeled
### Task 8. Import prelabeled training document
- Build
- Import Documents
- Import ducuments from Google Cloud Storage
- Source path: cloud-samples-data/documentai/Custom/W2/JSON-2
- Data split: Auto-split
- Import
### Task 9. Train the processor
- Train a custom model
- Create New Version
- name: w2-custom-model
- Model training method: Model based
- Start training

## Build Custom Processors with Document AI: Challenge Lab [GSP513]
### Task 1. Enable the Document AI API
```shell
gcloud services enable documentai.googleapis.com
pip3 install --upgrade google-cloud-documentai
```
### Task 2. Create a processor
- Navigation menu > View All Products > Artificial Intelligence > Document AI
- Create Custom Processor
- Custom Extractor
- Create Processor
- name: custom-document-extractor-egec [us]
- create

### Task 3. Define processor fields
- Get started 
- Fields
- Create New Field
- Name: Data type: Occurrence > Create
  - applicant_line_1    Plain Text	Required once
  - application_number	Number	    Required once
  - class_international	Plain Text	Required once
  - class_us	        Plain Text	Required once
  - filing_date	        Datetime	Required once
  - inventor_line_1	    Plain Text	Required once
  - issuer	            Plain Text	Required once
  - patent_number	    Number	    Required once
  - publication_date	Datetime	Required once
  - title_line_1	    Plain Text	Required once
### Task 4. Import a document into a dataset
- Upload Sample Document
```
cloud-samples-data/documentai/codelabs/challenge/unlabeled/us_001.pdf
```
- Import
### Task 5. Label a document
- us_001
  - applicant_line_1 = Colby Green
  - application_number = 679,694
  - class_international = H04W 64/00
  - class_us = H04W 64/003
  - filing_date = Aug. 17, 2017
  - inventor_line_1 = Colby Green
  - issuer = US
  - patent_number = 10,136,408
  - publication_date = Nov. 20, 2018
  - title_line_1 = DETERMINING HIGH VALUE
### Task 6. Assign an annotated document to the training set
- select us_001 document
- Assign to Set
- Training
### Task 7. Import pre-labeled data to the training and test sets
- Build
- Import Documents
- Import ducuments from Google Cloud Storage
- Source path: cloud-samples-data/documentai/codelabs/challenge/labeled
- Data split: Auto-split
- Import
### Task 8. Train the processor
### Task 9. Train the processor
- Train a custom model
- Create New Version
- name: my-cde-version-1
- Model training method: Model based
- Start training

