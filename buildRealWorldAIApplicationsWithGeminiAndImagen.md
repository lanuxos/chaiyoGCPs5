# Build Real World AI Applications with Gemini and Imagen

# Build an AI Image Recognition app using Gemini on Vertex AI [bb-ide-genai-001]
## Working with Vertex AI Python SDK
- File > New File
```python
import vertexai
from vertexai.generative_models import GenerativeModel, Part


def generate_text(project_id: str, location: str) -> str:
    # Initialize Vertex AI
    vertexai.init(project=project_id, location=location)
    # Load the model
    multimodal_model = GenerativeModel("gemini-1.0-pro-vision")
    # Query the model
    response = multimodal_model.generate_content(
        [
            # Add an example image
            Part.from_uri(
                "gs://generativeai-downloads/images/scones.jpg", mime_type="image/jpeg"
            ),
            # Add an example query
            "what is shown in this image?",
        ]
    )

    return response.text

# --------  Important: Variable declaration  --------

project_id = "project-id"
location = "REGION"

#  --------   Call the Function  --------

response = generate_text(project_id, location)
print(response)
```
- File > Save [genai.py]
```shell
/usr/bin/python3 /home/student/genai.py
```

- 

## Code Explanation
The code snippet is loading a pre-trained AI model called Gemini (gemini-1.0-pro-vision) on Vertex AI.
The code calls the generate_content method of the loaded Gemini model.
The input to the method is an image URI and a prompt containing a question about the image.
The code uses Gemini's ability to understand images and text together. It uses the text provided in the prompt to describe the contents of the image.

# Build an AI Image Generator app using Imagen on Vertex AI [bb-ide-genai-002]
## Working with generative ai
```python
import argparse

import vertexai
from vertexai.preview.vision_models import ImageGenerationModel

def generate_image(
    project_id: str, location: str, output_file: str, prompt: str
) -> vertexai.preview.vision_models.ImageGenerationResponse:
    """Generate an image using a text prompt.
    Args:
      project_id: Google Cloud project ID, used to initialize Vertex AI.
      location: Google Cloud region, used to initialize Vertex AI.
      output_file: Local path to the output image file.
      prompt: The text prompt describing what you want to see."""

    vertexai.init(project=project_id, location=location)

    model = ImageGenerationModel.from_pretrained("imagegeneration@002")

    images = model.generate_images(
        prompt=prompt,
        # Optional parameters
        number_of_images=1,
        seed=1,
        add_watermark=False,
    )

    images[0].save(location=output_file)

    return images

generate_image(
    project_id='"project-id"',
    location='"REGION"',
    output_file='image.jpeg',
    prompt='Create an image of a cricket ground in the heart of Los Angeles',
    )
```
## Code explanation
The code snippet is loading a pre-trained AI model called ImageGenerationModel (imagegeneration@002) on Vertex AI.
The code calls the generate_image method of the loaded Gemini model.
The input to the method is a text prompt.
The code uses Gemini's ability to understand the text prompt and use it to build an AI Image.
Note: By default, a SynthID watermark is added to images, but you can disable it by specifying the optional parameter add_watermark=False. You can't use a seed value and watermark at the same time. Learn more about SynthID watermark

# Build an application to send Chat Prompts using the Gemini model [bb-ide-genai-003]
## Working with Generative AI
```python
import vertexai
from vertexai.generative_models import GenerativeModel, ChatSession

import logging
from google.cloud import logging as gcp_logging

# ------  Below cloud logging code is for Qwiklab's internal use, do not edit/remove it. --------
# Initialize GCP logging
gcp_logging_client = gcp_logging.Client()
gcp_logging_client.setup_logging()

project_id = ""your-project-id""
location = ""REGION""

vertexai.init(project=project_id, location=location)
model = GenerativeModel("gemini-1.0-pro")
chat = model.start_chat()

def get_chat_response(chat: ChatSession, prompt: str) -> str:
    logging.info(f'Sending prompt: {prompt}')
    response = chat.send_message(prompt)
    logging.info(f'Received response: {response.text}')
    return response.text

prompt = "Hello."
print(get_chat_response(chat, prompt))

prompt = "What are all the colors in a rainbow?"
print(get_chat_response(chat, prompt))

prompt = "Why does it appear when it rains?"
print(get_chat_response(chat, prompt))

```

## Code Explanation
The code snippet is loading a pre-trained AI model called Gemini (gemini-1.0-pro) on Vertex AI.
The code calls the get_chat_response method of the loaded Gemini model.
The input to the method is a text prompt.
The code uses Gemini's ability to chat. It uses the text provided in the prompt to chat.

