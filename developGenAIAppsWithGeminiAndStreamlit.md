# Develop GenAI Apps with Gemini and Streamlit

# Develop GenAI Apps with Gemini And Streamlit: Challenge Lab
Challenge scenario
You onboarded at Cymbal Health just a few months ago. Cymbal Health is an established health network in East Central Minnesota dedicated to reimagining and transforming the way that healthcare can be delivered. Cymbal Health connects care and coverage under one health plan to make it easier for patients to receive high quality care at an affordable cost.

As a value added service, Cymbal Health is interested in improving customer Healthy Living and Wellness, with tips, and advice within apps. One particular area they want to focus on is improving customer nutrition.
```shell
git clone https://github.com/GoogleCloudPlatform/generative-ai.git --depth=1
cd generative-ai/gemini/sample-apps/gemini-streamlit-cloudrun
python3 -m venv gemini-streamlit
source gemini-streamlit/bin/activate
pip install -r requirements.txt
GOOGLE_CLOUD_PROJECT='qwiklabs-gcp-00-a8e67866a8c6'
GOOGLE_CLOUD_REGION='us-west1'
streamlit run app.py   --browser.serverAddress=localhost   --server.enableCORS=false   --server.enableXsrfProtection=false   --server.port 8080
AR_REPO='gemini-repo'
SERVICE_NAME='gemini-streamlit-app' 
gcloud artifacts repositories create "$AR_REPO" --location="$GOOGLE_CLOUD_REGION" --repository-format=Docker
gcloud builds submit --tag "$GOOGLE_CLOUD_REGION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/$AR_REPO/$SERVICE_NAME"
gcloud run deploy "$SERVICE_NAME"   --port=8080   --image="$GOOGLE_CLOUD_REGION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/$AR_REPO/$SERVICE_NAME"   --allow-unauthenticated   --region=$GOOGLE_CLOUD_REGION   --platform=managed    --project=$GOOGLE_CLOUD_PROJECT   --set-env-vars=GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT,GOOGLE_CLOUD_REGION=$GOOGLE_CLOUD_REGION
```