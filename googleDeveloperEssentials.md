# Google Developer Essentials

# Weather Data in BigQuery [GSP009]
## Task 1. Explore weather data
- Navigation menu > BigQuery
- Done
- + ADD
- Star a project by name
- bigquery-public-data
- STAR
- search for 'noaa_gsod' and select gsod2014
```sql
SELECT
  -- Create a timestamp from the date components.
  stn,
  TIMESTAMP(CONCAT(year,"-",mo,"-",da)) AS timestamp,
  -- Replace numerical null values with actual null
  AVG(IF (temp=9999.9,
      null,
      temp)) AS temperature,
  AVG(IF (wdsp="999.9",
      null,
      CAST(wdsp AS Float64))) AS wind_speed,
  AVG(IF (prcp=99.99,
      0,
      prcp)) AS precipitation
FROM
  `bigquery-public-data.noaa_gsod.gsod20*`
WHERE
  CAST(YEAR AS INT64) > 2010
  AND CAST(MO AS INT64) = 6
  AND CAST(DA AS INT64) = 12
  AND (stn="725030" OR  -- La Guardia
    stn="744860")    -- JFK
GROUP BY
  stn,
  timestamp
ORDER BY
  timestamp DESC,
  stn ASC
```

## Task 2. Explore New York citizen complaints data
- search for 'new_york_311' and select '311_service_requests'
- preview
```sql
SELECT
  EXTRACT(YEAR
  FROM
    created_date) AS year,
  complaint_type,
  COUNT(1) AS num_complaints
FROM
  `bigquery-public-data.new_york.311_service_requests`
GROUP BY
  year,
  complaint_type
ORDER BY
  num_complaints DESC
```
## Task 3. Saving a new table of weather data
- three dots
- Create dataset
- Dataset ID: 'demos'
- Create dataset
```sql
SELECT
  -- Create a timestamp from the date components.
  timestamp(concat(year,"-",mo,"-",da)) as timestamp,
  -- Replace numerical null values with actual nulls
  AVG(IF (temp=9999.9, null, temp)) AS temperature,
  AVG(IF (visib=999.9, null, visib)) AS visibility,
  AVG(IF (wdsp="999.9", null, CAST(wdsp AS Float64))) AS wind_speed,
  AVG(IF (gust=999.9, null, gust)) AS wind_gust,
  AVG(IF (prcp=99.99, null, prcp)) AS precipitation,
  AVG(IF (sndp=999.9, null, sndp)) AS snow_depth
FROM
  `bigquery-public-data.noaa_gsod.gsod20*`
WHERE
  CAST(YEAR AS INT64) > 2008
  AND (stn="725030" OR  -- La Guardia
       stn="744860")    -- JFK
GROUP BY timestamp
```
- More > Query settings
- Destination: 'Set a destination table for query results'
- Dataset: 'demos'
- Table Id: 'nyc_weather'
- results size: check 'Allow large results(no size limit)'
- SAVE
- RUN
- More > Query settings
- Save query results in a temporary table
- SAVE
## Task 4. Finding correlation between datasets
Strong correlation, as measured by the CORR function, indicates a close and consistent relationship between two variables. As the value of one variable increases, the value of the other variable also tends to increase (positive correlation) or decrease (negative correlation) in a predictable way. Strong correlation is generally considered to be a value greater than or equal to 0.7, in absolute terms. This means that the changes in one variable can explain at least 49% of the changes in the other variable.
- compare the number of complaints received and daily temperature using the CORR function.
```sql
SELECT
  descriptor,
  sum(complaint_count) as total_complaint_count,
  count(temperature) as data_count,
  ROUND(corr(temperature, avg_count),3) AS corr_count,
  ROUND(corr(temperature, avg_pct_count),3) AS corr_pct
From (
SELECT
  avg(pct_count) as avg_pct_count,
  avg(day_count) as avg_count,
  sum(day_count) as complaint_count,
  descriptor,
  temperature
FROM (
  SELECT
    DATE(timestamp) AS date,
    temperature
  FROM
    demos.nyc_weather) a
  JOIN (
  SELECT x.date, descriptor, day_count, day_count / all_calls_count as pct_count
  FROM
    (SELECT
      DATE(created_date) AS date,
      concat(complaint_type, ": ", descriptor) as descriptor,
      COUNT(*) AS day_count
    FROM
      `bigquery-public-data.new_york.311_service_requests`
    GROUP BY
      date,
      descriptor)x
    JOIN (
      SELECT
        DATE(timestamp) AS date,
        COUNT(*) AS all_calls_count
      FROM `demos.nyc_weather`
      GROUP BY date
    )y
  ON x.date=y.date
)b
ON
  a.date = b.date
GROUP BY
  descriptor,
  temperature
)
GROUP BY descriptor
HAVING
  total_complaint_count > 5000 AND
  ABS(corr_pct) > 0.5 AND
  data_count > 5
ORDER BY
  ABS(corr_pct) DESC
```
The results indicate that Heating complaints are negatively correlated with temperature (i.e., more heating calls on cold days) and calls about dead trees are positively correlated with temperature (i.e., more calls on hot days).
- compare the number of complaints and wind speed with the CORR function.
```sql
SELECT
  descriptor,
  sum(complaint_count) as total_complaint_count,
  count(wind_speed) as data_count,
  ROUND(corr(wind_speed, avg_count),3) AS corr_count,
  ROUND(corr(wind_speed, avg_pct_count),3) AS corr_pct
From (
SELECT
  avg(pct_count) as avg_pct_count,
  avg(day_count) as avg_count,
  sum(day_count) as complaint_count,
  descriptor,
  wind_speed
FROM (
  SELECT
    DATE(timestamp) AS date,
    wind_speed
  FROM
    demos.nyc_weather) a
  JOIN (
  SELECT x.date, descriptor, day_count, day_count / all_calls_count as pct_count
  FROM
    (SELECT
      DATE(created_date) AS date,
      concat(complaint_type, ": ", descriptor) as descriptor,
      COUNT(*) AS day_count
    FROM
      `bigquery-public-data.new_york.311_service_requests`
    GROUP BY
      date,
      descriptor)x
    JOIN (
      SELECT
        DATE(timestamp) AS date,
        COUNT(*) AS all_calls_count
      FROM `demos.nyc_weather`
      GROUP BY date
    )y
  ON x.date=y.date
)b
ON
  a.date = b.date
GROUP BY
  descriptor,
  wind_speed
)
GROUP BY descriptor
HAVING
  total_complaint_count > 5000 AND
  ABS(corr_pct) > 0.5 AND
  data_count > 5
ORDER BY
  ABS(corr_pct) DESC
```

