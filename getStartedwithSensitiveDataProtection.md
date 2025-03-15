# Get Started with Sensitive Data Protection

# Cloud Data Loss Prevention API: Qwik Start [GSP107]
## Task 1. Inspect a String for sensitive information
- create JSON file: inspect-request.json
```json
{
  "item":{
    "value":"My phone number is (206) 555-0123."
  },
  "inspectConfig":{
    "infoTypes":[
      {
        "name":"PHONE_NUMBER"
      },
      {
        "name":"US_TOLLFREE_PHONE_NUMBER"
      }
    ],
    "minLikelihood":"POSSIBLE",
    "limits":{
      "maxFindingsPerItem":0
    },
    "includeQuote":true
  }
}
```
- obtain an authorization token using ur acc
```shell
gcloud auth print-access-token
```
- curl to make a content:inspect request
```shell
curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" https://dlp.googleapis.com/v2/projects/qwiklabs-gcp-03-a06a3de59484/content:inspect -d @inspect-request.json -o inspect-output.txt
```
- display output
```shell
cat inspect-output.txt
```
- upload output to Cloud Storage
```shell
gsutil cp inspect-output.txt gs://bucket_name_filled_after_lab_start
```

## Task 2. Redacting sensitive data from text content
- create json file: new-inspect-file.json
```json
{
  "item": {
     "value":"My email is test@gmail.com",
   },
   "deidentifyConfig": {
     "infoTypeTransformations":{
          "transformations": [
            {
              "primitiveTransformation": {
                "replaceWithInfoTypeConfig": {}
              }
            }
          ]
        }
    },
    "inspectConfig": {
      "infoTypes": {
        "name": "EMAIL_ADDRESS"
      }
    }
}
```
- curl to request content:deidentity
```shell
curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" https://dlp.googleapis.com/v2/projects/qwiklabs-gcp-03-a06a3de59484/content:deidentify -d @new-inspect-file.json -o redact-output.txt
```
- display the output
```shell
cat redact-output.txt
```
- upload to cloud
```shell
gsutil cp redact-output.txt gs://bucket_name_filled_after_lab_start
```

