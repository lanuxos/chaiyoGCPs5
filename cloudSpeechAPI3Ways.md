# Cloud Speech API: 3 Ways
## It Speaks! Create Synthetic Speech Using Text-to-Speech [GSP222]
### Task 1. Enable the Text-to-Speech API
- Cloud Text-to-Speech API
### Task 2. Create a virtual environment
```shell
sudo apt-get install -y virtualenv
python3 -m venv venv
source venv/bin/activate
```
### Task 3. Create a service account
- create a service account
```shell
gcloud iam service-accounts create tts-qwiklab
```
- generate a key
```shell
gcloud iam service-accounts keys create tts-qwiklab.json --iam-account tts-qwiklab@Project ID.iam.gserviceaccount.com
```
- export key
```shell
export GOOGLE_APPLICATION_CREDENTIALS=tts-qwiklab.json
```
### Task 4. Get a list of available voices
- curl all voices
```shell
curl -H "Authorization: Bearer "$(gcloud auth application-default print-access-token) \
    -H "Content-Type: application/json; charset=utf-8" \
    "https://texttospeech.googleapis.com/v1/voices"
```
- curl just a single language code
```shell
curl -H "Authorization: Bearer "$(gcloud auth application-default print-access-token) \
    -H "Content-Type: application/json; charset=utf-8" \
    "https://texttospeech.googleapis.com/v1/voices?language_code=en"
```
### Task 5. Create synthetic speech from text
- create json file: synthesize-text.json
```json
{
    'input':{
        'text':'Cloud Text-to-Speech API allows developers to include
           natural-sounding, synthetic human speech as playable audio in
           their applications. The Text-to-Speech API converts text or
           Speech Synthesis Markup Language (SSML) input into audio data
           like MP3 or LINEAR16 (the encoding used in WAV files).'
    },
    'voice':{
        'languageCode':'en-gb',
        'name':'en-GB-Standard-A',
        'ssmlGender':'FEMALE'
    },
    'audioConfig':{
        'audioEncoding':'MP3'
    }
}
```
- call text-to-speech api
```shell
curl -H "Authorization: Bearer "$(gcloud auth application-default print-access-token) \
  -H "Content-Type: application/json; charset=utf-8" \
  -d @synthesize-text.json "https://texttospeech.googleapis.com/v1/text:synthesize" \
  > synthesize-text.txt
```
- create tts_decode.py
```python
import argparse
from base64 import decodebytes
import json

"""
Usage:
        python tts_decode.py --input "synthesize-text.txt" \
        --output "synthesize-text-audio.mp3"

"""

def decode_tts_output(input_file, output_file):
    """ Decode output from Cloud Text-to-Speech.

    input_file: the response from Cloud Text-to-Speech
    output_file: the name of the audio file to create

    """

    with open(input_file) as input:
        response = json.load(input)
        audio_data = response['audioContent']

        with open(output_file, "wb") as new_file:
            new_file.write(decodebytes(audio_data.encode('utf-8')))

if __name__ == '__main__':
    parser = argparse.ArgumentParser(
        description="Decode output from Cloud Text-to-Speech",
        formatter_class=argparse.RawDescriptionHelpFormatter)
    parser.add_argument('--input',
                       help='The response from the Text-to-Speech API.',
                       required=True)
    parser.add_argument('--output',
                       help='The name of the audio file to create',
                       required=True)

    args = parser.parse_args()
    decode_tts_output(args.input, args.output)
```
- create an audio file
```shell
python tts_decode.py --input "synthesize-text.txt" --output "synthesize-text-audio.mp3"
```
- create a web server: index.html
```html
<html>
  <body>
  <h1>Cloud Text-to-Speech codelab</h1>
  <p>
  Output from synthesizing text:
  </p>
  <audio controls>
  <source src="synthesize-text-audio.mp3" />
  </audio>
  </body>
</html>
</ql-code-block>
```
- run server
```python
python -m http.server 8080
```
- Web Preview
### Task 6. Create synthetic speech from SSML
- build request text-to-speech json file: synthesize-ssml.json
```json
{
    'input':{
        'ssml':'<speak><s>
           <emphasis level="moderate">Cloud Text-to-Speech API</emphasis>
           allows developers to include natural-sounding
           <break strength="x-weak"/>
           synthetic human speech as playable audio in their
           applications.</s>
           <s>The Text-to-Speech API converts text or
           <prosody rate="slow">Speech Synthesis Markup Language</prosody>
           <say-as interpret-as=\"characters\">SSML</say-as>
           input into audio data
           like <say-as interpret-as=\"characters\">MP3</say-as> or
           <sub alias="linear sixteen">LINEAR16</sub>
           <break strength="weak"/>
           (the encoding used in
           <sub alias="wave">WAV</sub> files).</s></speak>'
    },
    'voice':{
        'languageCode':'en-gb',
        'name':'en-GB-Standard-A',
        'ssmlGender':'FEMALE'
    },
    'audioConfig':{
        'audioEncoding':'MP3'
    }
}
```
- curl text-to-speech
```shell
curl -H "Authorization: Bearer "$(gcloud auth application-default print-access-token) \
  -H "Content-Type: application/json; charset=utf-8" \
  -d @synthesize-ssml.json "https://texttospeech.googleapis.com/v1/text:synthesize" \
  > synthesize-ssml.txt
```
- create audio file
```python
python tts_decode.py --input "synthesize-ssml.txt" --output "synthesize-ssml-audio.mp3"
```
- host audio file: index.html
```html
<html>
  <body>
  <h1>Cloud Text-to-Speech Demo</h1>
  <p>
  Output from synthesizing text:
  </p>
  <audio controls>
    <source src="synthesize-text-audio.mp3" />
  </audio>
  <p>
  Output from synthesizing SSML:
  </p>
  <audio controls>
    <source src="synthesize-ssml-audio.mp3" />
  </audio>
  </body>
</html>
```
- run server
```shell
python -m http.server 8080
```
### Task 7. Configure audio output and device profiles
- build request json file: synthesize-with-settings.json
```json
{
    'input':{
        'text':'The Text-to-Speech API is ideal for any application
          that plays audio of human speech to users. It allows you
          to convert arbitrary strings, words, and sentences into
          the sound of a person speaking the same things.'
    },
    'voice':{
        'languageCode':'en-us',
        'name':'en-GB-Standard-A',
        'ssmlGender':'FEMALE'
    },
    'audioConfig':{
      'speakingRate': 1.15,
      'pitch': -2,
      'audioEncoding':'OGG_OPUS',
      'effectsProfileId': ['headphone-class-device']
    }
}
```
- curl API
```shell
curl -H "Authorization: Bearer "$(gcloud auth application-default print-access-token) \
  -H "Content-Type: application/json; charset=utf-8" \
  -d @synthesize-with-settings.json "https://texttospeech.googleapis.com/v1beta1/text:synthesize" \
  > synthesize-with-settings.txt
```
- create audio file
```shell
python tts_decode.py --input "synthesize-with-settings.txt" --output "synthesize-with-settings-audio.ogg"
```
- host file: index.html
```html
<html>
  <body>
  <h1>Cloud Text-to-Speech Demo</h1>
  <p>
  Output from synthesizing text:
  </p>
  <audio controls>
    <source src="synthesize-text-audio.mp3" />
  </audio>
  <p>
  Output from synthesizing SSML:
  </p>
  <audio controls>
    <source src="synthesize-ssml-audio.mp3" />
  </audio>
  </body>
  <p>
  Output with audio settings:
  </p>
  <audio controls>
    <source src="synthesize-with-settings-audio.ogg" />
  </audio>
</html>
```
- runserver
```shell
python -m http.server 8080
```
## Translate Text with the Cloud Translation API [GSP049]
### Task 1. Create an API Key
- Navigation menu > APIs & Services > Credentials
- + CREATE CREDENTIALS
- API key
- Copy
```shell
export API_KEY= YOUR_API_KEY
```
### Task 2. Translate text
```shell
TEXT="My%20name%20is%20Steve"
curl "https://translation.googleapis.com/language/translate/v2?target=es&key=${API_KEY}&q=${TEXT}"
```
### Task 3. Detect the language
```shell
TEXT_ONE="Meu%20nome%20é%20Steven"
TEXT_TWO="日本のグーグルのオフィスは、東京の六本木ヒルズにあります"
curl -X POST "https://translation.googleapis.com/language/translate/v2/detect?key=${API_KEY}" -d "q=${TEXT_ONE}" -d "q=${TEXT_TWO}"
```