# Classify Images of Clouds in the Cloud with AutoML Images [GSP223]
## Task 1. Set up the AutoML environment
- Confirm that Cloud AutoML API is enabled
  - Navigation menu > APIs & Services > Library
  - Search for APIs & Services: Cloud AutoML
  - Cloud AutoML API
  - Enable
- Open the Vertex AI Dashboard
  - Navigation menu > Vertex AI
- Create a storage bucket
```console
gsutil mb -p $GOOGLE_CLOUD_PROJECT \
    -c standard    \
    -l us \
    gs://$GOOGLE_CLOUD_PROJECT-vcm/
```

## Task 2. Upload training images to Cloud Storage
- create an environment variable
```console
export BUCKET=$GOOGLE_CLOUD_PROJECT-vcm
```
- copy image to cloud storage
```console
gsutil -m cp -r gs://spls/gsp223/images/* gs://${BUCKET}
```
- viewing image
  - Cloud storage > Buckets

## Task 3. Create a dataset
- copy csv to cloud shell instance
```console
gsutil cp gs://spls/gsp223/data.csv .
```
- update the csv file with the files in your project
```console
sed -i -e "s/placeholder/${BUCKET}/g" ./data.csv
```
- upload this file to cloud storage bucket
```console
gsutil cp ./data.csv gs://${BUCKET}
```
- Refresh
- Navigation menu > Vertex AI > Datasets
- Create
- Dataset Name: 'clouds'
- Single-label classification
- Create
- Select import files from Cloud Storage
- Browse > Project_ID-vcm
- data.csv
- select
- Continue

## Task 4. Inspect images
- If you were building a production model, you'd want at least 100 images per label to ensure high accuracy. This is just a demo so only 20 images were used so the model could train quickly.

## Task 5. Generate predictions
- download images
```console
gsutil cp gs://spls/gsp223/examples/* .
```
- view the example file
```console
cat CLOUD1-JSON
```
- copy endpoint value to an environment variable
```console
ENDPOINT=$(gcloud run services describe automl-service --platform managed --region REGION --format 'value(status.url)')
```
- request a prediction
```console
curl -X POST -H "Content-Type: application/json" $ENDPOINT/v1 | jq
```

## Task 6. Pop Quiz
- 


# Entity and Sentiment Analysis with the Natural Language API [038]
## Task 1. Create an API Key
- Navigation menu > APIs & Services > Credentials
- Create credentials 
- API key
- copy the generated API key
- close
- Navigation menu > Compute Engine
- SSH
- export API key
```shell
export API_KEY=<YOUR_API_KEY>
```

## Task 2. Make an entity analysis request
- nano request.json
```json
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"Joanne Rowling, who writes under the pen names J. K. Rowling and Robert Galbraith, is a British novelist and screenwriter who wrote the Harry Potter fantasy series."
  },
  "encodingType":"UTF8"
}
```
In the request, you're telling the Natural Language API about the text being sent. Supported type values are PLAIN_TEXT or HTML. In content, you pass the text to send to the Natural Language API for analysis.

The Natural Language API also supports sending files stored in Cloud Storage for text processing. If you wanted to send a file from Cloud Storage, you would replace content with gcsContentUri and give it a value of the text file's uri in Cloud Storage.