# Redacting Critical Data with Sensitive Data Protection [GSP864]
## Task 1. Clone the repo and enable APIs
- clone the repo
```shell
git clone https://github.com/googleapis/synthtool
cd synthtool/tests/fixtures/nodejs-dlp/samples/ && npm install
export PROJECT_ID=qwiklabs-gcp-01-29942b854572
gcloud config set project $PROJECT_ID
```
- enable APIs
```shell
gcloud services enable dlp.googleapis.com cloudkms.googleapis.com --project $PROJECT_ID
```
## Task 2. Inspect strings and files
- inspect string for sensitive info types
```javascript
// Copyright 2020 Google LLC
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//      http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

'use strict';

// sample-metadata:
//  title: Inspects strings
//  description: Inspect a string using the Data Loss Prevention API.
//  usage: node inspectString.js my-project string minLikelihood maxFindings infoTypes customInfoTypes includeQuote

function main(
  projectId,
  string,
  minLikelihood,
  maxFindings,
  infoTypes,
  customInfoTypes,
  includeQuote
) {
  [infoTypes, customInfoTypes] = transformCLI(infoTypes, customInfoTypes);

  // [START dlp_inspect_string]
  // Imports the Google Cloud Data Loss Prevention library
  const DLP = require('@google-cloud/dlp');

  // Instantiates a client
  const dlp = new DLP.DlpServiceClient();

  // The project ID to run the API call under
  // const projectId = 'my-project';

  // The string to inspect
  // const string = 'My name is Gary and my email is gary@example.com';

  // The minimum likelihood required before returning a match
  // const minLikelihood = 'LIKELIHOOD_UNSPECIFIED';

  // The maximum number of findings to report per request (0 = server maximum)
  // const maxFindings = 0;

  // The infoTypes of information to match
  // const infoTypes = [{ name: 'PHONE_NUMBER' }, { name: 'EMAIL_ADDRESS' }, { name: 'CREDIT_CARD_NUMBER' }];

  // The customInfoTypes of information to match
  // const customInfoTypes = [{ infoType: { name: 'DICT_TYPE' }, dictionary: { wordList: { words: ['foo', 'bar', 'baz']}}},
  //   { infoType: { name: 'REGEX_TYPE' }, regex: {pattern: '\\(\\d{3}\\) \\d{3}-\\d{4}'}}];

  // Whether to include the matching string
  // const includeQuote = true;

  async function inspectString() {
    // Construct item to inspect
    const item = {value: string};

    // Construct request
    const request = {
      parent: `projects/${projectId}/locations/global`,
      inspectConfig: {
        infoTypes: infoTypes,
        customInfoTypes: customInfoTypes,
        minLikelihood: minLikelihood,
        includeQuote: includeQuote,
        limits: {
          maxFindingsPerRequest: maxFindings,
        },
      },
      item: item,
    };

    console.log(request.inspectConfig.infoTypes);
    console.log(Array.isArray(request.inspectConfig.infoTypes));

    // Run request
    const [response] = await dlp.inspectContent(request);
    const findings = response.result.findings;
    if (findings.length > 0) {
      console.log('Findings:');
      findings.forEach(finding => {
        if (includeQuote) {
          console.log(`\tQuote: ${finding.quote}`);
        }
        console.log(`\tInfo type: ${finding.infoType.name}`);
        console.log(`\tLikelihood: ${finding.likelihood}`);
      });
    } else {
      console.log('No findings.');
    }
  }
  inspectString();
  // [END dlp_inspect_string]
}

main(...process.argv.slice(2));
process.on('unhandledRejection', err => {
  console.error(err.message);
  process.exitCode = 1;
});

function transformCLI(infoTypes, customInfoTypes) {
  infoTypes = infoTypes
    ? infoTypes.split(',').map(type => {
        return {name: type};
      })
    : undefined;

  if (customInfoTypes) {
    customInfoTypes = customInfoTypes.includes(',')
      ? customInfoTypes.split(',').map((dict, idx) => {
          return {
            infoType: {name: 'CUSTOM_DICT_'.concat(idx.toString())},
            dictionary: {wordList: {words: dict.split(',')}},
          };
        })
      : customInfoTypes.split(',').map((rgx, idx) => {
          return {
            infoType: {name: 'CUSTOM_REGEX_'.concat(idx.toString())},
            regex: {pattern: rgx},
          };
        });
  }

  return [infoTypes, customInfoTypes];
}
```
```shell
node inspectString.js $PROJECT_ID "My email address is jenny@somedomain.com and you can call me at 555-867-5309" > inspected-string.txt
cat inspected-string.txt 
cat resources/accounts.txt 
node inspectFile.js $PROJECT_ID resources/accounts.txt > inspected-file.txt
cat inspected-string.txt 
cat inspected-file.txt 
gsutil cp inspected-string.txt gs://qwiklabs-gcp-01-29942b854572-bucket
gsutil cp inspected-file.txt gs://qwiklabs-gcp-01-29942b854572-bucket
```
## Task 3. De-identification
- detects sensitive data as defined by info types
```javascript
// Copyright 2020 Google LLC
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//      http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

'use strict';

// sample-metadata:
//  title: Deidentify with Mask
//  description: Deidentify sensitive data in a string by masking it with a character.
//  usage: node deidentifyWithMask.js my-project string maskingCharacter numberToMask

function main(projectId, string, maskingCharacter, numberToMask) {
  // [START dlp_deidentify_masking]
  // Imports the Google Cloud Data Loss Prevention library
  const DLP = require('@google-cloud/dlp');

  // Instantiates a client
  const dlp = new DLP.DlpServiceClient();

  // The project ID to run the API call under
  // const projectId = 'my-project-id';

  // The string to deidentify
  // const string = 'My SSN is 372819127';

  // (Optional) The maximum number of sensitive characters to mask in a match
  // If omitted from the request or set to 0, the API will mask any matching characters
  // const numberToMask = 5;

  // (Optional) The character to mask matching sensitive data with
  // const maskingCharacter = 'x';

  // Construct deidentification request
  const item = {value: string};

  async function deidentifyWithMask() {
    const request = {
      parent: `projects/${projectId}/locations/global`,
      deidentifyConfig: {
        infoTypeTransformations: {
          transformations: [
            {
              primitiveTransformation: {
                characterMaskConfig: {
                  maskingCharacter: maskingCharacter,
                  numberToMask: numberToMask,
                },
              },
            },
          ],
        },
      },
      item: item,
    };

    // Run deidentification request
    const [response] = await dlp.deidentifyContent(request);
    const deidentifiedItem = response.item;
    console.log(deidentifiedItem.value);
  }

  deidentifyWithMask();
  // [END dlp_deidentify_masking]
}

main(...process.argv.slice(2));
process.on('unhandledRejection', err => {
  console.error(err.message);
  process.exitCode = 1;
});
```
```shell
node deidentifyWithMask.js $PROJECT_ID "My order number is F12312399. Email me at anthony@somedomain.com" > de-identify-output.txt
cat de-identify-output.txt
gsutil cp de-identify-output.txt gs://qwiklabs-gcp-01-29942b854572-bucket
```
## Task 4. Redact strings and images
- redaction string
```javascript
// Copyright 2020 Google LLC
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//      http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

// sample-metadata:
//  title: Redact Text
//  description: Redact sensitive data from text using the Data Loss Prevention API.
//  usage: node redactText.js my-project string minLikelihood infoTypes

function main(projectId, string, minLikelihood, infoTypes) {
  infoTypes = transformCLI(infoTypes);
  // [START dlp_redact_text]
  // Imports the Google Cloud Data Loss Prevention library
  const DLP = require('@google-cloud/dlp');

  // Instantiates a client
  const dlp = new DLP.DlpServiceClient();

  // The project ID to run the API call under
  // const projectId = 'my-project';

  // Construct transformation config which replaces sensitive info with its info type.
  // E.g., "Her email is xxx@example.com" => "Her email is [EMAIL_ADDRESS]"
  const replaceWithInfoTypeTransformation = {
    primitiveTransformation: {
      replaceWithInfoTypeConfig: {},
    },
  };

  async function redactText() {
    // Construct redaction request
    const request = {
      parent: `projects/${projectId}/locations/global`,
      item: {
        value: string,
      },
      deidentifyConfig: {
        infoTypeTransformations: {
          transformations: [replaceWithInfoTypeTransformation],
        },
      },
      inspectConfig: {
        minLikelihood: minLikelihood,
        infoTypes: infoTypes,
      },
    };

    // Run string redaction
    const [response] = await dlp.deidentifyContent(request);
    const resultString = response.item.value;
    console.log(`Redacted text: ${resultString}`);
  }
  redactText();
  // [END dlp_redact_text]
}

main(...process.argv.slice(2));
process.on('unhandledRejection', err => {
  console.error(err.message);
  process.exitCode = 1;
});

function transformCLI(infoTypes) {
  infoTypes = infoTypes
    ? infoTypes.split(',').map(type => {
        return {name: type};
      })
    : undefined;
  return infoTypes;
}
```
```shell
node redactText.js $PROJECT_ID  "Please refund the purchase to my credit card 4012888888881881" CREDIT_CARD_NUMBER > redacted-string.txt
cat redacted-string.txt
```
- redact image
```javascript
// Copyright 2020 Google LLC
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//      http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

// sample-metadata:
//  title: Redact Image
//  description: Redact sensitive data from an image using the Data Loss Prevention API.
//  usage: node redactImage.js my-project filepath minLikelihood infoTypes outputPath

function main(projectId, filepath, minLikelihood, infoTypes, outputPath) {
  infoTypes = transformCLI(infoTypes);
  // [START dlp_redact_image]
  // Imports the Google Cloud Data Loss Prevention library
  const DLP = require('@google-cloud/dlp');

  // Imports required Node.js libraries
  const mime = require('mime');
  const fs = require('fs');

  // Instantiates a client
  const dlp = new DLP.DlpServiceClient();

  // The project ID to run the API call under
  // const projectId = 'my-project';

  // The path to a local file to inspect. Can be a JPG or PNG image file.
  // const filepath = 'path/to/image.png';

  // The minimum likelihood required before redacting a match
  // const minLikelihood = 'LIKELIHOOD_UNSPECIFIED';

  // The infoTypes of information to redact
  // const infoTypes = [{ name: 'EMAIL_ADDRESS' }, { name: 'PHONE_NUMBER' }];

  // The local path to save the resulting image to.
  // const outputPath = 'result.png';
  async function redactImage() {
    const imageRedactionConfigs = infoTypes.map(infoType => {
      return {infoType: infoType};
    });

    // Load image
    const fileTypeConstant =
      ['image/jpeg', 'image/bmp', 'image/png', 'image/svg'].indexOf(
        mime.getType(filepath)
      ) + 1;
    const fileBytes = Buffer.from(fs.readFileSync(filepath)).toString('base64');

    // Construct image redaction request
    const request = {
      parent: `projects/${projectId}/locations/global`,
      byteItem: {
        type: fileTypeConstant,
        data: fileBytes,
      },
      inspectConfig: {
        minLikelihood: minLikelihood,
        infoTypes: infoTypes,
      },
      imageRedactionConfigs: imageRedactionConfigs,
    };

    // Run image redaction request
    const [response] = await dlp.redactImage(request);
    const image = response.redactedImage;
    fs.writeFileSync(outputPath, image);
    console.log(`Saved image redaction results to path: ${outputPath}`);
  }
  redactImage();
  // [END dlp_redact_image]
}

main(...process.argv.slice(2));
process.on('unhandledRejection', err => {
  console.error(err.message);
  process.exitCode = 1;
});

function transformCLI(infoTypes) {
  infoTypes = infoTypes
    ? infoTypes.split(',').map(type => {
        return {name: type};
      })
    : undefined;
  return infoTypes;
}
```
```shell
node redactImage.js $PROJECT_ID resources/test.png "" PHONE_NUMBER ./redacted-phone.png
node redactImage.js $PROJECT_ID resources/test.png "" EMAIL_ADDRESS ./redacted-email.png
gsutil cp redacted-string.txt gs://qwiklabs-gcp-01-29942b854572-bucket
gsutil cp redacted-phone.png gs://qwiklabs-gcp-01-29942b854572-bucket
gsutil cp redacted-email.png gs://qwiklabs-gcp-01-29942b854572-bucket
```