## Speech to Text Transcription with the Cloud Speech API [GSP048]
### Task 1. Create an API key
- Navigation menu > APIs & Services > Credentials
- + CREATE CREDENTIALS
- API key
- Copy
- Navigation menu > Compute Engine > VM Instances
- SSH
```shell
export API_KEY=YOUR_KEY
```
### Task 2. Create your API request
- build request json file: request.json
```json
{
  "config": {
      "encoding":"FLAC",
      "languageCode": "en-US"
  },
  "audio": {
      "uri":"gs://cloud-samples-data/speech/brooklyn_bridge.flac"
  }
}
```
### Task 3. Call the Speech-to-Text API
```shell
curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json \
"https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" > result.json
```
### Task 4. Speech-to-Text transcription in difference languages
- edit request.json
```json
 {
  "config": {
      "encoding":"FLAC",
      "languageCode": "fr"
  },
  "audio": {
      "uri":"gs://cloud-samples-data/speech/corbeau_renard.flac"
  }
}
```
```shell
curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json \
"https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" > result.json
```
## Cloud Speech API 3 Ways: Challenge Lab [ARC132]
### Task 1. Create an API key
- Navigation menu > APIs & Services > Credentials
- + CREATE CREDENTIALS
- API key
- Copy
AIzaSyCsyORKr9CHyN9AvCYsZaLqNcsY9SHBboM
- Navigation menu > Compute Engine > VM Instances
- SSH
```shell
export API_KEY=YOUR_KEY
```
### Task 2. Create synthetic speech from text using the Text-to-Speech API
- enable Cloud Text-to-Speech API
- Create a virtual environment
```shell
source venv/bin/activate
```
- create json file: synthesize-text.json
```json
{
    'input':{
        'text':'Cloud Text-to-Speech API allows developers to include
           natural-sounding, synthetic human speech as playable audio in
           their applications. The Text-to-Speech API converts text or
           Speech Synthesis Markup Language (SSML) input into audio data
           like MP3 or LINEAR16 (the encoding used in WAV files).'
    },
    'voice':{
        'languageCode':'en-gb',
        'name':'en-GB-Standard-A',
        'ssmlGender':'FEMALE'
    },
    'audioConfig':{
        'audioEncoding':'MP3'
    }
}
```
- call text-to-speech api
```shell
curl -H "Authorization: Bearer "$(gcloud auth application-default print-access-token) \
  -H "Content-Type: application/json; charset=utf-8" \
  -d @synthesize-text.json "https://texttospeech.googleapis.com/v1/text:synthesize" \
  > synthesize-text.txt
```
- create tts_decode.py
```python
import argparse
from base64 import decodebytes
import json

"""
Usage:
        python tts_decode.py --input "Filled in at lab start" \
        --output "synthesize-text-audio.mp3"

"""

def decode_tts_output(input_file, output_file):
    """ Decode output from Cloud Text-to-Speech.

    input_file: the response from Cloud Text-to-Speech
    output_file: the name of the audio file to create

    """

    with open(input_file) as input:
        response = json.load(input)
        audio_data = response['audioContent']

        with open(output_file, "wb") as new_file:
            new_file.write(decodebytes(audio_data.encode('utf-8')))

if __name__ == '__main__':
    parser = argparse.ArgumentParser(
        description="Decode output from Cloud Text-to-Speech",
        formatter_class=argparse.RawDescriptionHelpFormatter)
    parser.add_argument('--input',
                       help='The response from the Text-to-Speech API.',
                       required=True)
    parser.add_argument('--output',
                       help='The name of the audio file to create',
                       required=True)

    args = parser.parse_args()
    decode_tts_output(args.input, args.output)
```
- create an audio file
```shell
python tts_decode.py --input "synthesize-text.txt" --output "synthesize-text-audio.mp3"
```

