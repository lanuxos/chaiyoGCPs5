# Google Cloud AI Study Jam: #ChaiyoGCP Season 5
# Prompt Design in Vertex AI

# GSP1151
## Task 1. Open the notebook in Vertex AI Workbench
- Navigation menu
- Vertex AI > Workbench
- Open JupyterLab
## Task 2. Set up the notebook
- Select Kernel: Python 3
- Getting Started and the Import libraries
## Task 3. Prompt engineering best practices
- Be concise
## Task 4. Reduce Output Variability
## Task 5. Improve Response Quality by Including Examples

# GSP1154
## Task 1. Analyze images with Gemini in Freeform mode
- search for "Vertex AI API" and Enable it
- Navigation menu > Vertex AI > Vertex AI Studio > Overview
- Open Freeform
- download image
- Insert media > Upload
- type or copy/paste the prompt text
- submit
- configure
- save
## Task 2. Explore multimodal capabilities
- Cloud storage > Buckets
- Activate Cloud Shell
```bash
gcloud storage cp gs://spls/gsp154/video/train.mp4 gs://BUCKET'S_NAME
```
- Navigation menu > Vertex AI > Vertex AI Studio > Overview
- Open Freeform
- Configuration [model name, region]
- Insert Media > Import from Cloud Storage
- Select
- type the prompt
- submit
## Task 3. Design text prompts
- Zero-shot prompting - This is a method where the LLM is given only a prompt that describes the task and no additional data. For example, if you want the LLM to answer a question, you just prompt "what is prompt design?".
- One-shot prompting - This is a method where the LLM is given a single example of the task that it is being asked to perform. For example, if you want the LLM to write a poem, you might give it a single example poem.
- Few-shot prompting - This is a method where the LLM is given a small number of examples of the task that it is being asked to perform. For example, if you want the LLM to write a news article, you might give it a few news articles to read.
- parameters [Temperature/Output token limit]
  - Temperature controls the randomness in token selection. A lower temperature is good when you expect a true or correct response. A temperature of 0 means the highest probability token is always selected. A higher temperature can lead to diverse, unexpected, or potentially biased results. The model name model has a temperature range of 0 - 2 and a default of 1.
  - Output token limit determines the maximum amount of text output from one prompt. A token is approximately four characters.
- FOR ONE/FEW SHOT
  - Vertex AI Studio > Overview > Open Freeform
  - model/region
  - Add Examples
## Task 4. Generate conversations
- Chat
- model/region
- System instructions
```
Your name is Roy.
You are a support technician of an IT department.
You only respond with "Have you tried turning it off and on again?" to any queries.
```
- submit
```
Your name is Roy.
You are a support technician of an IT department.
You are here to support the users with their queries.
```
- submit
# GSP1209
## Task 1. Open the notebook in Vertex AI Workbench
- Navigation menu > Vertex AI > Workbench > Open JupyterLab
  
## Task 2. Set up the notebook
- Open the notebook_name
- select Kernel: Python 3
- Getting started and Import libraries sections
## Task 3. Use the Gemini 1.5 Pro model
- 
## Task 4. Generate text from a multimodal prompt
-

# GSP519: Prompt Design in Vertex AI: Challenge Lab
Challenge Scenario
You're a member of an educational content startup specializing in engaging learners with the natural world. You've formed a partnership with Cymbal Direct, an online retailer launching a new line of outdoor gear and apparel designed to encourage young people to explore and connect with nature.

Cymbal Direct wants to create a marketing campaign for its new product line that leverages the power of generative AI. Your task is to help them develop a set of tools within Google Cloud's Vertex AI platform that will streamline the generation of the following:

Evocative Product Descriptions: using image analysis to inspire short, descriptive text that captures the essence of their products and the feeling of being in nature.
Catchy Taglines: focused on highlighting product features, the target audience, and the desired emotional response.
## Task 1. Build a Gemini image analysis tool


## Task 2. Build a Gemini tagline generator


## Task 3. Experiment with image analysis code
```python
from google import genai
from google.genai import types
import base64

def generate():
  client = genai.Client(
      vertexai=True,
      project="qwiklabs-gcp-03-9fb8aabc8f7b",
      location="us-east4",
  )

  image1 = types.Part.from_uri(
      file_uri="gs://qwiklabs-gcp-03-9fb8aabc8f7b-labconfig-bucket/cymbal-product-image.png",
      mime_type="image/png",
  )

  model = "gemini-2.0-flash-001"
  contents = [
    types.Content(
      role="user",
      parts=[
        image1,
        types.Part.from_text(text="""Change the wording of the prompt in the code cell to make the output less than 10 words""")
      ]
    )
  ]
  generate_content_config = types.GenerateContentConfig(
    temperature = 1,
    top_p = 0.95,
    max_output_tokens = 8192,
    response_modalities = ["TEXT"],
  )

  for chunk in client.models.generate_content_stream(
    model = model,
    contents = contents,
    config = generate_content_config,
    ):
    print(chunk.text, end="")

generate()
```

## Task 4. Experiment with tagline generation code
```python
from google import genai
from google.genai import types
import base64

def generate():
  client = genai.Client(
      vertexai=True,
      project="qwiklabs-gcp-03-9fb8aabc8f7b",
      location="us-east4",
  )

  text1 = types.Part.from_text(text="""input: Write a tagline for a durable backpack designed for hikers that makes them feel prepared.
output: Built for the Journey: Your Adventure

input: Consider styles like minimalist.
output: Essentials.


input: adventurers
output:""")
  si_text1 = """specifically request that the tagline includes the keyword nature"""

  model = "gemini-2.0-flash-001"
  contents = [
    types.Content(
      role="user",
      parts=[
        text1
      ]
    )
  ]
  generate_content_config = types.GenerateContentConfig(
    temperature = 1,
    top_p = 0.95,
    max_output_tokens = 8192,
    response_modalities = ["TEXT"],
    system_instruction=[types.Part.from_text(text=si_text1)],
  )

  for chunk in client.models.generate_content_stream(
    model = model,
    contents = contents,
    config = generate_content_config,
    ):
    print(chunk.text, end="")

generate()
```