# Creating a De-identified Copy of Data in Cloud Storage [GSP1073]
## Task 1. Create de-identify templates
- create a template for unstructured data
  - Navigation menu > Security > Data Loss Prevention
  - Configuration > Templates
  - Create Template
  - Template type: De-identify (remove sensitive data)
  - Template ID: deid_unstruct1
  - Display name: deid_unstruct1 template
  - Description: leave empty
  - Resource location: Global (any region)
  - Continue
  - Transformation Rule: Replace with info Type name
  - InfoTypes to transform: Any detected info Type defined in an inspection template or inspect config that are not specified in other rules
  - Create
- create a template for structured data
  - Navigation menu > Security > Data Loss Prevention
  - Configuration > Templates > Create Template
  - Data transformation type: Record
  - Template ID: deid_struct1
  - Display name: deid_struct1 template
  - Description: leave empty
  - Resource location: Global (any region)
  - Continue
  - Transformation Rule: ssn ccn email vin id agent_id user_id
  - Transformation type: Primitive field transformation
  - Transformation method: Replace
  - +Add Transformation Rule
  - message
  - Transformation type: Match on infoType
  - Add Transformation
  - Transformation Method: Replace with infoType name
  - InfoTypes to transform: Any detected infoTypes defined in an inspection template or inspect config that are not specified in other rules. 
  - Create