### Task 3. Perform speech to text transcription with the Cloud Speech API
```shell
export API_KEY=YOUR_KEY
```
- Create your API request
  - build request json file: request.json
```json
{
"config": {
"encoding": "FLAC",
"sampleRateHertz": 44100,
"languageCode": "fr-FR"
},
"audio": {
"uri": "gs://cloud-samples-data/speech/corbeau_renard.flac"
}
}
```
  - Call the Speech-to-Text API
```shell
curl -s -X POST -H "Content-Type: application/json" \
--data-binary @speech_request.json \
"https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" \
-o "speech_response_fr.json"
```
### Task 4. Translate text with the Cloud Translation API 
- Create an API Key
- Navigation menu > APIs & Services > Credentials
- + CREATE CREDENTIALS
- API key
- Copy
```shell
export API_KEY= YOUR_API_KEY
```
- Translate text
```shell
export TASK4="これは日本語です。"
curl -s -X POST \
-H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
-H "Content-Type: application/json; charset=utf-8" \
-d "{\"q\": \"$TASK4\"}" \
"https://translation.googleapis.com/language/translate/v2?key=${API_KEY}&source=ja&target=en" > translated_response.txt
```
### Task 5. Detect a language with the Cloud Translation
- Detect the language
```shell
export TASK5="Este%é%japonês."
curl -X POST "https://translation.googleapis.com/language/translate/v2/detect?key=${API_KEY}" -d "q=${TEXT_ONE}" -d "q=${TEXT_TWO}"
curl -s -X POST \
-H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
-H "Content-Type: application/json; charset=utf-8" \
-d "{\"q\": [\"$TASK5\"]}" \
"https://translation.googleapis.com/language/translate/v2/detect?key=${API_KEY}" \
-o "detection_response.txt"
```