encodingType tells the API which type of text encoding to use when processing our text. The API will use this to calculate where specific entities appear in our text.
## Task 3. Call the Natural Language API
- curl
```shell
curl "https://language.googleapis.com/v1/documents:analyzeEntities?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json > result.json
```
- cat result.json
## Task 4. Sentiment analysis with the Natural Language API
- nano request.json
```json
 {
  "document":{
    "type":"PLAIN_TEXT",
    "content":"Harry Potter is the best book. I think everyone should read it."
  },
  "encodingType": "UTF8"
}
```
- curl
```shell
curl "https://language.googleapis.com/v1/documents:analyzeSentiment?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json
```
## Task 5. Analyzing entity sentiment
- nano request.json
```json
 {
  "document":{
    "type":"PLAIN_TEXT",
    "content":"I liked the sushi but the service was terrible."
  },
  "encodingType": "UTF8"
}
```
- curl
```shell
curl "https://language.googleapis.com/v1/documents:analyzeEntitySentiment?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json
```

## Task 6. Analyzing syntax and parts of speech
- nano request.json
```json
{
  "document":{
    "type":"PLAIN_TEXT",
    "content": "Joanne Rowling is a British novelist, screenwriter and film producer."
  },
  "encodingType": "UTF8"
}
```
- curl
```shell
curl "https://language.googleapis.com/v1/documents:analyzeSyntax?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json
```
## Task 7. Multilingual natural language processing
- request.json
```json
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"日本のグーグルのオフィスは、東京の六本木ヒルズにあります"
  }
}
```
- curl
```shell
curl "https://language.googleapis.com/v1/documents:analyzeEntities?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json
```

# App Engine: Qwik Start - Java [GSP068]
## Task 1. Download the sample HTTP Server app
- activate cloud shell
```shell
gcloud storage cp -r gs://spls/gsp068/appengine-java21/appengine-java21/* .
```
- navigate to directory
```shell
cd helloworld/http-server
```
## Task 2. Deploy and view your app
- create application on an App Engine
```shell
gcloud app deploy
```
- choose region
- view application
```shell
gcloud app browse
```

## Task 3. Test your knowledge
all of them

# App Engine: Qwik Start - Python [GSP067]
## Task 1. Enable Google App Engine Admin API
- Navigation menu > APIs & Services > Library
- App Engine Admin API
- Enable

## Task 2. Download the Hello World App
- clone
```shell
git clone https://github.com/GoogleCloudPlatform/python-docs-samples.git
```
- navigate to directory
```shell
cd python-docs-samples/appengine/standard_python3/hello_world
```
- setup python environment
```shell
sudo apt install python3 -y
sudo apt install python3.11-venv
python3 -m venv create myvenv
source myvenv/bin/activate
```

## Task 3. Test the application
- test app using Google Cloud development server
```shell
dev_appserver.py app.yaml
```
- Web preview > Preview on port 8080

## Task 4. Make a change
- on new shell
```shell
cd python-docs-samples/appengine/standard_python3/hello_world
```
- nano main.py change the application to "Hello, Cruel World!"

## Task 5. Deploy your app
- deploy
```shell
gcloud app deploy
```
- choose region
- Y

## Task 6. View your application
```shell
gcloud app browse
```

# Task 7. Test your knowledge
application code
python
java
php
go
nodejs
ruby
cloudFunction
cloudRun

# Autoscaling an Instance Group with Custom Cloud Monitoring Metrics [GSP087]
## Task 1. Creating the application
- uploading the script files to Cloud Storage

## Task 2. Create a bucket
- Navigation menu > Cloud Storage > Buckets
- Create
- Create
- Confirm
- copy startup script
```shell
gsutil cp -r gs://spls/gsp087/* gs://<YOUR BUCKET>
```

## Task 3. Creating an instance template
- Navigation menu > Compute Engine > Instance templates
- Create Instance Template
- name: 'autoscaling-instance01'
- set Location as Global
- Advanced options
- Metadata section of the Management tab
- +Add item
startup-script-url  gs://[YOUR_BUCKET_NAME]/startup.sh
gcs-bucket	        gs://[YOUR_BUCKET_NAME]
- Create
## Task 4. Creating the instance group
- Instance groups
- Create instance group
- name: 'autoscaling-instance-group-1'
- Instance template: select just created instance
- Location/Single Zone
- Autoscaling mode: Off:do not autoscale
- Create

## Task 5. Verifying that the instance group has been created

## Task 6. Verifying that the Node.js script is running
- autoscaling-instance-group-1
- Details
- Logs
- Logging
- Show query toggle
- @ line 3 add: "nodeapp"
- Run query

## Task 7. Configure autoscaling for the instance groups
- Compute Engine > Instance groups
- autoscaling-instance-group-1
- Edit
- Autoscaling: Autoscaling mode => On: add and remove instances to the group
- Minimum number of instances: 1
- Maximum number of instances: 3
- Autoscaling signals > ADD SIGNAL
- Signal type: Cloud Monitoring metric new
- Configure
- Resource and metric
- SELECT A METRIC
- VM Instance > Custom metric > Custom/appdemo_queue_depth_01
- Apply
- Utilization target: 150
- Utilization target type: Gauge
- Select
- Save

## Task 8. Watching the instance group perform autoscaling
- Instance groups
- builtin-igm
- Monitoring
- Enable AutoRefresh
- 

## Task 9. Autoscaling example
- 