## Task 2. Create a DLP inspection job trigger
- Navigation menu > Data Loss Prevention
- Inspection
- Create Job and Job Triggers
- Job ID: DeID_Storage_Demo1
- Location: Global (any region)
- Storage type: Google Cloud Storage
- Location Type: Scan a bucket with optional include/exclude rules
- URL: BUCKET
- "Percentage of included objects scanned within the bucket" => 100% / No Sampling
- Continue
- Configure detection
- Continue
- Add Actions 
- Make a de-identify copy
projects/project-id/locations/global/deidentifyTemplates/deid_unstruct1
projects/project-id/locations/global/deidentify
- Cloud Storage output location: output bucket name
- Continue
- Schedule: Create a trigger to run the job on a periodic schedule
- Weekly
- Continue
- Create > Confirm Create
- Inspection > Job Triggers

## Task 3. Run DLP Inspection and review results
- Navigation menu > Data Loss Protection
- Inspection
- Job Triggers
- Select this job trigger
- Run Now
- triggered jobs
- Done
  
- Configuration
- Output bucket for de-identified Cloud Storage Data

# Get Started with Sensitive Data Protection: Challenge Lab [ARC116]
Challenge scenario
You are working as a junior cloud engineer in your organization. You're part of a team of cloud engineers assigned to using Sensitive Data Protection API's powerful detection engine to protect and screen for personally identifiable information (PII) and other privacy-sensitive data. As part of this project, you are asked to use the Sensitive Data Protection service in Google Cloud to redact sensitive information from text, de-identify sensitive data, and create a DLP template to use for inspecting data.
Your challenge
For this challenge, you have been tasked with redacting and de-identifying sensitive information, and creating templates to inspect structured and unstructured data.

