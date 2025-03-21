# Explore Generative AI with the Gemini API in Vertex AI
## Explore Generative AI with the Gemini API in Vertex AI: Challenge Lab [GSP515]
### Task 1. Generate text using Gemini
```shell
pip3 install --upgrade --user google-cloud-aiplatform

PROJECT_ID=$(gcloud config get-value project)
LOCATION="us-west1"
API_ENDPOINT=${LOCATION}-aiplatform.googleapis.com
MODEL_ID="gemini-1.0-pro"

gcloud services enable aiplatform.googleapis.com

cat > request.json << 'EOF'
{
  "contents": [{ "role": "user", "parts": { "text": "Why is the sky blue?" }}],
  "generation_config": {"temperature": 0.5}
}
EOF

curl -X POST \
     -H "Authorization: Bearer $(gcloud auth print-access-token)" \
     -H "Content-Type: application/json; charset=utf-8" \
     -d @request.json \
     "https://${LOCATION}-aiplatform.googleapis.com/v1/projects/${PROJECT_ID}/locations/${LOCATION}/publishers/google/models/gemini-1.0-pro:generateContent"
```
### Task 2. Open the notebook in Vertex AI Workbench
- Navigation menu > Vertex AI > Workbench
- gemini-explorer-challenge-v1.0.0.ipynb
### Task 3. Create a function call using Gemini