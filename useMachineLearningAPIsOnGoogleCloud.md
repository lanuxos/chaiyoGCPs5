# Use Machine Learning APIs on Google Cloud
## Introduction to APIs in Google Cloud [GSP294]
### Task 1. Using the API library
- 
### Task 2. Creating JSON File in the Cloud Console
```shell
cat <<EOF > values.json
{  "name": "qwiklabs-gcp-00-6f16a7de92b8-bucket",
   "location": "us",
   "storageClass": "multi_regional"
}
EOF
```
### Task 3. Authenticate and authorize the Cloud Storage JSON/REST API
- [OAuth 2.0 playground](https://developers.google.com/oauthplayground/)
- https://www.googleapis.com/auth/devstorage.full_control 
- Authorize APIs
- Allow
- Exchange authorization code for tokens.
- Copy
### Task 4. Create a bucket with the Cloud Storage JSON/REST API
```shell
export OAUTH2_TOKEN=ya29.a0AeXRPp46qobxXJ03Uq6x7kmquqQaE9ncCskYqPlE0gEYDAS6j37klOc_9hl8rvkbwiPJJuByKHuwHFv0lgPZV_9LfbKQJZW5PkGO3-B07dgXESlMHobCq9chjnYHFUVlfiQS3Z97oR7XxKRtZcrEozto88Xs5KCBJiZhHlXVaCgYKAaQSARISFQHGX2MiOqz3mJqh9o4tbE1OoT1TMA0175
export PROJECT_ID=$(gcloud config get-value project)
curl -X POST --data-binary @values.json     -H "Authorization: Bearer $OAUTH2_TOKEN"     -H "Content-Type: application/json"     "https://www.googleapis.com/storage/v1/b?project=$PROJECT_ID"
```
### Task 5. Upload a file using the Cloud Storage JSON/REST API
- save demo-image.png and upload to cloud storage
```shell
realpath demo-image.png
export OBJECT=/home/student_02_e9c2c7321e4d/demo-image.png
export BUCKET_NAME=qwiklabs-gcp-00-6f16a7de92b8-bucket 
curl -X POST --data-binary @$OBJECT     -H "Authorization: Bearer $OAUTH2_TOKEN"     -H "Content-Type: image/png"     "https://www.googleapis.com/upload/storage/v1/b/$BUCKET_NAME/o?uploadType=media&name=demo-image"
```

## Extract, Analyze, and Translate Text From Images with the Cloud ML APIs [GSP075]
### Task 1. Create an API key
- Navigation Menu > APIs & Services > Credentials
- + Create Credentials
- API Key > Copy > Close
```shell
export API_KEY=
```

### Task 2. Upload an image to a Cloud Storage bucket
- Navigation Menu > Cloud Storage > Create bucket
- Choose how to control access to objects
- Enforce public access prevention on this bucket
- Fine-grained
- Create
- upload sign.jpg to bucket
- three dors > Edit access
- Add Entry
- Public: allUsers/ Reader
- Save

### Task 3. Create your Cloud Vision API request
```shell
cat > orc-request.json << EOF
{
  "requests": [
      {
        "image": {
          "source": {
              "gcsImageUri": "gs://my-bucket-name/sign.jpg"
          }
        },
        "features": [
          {
            "type": "TEXT_DETECTION",
            "maxResults": 10
          }
        ]
      }
  ]
}
EOF
```
 
### Task 4. Call the text detection method
```shell
curl -s -X POST -H "Content-Type: application/json" --data-binary @ocr-request.json  https://vision.googleapis.com/v1/images:annotate?key=${API_KEY}

curl -s -X POST -H "Content-Type: application/json" --data-binary @ocr-request.json  https://vision.googleapis.com/v1/images:annotate?key=${API_KEY} -o ocr-response.json
```

### Task 5. Send text from the image to the Translation API
```shell
cat > translation-request.json << EOF
{
  "q": "your_text_here",
  "target": "en"
}
EOF

STR=$(jq .responses[0].textAnnotations[0].description ocr-response.json) && STR="${STR//\"}" && sed -i "s|your_text_here|$STR|g" translation-request.json

curl -s -X POST -H "Content-Type: application/json" --data-binary @translation-request.json https://translation.googleapis.com/language/translate/v2?key=${API_KEY} -o translation-response.json

cat translation-response.json
```

### Task 6. Analyzing the image's text with the Natural Language API
```shell
cat > request.json << EOF
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"your_text_here"
  },
  "encodingType":"UTF8"
}
EOF

STR=$(jq .data.translations[0].translatedText  translation-response.json) && STR="${STR//\"}" && sed -i "s|your_text_here|$STR|g" nl-request.json

curl "https://language.googleapis.com/v1/documents:analyzeEntities?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @nl-request.json
```

## Classify Text into Categories with the Natural Language [GSP063]
### Task 1. Enable the Cloud Natural Language API
- Navigation Menu > APIs & Services > Enabled APIs and Services
- Enable APIs and Services > Cloud Natural Language API
- Enable

### Task 2. Create an API Key
- Navigation Menu > API & Services > Credentials
- API key
- Copy & Close
- Navigation Menu > Compute Engine > VM Instances > SSH
```shell
export API_KEY=
```

### Task 3. Classify a news article
- create request.json file
```shell
cat > request.json << EOF
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"A Smoky Lobster Salad With a Tapa Twist. This spin on the Spanish pulpo a la gallega skips the octopus, but keeps the sea salt, olive oil, pimentón and boiled potatoes."
  }
}
EOF

curl "https://language.googleapis.com/v1/documents:classifyText?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json

curl "https://language.googleapis.com/v1/documents:classifyText?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json > result.json
```
### Task 4. Classify a large text dataset
- view one article on google cloud
```shell
gsutil cat gs://spls/gsp063/bbc_dataset/entertainment/001.txt
```
### Task 5. Create a BigQuery table for categorized text data
- Navigation Menu > BigQuery > Done
- three dots: Create dataset [news_classification_dataset]
- Create dataset
- three dots: Create Table [article_datat]
  - create table from: Empty table
  - Schema: Add Field
    - Field Name: article_text; Type: STRING; Mode: NULLABLE
    - Field Name: category; Type: STRING; Mode: NULLABLE
    - Field Name: confidence; Type: FLOAT; Mode: NULLABLE
- Create Table

### Task 6. Classify news data and store the result in BigQuery
- set up environment
```shell
export PROJECT=Project ID
```
- create service account
```shell
gcloud iam service-accounts create my-account --display-name my-account
gcloud projects add-iam-policy-binding $PROJECT --member=serviceAccount:my-account@$PROJECT.iam.gserviceaccount.com --role=roles/bigquery.admin
gcloud projects add-iam-policy-binding $PROJECT --member=serviceAccount:my-account@$PROJECT.iam.gserviceaccount.com --role=roles/serviceusage.serviceUsageConsumer
gcloud iam service-accounts keys create key.json --iam-account=my-account@$PROJECT.iam.gserviceaccount.com
export GOOGLE_APPLICATION_CREDENTIALS=key.json
```
- write python script [classify-text.py]
```python

from google.cloud import storage, language, bigquery

# Set up your GCS, NL, and BigQuery clients

storage_client = storage.Client()
nl_client = language.LanguageServiceClient()
bq_client = bigquery.Client(project='Project ID')

dataset_ref = bq_client.dataset('news_classification_dataset')
dataset = bigquery.Dataset(dataset_ref)
table_ref = dataset.table('article_data')
table = bq_client.get_table(table_ref)

# Send article text to the NL API's classifyText method

def classify_text(article):
    response = nl_client.classify_text(
        document=language.Document(
            content=article,
            type=language.Document.Type.PLAIN_TEXT
        )
    )
    return response

rows_for_bq = []
files = storage_client.bucket('qwiklabs-test-bucket-gsp063').list_blobs()
print("Got article files from GCS, sending them to the NL API (this will take ~2 minutes)...")

# Send files to the NL API and save the result to send to BigQuery

for file in files:
    if file.name.endswith('txt'):
        article_text = file.download_as_bytes().decode('utf-8')  # Decode bytes to string
        nl_response = classify_text(article_text)
        if len(nl_response.categories) > 0:
            rows_for_bq.append((article_text, nl_response.categories[0].name, nl_response.categories[0].confidence))

print("Writing NL API article data to BigQuery...")

# Write article text + category data to BQ

if rows_for_bq:
    errors = bq_client.insert_rows(table, rows_for_bq)
    if errors:
        print("Encountered errors while writing to BigQuery:", errors)
else:
    print("No articles found in the specified bucket.")
```
- run script
```shell
python3 classify-text.py
```
- verify data table
```sql
SELECT * FROM `Project ID.news_classification_dataset.article_data`
```
### Task 7. Analyze categorized news data in BigQuery
- view the most common categories
```sql
SELECT
  category,
  COUNT(*) c
FROM
  `Project ID.news_classification_dataset.article_data`
GROUP BY
  category
ORDER BY
  c DESC
```
- find the article returned for a more obscure category like /Arts & Entertainment/Music & Audio/Classical Music
```sql
SELECT * FROM `Project ID.news_classification_dataset.article_data`
WHERE category = "/Arts & Entertainment/Music & Audio/Classical Music"
```
- get only the articles where the Natural language API returned a confidence score greater than 90%
```shell
SELECT
  article_text,
  category
FROM `Project ID.news_classification_dataset.article_data`
WHERE cast(confidence as float64) > 0.9
```
- 
## Detect Labels, Faces, and Landmarks in Images with the Cloud Vision API [GSP037]
### Task 1. Create an API key
- Navigation menu > APIs & Services > Credentials
- Create Credentials > API key
- copy & Close
```shell
export API_KEY=
```

### Task 2. Upload an image to a Cloud Storage bucket
- Navigation Menu > Cloud Storage > Bucket > Create
- Choose how to control access to objects
- Enforce public access prevention on this bucket
- Fine-grained
- Create
- upload donuts.png 
- Edit access
- Add entry
  - Entity: Public
  - Name: allUsers
  - Access: Reader
- Save

### Task 3. Create your request
```shell
cat > request.json << EOF
{
  "requests": [
      {
        "image": {
          "source": {
              "gcsImageUri": "gs://my-bucket-name/donuts.png"
          }
        },
        "features": [
          {
            "type": "LABEL_DETECTION",
            "maxResults": 10
          }
        ]
      }
  ]
}
EOF
```

### Task 4. Label detection
```shell 
curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json  https://vision.googleapis.com/v1/images:annotate?key=${API_KEY}
```

### Task 5. Web detection
```shell
cat > request.json << EOF
{
  "requests": [
      {
        "image": {
          "source": {
              "gcsImageUri": "gs://my-bucket-name/donuts.png"
          }
        },
        "features": [
          {
            "type": "WEB_DETECTION",
            "maxResults": 10
          }
        ]
      }
  ]
}
EOF

curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json  https://vision.googleapis.com/v1/images:annotate?key=${API_KEY}
```

### Task 6. Face detection
- upload selfie.png
- update request file
```shell
cat > request.json << EOF
{
  "requests": [
      {
        "image": {
          "source": {
              "gcsImageUri": "gs://my-bucket-name/selfie.png"
          }
        },
        "features": [
          {
            "type": "FACE_DETECTION"
          },
          {
            "type": "LANDMARK_DETECTION"
          }
        ]
      }
  ]
}
EOF

curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json  https://vision.googleapis.com/v1/images:annotate?key=${API_KEY}
```

### Task 7. Landmark annotation
- upload city.png
- update request file
```shell
cat > request.json << EOF
{
  "requests": [
      {
        "image": {
          "source": {
              "gcsImageUri": "gs://my-bucket-name/city.png"
          }
        },
        "features": [
          {
            "type": "LANDMARK_DETECTION",
            "maxResults": 10
          }
        ]
      }
  ]
}
EOF

curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json  https://vision.googleapis.com/v1/images:annotate?key=${API_KEY}
```

### Task 8. Object localization
- update request file
```shell
cat > request.json << EOF
{
  "requests": [
    {
      "image": {
        "source": {
          "imageUri": "https://cloud.google.com/vision/docs/images/bicycle_example.png"
        }
      },
      "features": [
        {
          "maxResults": 10,
          "type": "OBJECT_LOCALIZATION"
        }
      ]
    }
  ]
}
EOF

curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json  https://vision.googleapis.com/v1/images:annotate?key=${API_KEY}
```

### Task 9. Explore other Vision API methods
- 

## Entity and Sentiment Analysis with the Natural Language API [GSP038]
### Task 1. Create an API key
- Navigation Menu > APIs & Services > Credentials
- Create Credentials > API key
- copy & Close
- Navigation Menu > Compute Engine > linux-instance > SHH
```shell
export API_KEY=
```

### Task 2. Make an entity analysis request
```shell
cat > request.json << EOF
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"Joanne Rowling, who writes under the pen names J. K. Rowling and Robert Galbraith, is a British novelist and screenwriter who wrote the Harry Potter fantasy series."
  },
  "encodingType":"UTF8"
}
EOF
```

### Task 3. Call the Natural Language API
```shell
curl "https://language.googleapis.com/v1/documents:analyzeEntities?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json > result.json

cat result.json
```

### Task 4. Sentiment analysis with the Natural Language API
- update request.json and request API
```shell
cat > request.json << EOF
 {
  "document":{
    "type":"PLAIN_TEXT",
    "content":"Harry Potter is the best book. I think everyone should read it."
  },
  "encodingType": "UTF8"
}
EOF

curl "https://language.googleapis.com/v1/documents:analyzeSentiment?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json
```

### Task 5. Analyzing entity sentiment
- update request.json and call API
```shell
cat > request.json << EOF
 {
  "document":{
    "type":"PLAIN_TEXT",
    "content":"I liked the sushi but the service was terrible."
  },
  "encodingType": "UTF8"
}
EOF

curl "https://language.googleapis.com/v1/documents:analyzeEntitySentiment?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json
```

### Task 6. Analyzing syntax and parts of speech
- update request.json and call API
```shell
cat > request.json << EOF
{
  "document":{
    "type":"PLAIN_TEXT",
    "content": "Joanne Rowling is a British novelist, screenwriter and film producer."
  },
  "encodingType": "UTF8"
}
EOF

curl "https://language.googleapis.com/v1/documents:analyzeSyntax?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json
```
### Task 7. Multilingual natural language processing
- update request.json and call API
```shell
cat > request.json << EOF
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"日本のグーグルのオフィスは、東京の六本木ヒルズにあります"
  }
}

curl "https://language.googleapis.com/v1/documents:analyzeEntities?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json
```

## Awwvision: Cloud Vision API from a Kubernetes Cluster [GSP066]
### Task 1. Create a Kubernetes Engine cluster
- config zone
```shell
gcloud config set compute/zone Zone
```
- create cluster
```shell
gcloud container clusters create awwvision \
    --num-nodes 2 \
    --scopes cloud-platform
```
- use container credentials
```shell
gcloud container clusters get-credentials awwvision
```
- get info
```shell
kubectl cluster-info
```
### Task 2. Create a virtual environment
```shell
sudo apt-get install -y virtualenv
python3 -m venv venv
source venv/bin/activate
```

### Task 3. Get the sample
```shell
gsutil -m cp -r gs://spls/gsp066/cloud-vision .
```

### Task 4. Deploy the sample
```shell
cd cloud-vision/python/awwvision
make all
```

### Task 5. Check the Kubernetes resources on the cluster
- get info
```shell
kubectl get pods
```
- deploy
```shell
kubectl get deployments -o wide
```
- get ip
```shell
kubectl get svc awwvision-webapp
```

### Task 6. Visit your new webapp and start its crawler
- use above retrieved ip

### Task 7. Test your understanding
- Cloud Vision API

## Use Machine Learning APIs on Google Cloud: Challenge Lab [GSP329]
### Task 1. Configure a service account to access the Machine Learning APIs, BigQuery, and Cloud Storage
```shell
export LANG=Japanese
export LOCAL=ja
export BQ_ROLE=roles/bigquery.dataOwner
export STORAGE_ROLE=roles/storage.objectAdmin

gcloud iam service-accounts create sample-sa

gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID --member=serviceAccount:sample-sa@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role=$BQ_ROLE

gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID --member=serviceAccount:sample-sa@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role=$STORAGE_ROLE

gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID --member=serviceAccount:sample-sa@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role=roles/serviceusage.serviceUsageConsumer
```

### Task 2. Create and download a credential file for your service account
```shell
gcloud iam service-accounts keys create sample-sa-key.json --iam-account sample-sa@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com

export GOOGLE_APPLICATION_CREDENTIALS=${PWD}/sample-sa-key.json

wget https://raw.githubusercontent.com/QUICK-GCP-LAB/2-Minutes-Labs-Solutions/main/Use%20Machine%20Learning%20APIs%20on%20Google%20Cloud%20Challenge%20Lab/analyze-images-v2.py
```
### Task 3. Modify the Python script to extract text from image files
```shell
sed -i "s/'en'/'${LOCAL}'/g" analyze-images-v2.py

python3 analyze-images-v2.py
```
### Task 4. Modify the Python script to translate the text using the Translation API
```shell
python3 analyze-images-v2.py $DEVSHELL_PROJECT_ID $DEVSHELL_PROJECT_ID
```
### Task 5. Identify the most common language used in the signs in the dataset
```shell
bq query --use_legacy_sql=false "SELECT locale,COUNT(locale) as lcount FROM image_classification_dataset.image_text_detail GROUP BY locale ORDER BY lcount DESC"
```