You need to:

Inspect strings and files to perform de-identification.
Create de-identification inspection templates.
Configure a job trigger to run DLP inspections.

## Task 1. Redact sensitive data from text content
- create json file: redact-request.json
```json
{
	"item": {
		"value": "Please update my records with the following information:\n Email address: foo@example.com,\nNational Provider Identifier: 1245319599"
	},
	"deidentifyConfig": {
		"infoTypeTransformations": {
			"transformations": [{
				"primitiveTransformation": {
					"replaceWithInfoTypeConfig": {}
				}
			}]
		}
	},
	"inspectConfig": {
		"infoTypes": [{
				"name": "EMAIL_ADDRESS"
			},
			{
				"name": "US_HEALTHCARE_NPI"
			}
		]
	}
}
```
- curl [be careful with project_id]
```shell
gcloud auth print-access-token

curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" https://dlp.googleapis.com/v2/projects/qwiklabs-gcp-03-8ead4ca62aa2/content:deidentify -d @redact-request.json -o redact-response.txt

cat redact-response.txt

gsutil cp redact-response.txt gs://qwiklabs-gcp-03-8ead4ca62aa2-redact
```

## Task 2. Create DLP inspection templates
- create a de-identify template for structured data with the name: structured_data_template (in Global (any region))
  - Navigation menu > Security > Data Loss Prevention
  - Configuration > Templates > Create Template
  - Data transformation type: Record
  - Template ID: structured_data_template
  - Display name: structured_data_template template
  - Description: leave empty
  - Resource location: Global (any region)
  - Continue
  - Transformation Rule: bank name, zip code
  - Transformation type: Primitive field transformation
  - Transformation method: Mask with character
  - Masking Character: #
  - Mask all Characters: Enable mask all characters checkbox and do not ignore any characters
  - +Add Transformation Rule
  - message
  - Transformation type: Match on infoType
  - Add Transformation
  - Transformation Method: Replace with infoType name
  - InfoTypes to transform: Any detected infoTypes defined in an inspection template or inspect config that are not specified in other rules. 
  - Create
- Create a de-identify template for unstructured data with the name: unstructured_data_template (in Global (any region))
  - Navigation menu > Security > Data Loss Prevention
  - Configuration > Templates
  - Create Template
  - Template type: De-identify (remove sensitive data)
  - Template ID: unstructured_data_template
  - Display name: unstructured_data_template template
  - Description: leave empty
  - Resource location: Global (any region)
  - Continue
  - Transformation Rule: Replace
  - String value: [redacted]
  - Create

## Task 3. Configure a job trigger to ren DLP inspection
- Create a DLP inspection job trigger named dlp_job (in Global (any region))
  - Navigation menu > Data Loss Prevention
  - Inspection
  - Create Job and Job Triggers
  - Job ID: dlp_job
  - Location: Global (any region)
  - Storage type: Google Cloud Storage
  - Location Type: Scan a bucket with optional include/exclude rules
  - URL: BUCKET
  - "Percentage of included objects scanned within the bucket" => 100% / No Sampling
  - Continue
  - Configure detection
  - Continue
  - Add Actions 
  - Make a de-identify copy
  projects/project-id/locations/global/deidentifyTemplates/deid_unstruct1
  projects/project-id/locations/global/deidentify
  - Cloud Storage output location: output bucket name
  - Continue
  - Schedule: Create a trigger to run the job on a periodic schedule
  - Weekly
  - Continue
  - Create > Confirm Create
  - Inspection > Job Triggers
- Run DLP inspection and explore the various folders and files in the Cloud Storage Bucket output bucket to verify the redacted data.
  - Navigation menu > Data Loss Protection
  - Inspection
  - Job Triggers
  - Select this job trigger
  - Run Now
  - triggered jobs
  - Done