## Chat responses with using stream
```python
import vertexai
from vertexai.generative_models import GenerativeModel, ChatSession

import logging
from google.cloud import logging as gcp_logging

# ------  Below cloud logging code is for Qwiklab's internal use, do not edit/remove it. --------
# Initialize GCP logging
gcp_logging_client = gcp_logging.Client()
gcp_logging_client.setup_logging()

project_id = ""your-project-id""
location = ""REGION""

vertexai.init(project=project_id, location=location)
model = GenerativeModel("gemini-1.0-pro")
chat = model.start_chat()

def get_chat_response(chat: ChatSession, prompt: str) -> str:
    text_response = []
    logging.info(f'Sending prompt: {prompt}')
    responses = chat.send_message(prompt, stream=True)
    for chunk in responses:
        text_response.append(chunk.text)
    return "".join(text_response)
    logging.info(f'Received response: {response.text}')

prompt = "Hello."
print(get_chat_response(chat, prompt))

prompt = "What are all the colors in a rainbow?"
print(get_chat_response(chat, prompt))

prompt = "Why does it appear when it rains?"
print(get_chat_response(chat, prompt))
```
## Code Explanation
The code snippet is loading a pre-trained AI model called Gemini (gemini-1.0-pro) on Vertex AI.
The code calls the get_chat_response method of the loaded Gemini model.
The code is using stream=True while sending the messages. The stream=True argument indicates that the responses should be streamed back, allowing for real-time processing.
The code uses Gemini's ability to understand prompts and have a stateful chat conversation.

# Build a Multi-Modal GenAI Application: Challenge Lab [bb-ide-genai-004]
Challenge scenario
Scenario: You're a developer at an AI-powered boquet design company. Your clients can describe their dream bouquet, and your system generates realistic images for their approval. To further enhance the experience, you're integrating cutting-edge image analysis to provide descriptive summaries of the generated bouquets. Your main application will invoke the relevant methods based on the users' interaction and to facilitate that, you need to finish the below tasks:

## Task 1: 
Develop a Python function named generate_bouquet_image(prompt). This function should invoke the imagegeneration@002 model using the supplied prompt, generate the image, and store it locally. For this challenge, use the prompt: "Create an image containing a bouquet of 2 sunflowers and 3 roses".
```python
import argparse

import vertexai
from vertexai.preview.vision_models import ImageGenerationModel

def generate_bouquet_image(prompt: str) -> vertexai.preview.vision_models.ImageGenerationResponse:
    """Generate an image using a text prompt.
    Args:
      project_id: Google Cloud project ID, used to initialize Vertex AI.
      location: Google Cloud region, used to initialize Vertex AI.
      output_file: Local path to the output image file.
      prompt: The text prompt describing what you want to see."""

    project_id='qwiklabs-gcp-04-bd2f8ffec19e'
    location='us-east4'
    output_file='image.jpeg'

    vertexai.init(project=project_id, location=location)

    model = ImageGenerationModel.from_pretrained("imagegeneration@002")

    images = model.generate_images(
        prompt=prompt,
        # Optional parameters
        number_of_images=1,
        seed=1,
        add_watermark=False,
    )

    images[0].save(location=output_file)

    return images

generate_bouquet_image(
    prompt='Create an image containing a bouquet of 2 sunflowers and 3 roses',
    )
```

## Task 2: 
Develop a second Python function called analyze_bouquet_image(image_path). This function will take the image path as input along with a text prompt to generate birthday wishes based on the image passed and send it to the gemini-pro-vision model. To ensure responses can be obtained as and when they are generated, enable streaming on the prompt requests.
```python
import vertexai
from vertexai.generative_models import GenerativeModel, Part


def generate_text(project_id: str, location: str) -> str:
    # Initialize Vertex AI
    vertexai.init(project=project_id, location=location)
    # Load the model
    multimodal_model = GenerativeModel("gemini-1.0-pro-vision")
    # Query the model
    response = multimodal_model.generate_content(
        [
            # Add an example image
            Part.from_uri(
                "gs://generativeai-downloads/images/scones.jpg", mime_type="image/jpeg"
            ),
            # Add an example query
            "what is shown in this image?",
        ]
    )

    return response.text

# --------  Important: Variable declaration  --------

project_id = "$ID"
location = "$REGION"

#  --------   Call the Function  --------

response = generate_text(project_id, location)
print(response)
```
