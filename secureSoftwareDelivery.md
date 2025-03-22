# Secure Software Delivery

## Getting Deployments with Binary Authorization [GSP1183]
### Task 0. Environment setup
```shell
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID \
    --format='value(projectNumber)')

gcloud services enable \
  cloudkms.googleapis.com \
  cloudbuild.googleapis.com \
  container.googleapis.com \
  containerregistry.googleapis.com \
  artifactregistry.googleapis.com \
  containerscanning.googleapis.com \
  ondemandscanning.googleapis.com \
  binaryauthorization.googleapis.com 
```

### Task 1. Create Artifact Registry repository
- Create the repository
```shell
gcloud artifacts repositories create artifact-scanning-repo \
  --repository-format=docker \
  --location="REGION" \
  --description="Docker repository"
```
- Configure docker to utilize ur gcloud credentials
```shell
gcloud auth configure-docker "REGION"-docker.pkg.dev
```
- Create and change into directory
```shell
mkdir vuln-scan && cd vuln-scan
```
- create Dockerfile
```shell
cat > ./Dockerfile << EOF
FROM python:3.8-alpine  

# App
WORKDIR /app
COPY . ./

RUN pip3 install Flask==2.1.0
RUN pip3 install gunicorn==20.1.0
RUN pip3 install Werkzeug==2.2.2


CMD exec gunicorn --bind :\$PORT --workers 1 --threads 8 main:app

EOF
```
- Create main.py
```python
cat > ./main.py << EOF
import os
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello_world():
    name = os.environ.get("NAME", "Worlds")
    return "Hello {}!".format(name)

if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
EOF
```
- Use cloud build to build and auto push ur container to artifact registry
```shell
gcloud builds submit . -t "REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image
```
### Task 2. Image Signing
- Image Signing
  - Create an Attestor Note
```shell
cat > ./vulnz_note.json << EOM
{
  "attestation": {
    "hint": {
      "human_readable_name": "Container Vulnerabilities attestation authority"
    }
  }
}
EOM
```
  - Store the Note
```shell
NOTE_ID=vulnz_note

curl -vvv -X POST \
    -H "Content-Type: application/json"  \
    -H "Authorization: Bearer $(gcloud auth print-access-token)"  \
    --data-binary @./vulnz_note.json  \
    "https://containeranalysis.googleapis.com/v1/projects/${PROJECT_ID}/notes/?noteId=${NOTE_ID}"
```
  - Verify the Note
```shell
curl -vvv  \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    "https://containeranalysis.googleapis.com/v1/projects/${PROJECT_ID}/notes/${NOTE_ID}"
```
  - Creating an Attestor
```shell
ATTESTOR_ID=vulnz-attestor

gcloud container binauthz attestors create $ATTESTOR_ID \
    --attestation-authority-note=$NOTE_ID \
    --attestation-authority-note-project=${PROJECT_ID}
```
  - Verify Attestor
```shell
gcloud container binauthz attestors list
```
  - Provide the Binary Authorization access to IAM Role, to view the attestation
```shell
PROJECT_NUMBER=$(gcloud projects describe "${PROJECT_ID}"  --format="value(projectNumber)")

BINAUTHZ_SA_EMAIL="service-${PROJECT_NUMBER}@gcp-sa-binaryauthorization.iam.gserviceaccount.com"


cat > ./iam_request.json << EOM
{
  'resource': 'projects/${PROJECT_ID}/notes/${NOTE_ID}',
  'policy': {
    'bindings': [
      {
        'role': 'roles/containeranalysis.notes.occurrences.viewer',
        'members': [
          'serviceAccount:${BINAUTHZ_SA_EMAIL}'
        ]
      }
    ]
  }
}
EOM
```
  - use the file to create the IAM policy
```shell
curl -X POST  \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    --data-binary @./iam_request.json \
    "https://containeranalysis.googleapis.com/v1/projects/${PROJECT_ID}/notes/${NOTE_ID}:setIamPolicy"
```


### Task 3. Adding a KMS key
- environment variables
```shell
KEY_LOCATION=global
KEYRING=binauthz-keys
KEY_NAME=codelab-key
KEY_VERSION=1
```
- create a keyring to hold a set of keys
```shell
gcloud kms keyrings create "${KEYRING}" --location="${KEY_LOCATION}"
```
- create a new asymmetric signing key pair for the attestor
```shell
gcloud kms keys create "${KEY_NAME}" \
    --keyring="${KEYRING}" --location="${KEY_LOCATION}" \
    --purpose asymmetric-signing   \
    --default-algorithm="ec-sign-p256-sha256"
```
- associate the key to attestor
```shell
gcloud beta container binauthz attestors public-keys add  \
    --attestor="${ATTESTOR_ID}"  \
    --keyversion-project="${PROJECT_ID}"  \
    --keyversion-location="${KEY_LOCATION}" \
    --keyversion-keyring="${KEYRING}" \
    --keyversion-key="${KEY_NAME}" \
    --keyversion="${KEY_VERSION}"
```
- print the list to recheck
```shell
gcloud container binauthz attestors list
```

### Task 4. Creating a signed attestation
- to specify which container image to attest, run
```shell
CONTAINER_PATH="REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image

DIGEST=$(gcloud container images describe ${CONTAINER_PATH}:latest \
    --format='get(image_summary.digest)')
```
- use gcloud to create ur attestation
```shell
gcloud beta container binauthz attestations sign-and-create  \
    --artifact-url="${CONTAINER_PATH}@${DIGEST}" \
    --attestor="${ATTESTOR_ID}" \
    --attestor-project="${PROJECT_ID}" \
    --keyversion-project="${PROJECT_ID}" \
    --keyversion-location="${KEY_LOCATION}" \
    --keyversion-keyring="${KEYRING}" \
    --keyversion-key="${KEY_NAME}" \
    --keyversion="${KEY_VERSION}"
```
- ensure everything worked
```shell
gcloud container binauthz attestations list \
   --attestor=$ATTESTOR_ID --attestor-project=${PROJECT_ID}
```

### Task 5. Admission control policies
- create the GKE cluster with binary authorization enabled
```shell
gcloud beta container clusters create binauthz \
    --zone "ZONE"  \
    --binauthz-evaluation-mode=PROJECT_SINGLETON_POLICY_ENFORCE
```
- allow cloud build to deploy to cluster
```shell
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
        --member="serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
        --role="roles/container.developer"
```
- Allow All policy
  - review existing policy [ALWAYS_ALLOW]
```shell
gcloud container binauthz policy export
```
  - deploy sample to verify you can deploy anything
```shell
kubectl run hello-server --image gcr.io/google-samples/hello-app:1.0 --port 8080
```
  - verify the deploy worked
```shell
kubectl get pods
```
  - delete deployment
```shell
kubectl delete pod hello-server
```
- Deny All policy
  - export current policy
```shell
gcloud container binauthz policy export  > policy.yaml
```
  - edit policy.yaml
```yaml
globalPolicyEvaluationMode: ENABLE
defaultAdmissionRule:
  evaluationMode: ALWAYS_DENY
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
name: projects/PROJECT_ID/policy
```
  - apply the new edited policy
```shell
gcloud container binauthz policy import policy.yaml
```
  - attempt a sample workload deployment
```shell
kubectl run hello-server --image gcr.io/google-samples/hello-app:1.0 --port 8080
```
- Revert the policy to Allow All
  - edit policy.yaml
```yaml
globalPolicyEvaluationMode: ENABLE
defaultAdmissionRule:
  evaluationMode: ALWAYS_ALLOW
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
name: projects/PROJECT_ID/policy
```
  - apply the reversed policy
```shell
gcloud container binauthz policy import policy.yaml
```
### Task 6. Automatically signing images
- Roles needed
  - add binary authorization attestor viewer to cloud build service account
```shell
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role roles/binaryauthorization.attestorsViewer
```
  - add cloud KMS cryptokey signer/verifier role to cloud build service account
```shell
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role roles/cloudkms.signerVerifier
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com \
  --role roles/cloudkms.signerVerifier
```
  - add container analysis notes attacher role to cloud build service account
```shell
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role roles/containeranalysis.notes.attacher
```
- Provide access for Cloud Build Service Account
```shell
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
        --member="serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
        --role="roles/iam.serviceAccountUser"
        
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
        --member="serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
        --role="roles/ondemandscanning.admin"
```
- Prepare the Custom Build Cloud Build Step
```shell
git clone https://github.com/GoogleCloudPlatform/cloud-builders-community.git
cd cloud-builders-community/binauthz-attestation
gcloud builds submit . --config cloudbuild.yaml
cd ../..
rm -rf cloud-builders-community
```
- Add a signing step to your cloudbuild.yaml
  - write a cloudbuild.yaml
```shell
cat > ./cloudbuild.yaml << EOF
steps:

# build
- id: "build"
  name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', '"REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image', '.']
  waitFor: ['-']

# additional CICD checks (not shown)

#Retag
- id: "retag"
  name: 'gcr.io/cloud-builders/docker'
  args: ['tag',  '"REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image', '"REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image:good']


#pushing to artifact registry
- id: "push"
  name: 'gcr.io/cloud-builders/docker'
  args: ['push',  '"REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image:good']


#Sign the image only if the previous severity check passes
- id: 'create-attestation'
  name: 'gcr.io/${PROJECT_ID}/binauthz-attestation:latest'
  args:
    - '--artifact-url'
    - '"REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image:good'
    - '--attestor'
    - 'projects/${PROJECT_ID}/attestors/$ATTESTOR_ID'
    - '--keyversion'
    - 'projects/${PROJECT_ID}/locations/$KEY_LOCATION/keyRings/$KEYRING/cryptoKeys/$KEY_NAME/cryptoKeyVersions/$KEY_VERSION'



images:
  - "REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image:good
EOF
```
  - run the build
```shell
gcloud builds submit
```
- review the build in Cloud Build History
### Task 7. Authorizing signed images
- update GKE policy to require attestation
  - overwrite the policy /w the updated config 
```yaml
COMPUTE_ZONE="REGION"

cat > binauth_policy.yaml << EOM
defaultAdmissionRule:
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  evaluationMode: REQUIRE_ATTESTATION
  requireAttestationsBy:
  - projects/${PROJECT_ID}/attestors/vulnz-attestor
globalPolicyEvaluationMode: ENABLE
clusterAdmissionRules:
  ${COMPUTE_ZONE}.binauthz:
    evaluationMode: REQUIRE_ATTESTATION
    enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
    requireAttestationsBy:
    - projects/${PROJECT_ID}/attestors/vulnz-attestor
EOM
```
  - upload the new policy to Binary Authorization
```shell
gcloud beta container binauthz policy import binauth_policy.yaml
```
- deploy a signed image
  - get the image digest for the good image
```shell
CONTAINER_PATH="REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image

DIGEST=$(gcloud container images describe ${CONTAINER_PATH}:good \
    --format='get(image_summary.digest)')
```
  - use the digest in the kubernetes configuration
```yaml
cat > deploy.yaml << EOM
apiVersion: v1
kind: Service
metadata:
  name: deb-httpd
spec:
  selector:
    app: deb-httpd
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deb-httpd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deb-httpd
  template:
    metadata:
      labels:
        app: deb-httpd
    spec:
      containers:
      - name: deb-httpd
        image: ${CONTAINER_PATH}@${DIGEST}
        ports:
        - containerPort: 8080
        env:
          - name: PORT
            value: "8080"

EOM
```
  - deploy the app to GKE
```shell
kubectl apply -f deploy.yaml
```

### Task 8. Blocked unsigned Images
- Build an Image
  - use local docker to build the image to cache
```shell
docker build -t "REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image:bad .
```
  - push the unsigned image to repo
```shell
docker push "REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image:bad
```
  - get the image digest for the bad image
```shell
CONTAINER_PATH="REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image

DIGEST=$(gcloud container images describe ${CONTAINER_PATH}:bad \
    --format='get(image_summary.digest)')
```
  - use the digest in the kubernetes configuration
```yaml
cat > deploy.yaml << EOM
apiVersion: v1
kind: Service
metadata:
  name: deb-httpd
spec:
  selector:
    app: deb-httpd
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deb-httpd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deb-httpd
  template:
    metadata:
      labels:
        app: deb-httpd
    spec:
      containers:
      - name: deb-httpd
        image: ${CONTAINER_PATH}@${DIGEST}
        ports:
        - containerPort: 8080
        env:
          - name: PORT
            value: "8080"

EOM
```
  - attempt to deploy the app to GKE
```shell
kubectl apply -f deploy.yaml
```

## Secure Builds with Cloud Build [GSP1184]
### Task 0. Environment setup
```shell
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID \
    --format='value(projectNumber)')

gcloud services enable \
  cloudkms.googleapis.com \
  cloudbuild.googleapis.com \
  container.googleapis.com \
  containerregistry.googleapis.com \
  artifactregistry.googleapis.com \
  containerscanning.googleapis.com \
  ondemandscanning.googleapis.com \
  binaryauthorization.googleapis.com
```

### Task 1. Build Images with Cloud Build
- Provide access for Cloud Build Service Account
```shell
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
        --member="serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
        --role="roles/iam.serviceAccountUser"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
        --member="serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
        --role="roles/ondemandscanning.admin"
```
- Create and change into a work directory
```shell
mkdir vuln-scan && cd vuln-scan
```
- Define a sample image
```shell
cat > ./Dockerfile << EOF
FROM gcr.io/google-appengine/debian10@sha256:d25b680d69e8b386ab189c3ab45e219fededb9f91e1ab51f8e999f3edc40d2a1

# System
RUN apt update && apt install python3-pip -y

# App
WORKDIR /app
COPY . ./

RUN pip3 install Flask==1.1.4  
RUN pip3 install gunicorn==20.1.0  

CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 --timeout 0 main:app

EOF
```
- Create a file called main.py with the following contents
```shell
cat > ./main.py << EOF
import os
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello_world():
    name = os.environ.get("NAME", "Worlds")
    return "Hello {}!".format(name)

if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
EOF
```
- Create the Cloud Build pipeline
  - create cloudbuild.yalm
```shell
cat > ./cloudbuild.yaml << EOF
steps:

# build
- id: "build"
  name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', '"REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image', '.']
  waitFor: ['-']


EOF
```
  - run CI pipeline
```shell
gcloud builds submit
```
### Task 2. Use Artifact Registry for Containers
- Create Artifact Registry repository
    - create the repository with the following cmd
```shell
gcloud artifacts repositories create artifact-scanning-repo \
  --repository-format=docker \
  --location="REGION" \
  --description="Docker repository"
```
    - configure docker to utilize ur gcloud credentials when accessing Artifact registry
```shell
gcloud auth configure-docker "REGION"-docker.pkg.dev
```
    - Modify the Cloud Build pipeline to push the resulting image to Artifact Registry
```shell
cat > ./cloudbuild.yaml << EOF
steps:

# build
- id: "build"
  name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', '"REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image', '.']
  waitFor: ['-']

# push to artifact registry
- id: "push"
  name: 'gcr.io/cloud-builders/docker'
  args: ['push',  '"REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image']

images:
  - "REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image
EOF
```
    - run the CI pipeline
```shell
gcloud builds submit
```
### Task 3. Automated vulnerability scanning
- Review Image Details
  - Cloud console > Artifact Registry
  - artifact-scanning-repo
  - click into image details
  - click into the latest digest of your image
  - Once the scan has finished, click on the Vulnerabilities tab

### Task 4. On Demand Scanning
- use the local docker to build the image to your local cache
```shell
docker build -t "REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image .
```
- once the image has been build, request a scan of the image
```shell
gcloud artifacts docker images scan \
    "REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image \
    --format="value(response.scan)" > scan_id.txt
```
- Review the output which was stored in the scan_id.txt
```shell
cat scan_id.txt
```
- to view the actual results of the scan, use the list-vulnerabilities cmd on the report location noted in the output file
```shell
gcloud artifacts docker images list-vulnerabilities $(cat scan_id.txt)
```
- Use the commands below to read the report details and log if any CRITICAL vulnerabilities were found
```shell
export SEVERITY=CRITICAL

gcloud artifacts docker images list-vulnerabilities $(cat scan_id.txt) --format="value(vulnerability.effectiveSeverity)" | if grep -Fxq ${SEVERITY}; then echo "Failed vulnerability check for ${SEVERITY} level"; else echo "No ${SEVERITY} Vulnerabilities found"; fi
```

### Task 5. Use Artifact Registry in CI/CD in Cloud Build
- Provide access with the following commands
```shell
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
        --member="serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
        --role="roles/iam.serviceAccountUser"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
        --member="serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
        --role="roles/ondemandscanning.admin"
```
- update the cloud build pipeline, create cloudbuild.yaml
```shell
cat > ./cloudbuild.yaml << EOF
steps:

# build
- id: "build"
  name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', '"REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image', '.']
  waitFor: ['-']

#Run a vulnerability scan at _SECURITY level
- id: scan
  name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    (gcloud artifacts docker images scan \
    "REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image \
    --location us \
    --format="value(response.scan)") > /workspace/scan_id.txt

#Analyze the result of the scan
- id: severity check
  name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
      gcloud artifacts docker images list-vulnerabilities \$(cat /workspace/scan_id.txt) \
      --format="value(vulnerability.effectiveSeverity)" | if grep -Fxq CRITICAL; \
      then echo "Failed vulnerability check for CRITICAL level" && exit 1; else echo "No CRITICAL vulnerability found, congrats !" && exit 0; fi

#Retag
- id: "retag"
  name: 'gcr.io/cloud-builders/docker'
  args: ['tag',  '"REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image', '"REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image:good']


#pushing to artifact registry
- id: "push"
  name: 'gcr.io/cloud-builders/docker'
  args: ['push',  '"REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image:good']

images:
  - "REGION"-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image
EOF
```
- Submit the build for processing to verify that the build breaks when a CRITICAL severity vulnerability is found
```shell
gcloud builds submit
```
- review build failure in the Cloud Build History
- Fix the Vulnerability
  - overwrite the Dockerfile
```shell
cat > ./Dockerfile << EOF
FROM python:3.8-alpine 


# App
WORKDIR /app
COPY . ./

RUN pip3 install Flask==2.1.0
RUN pip3 install gunicorn==20.1.0
RUN pip3 install Werkzeug==2.2.2

CMD exec gunicorn --bind :\$PORT --workers 1 --threads 8 main:app

EOF
```
  - submit build
```shell
gcloud builds submit
```
## Securing Container Builds [GSP1185]
### Task 0. Environment setup
```shell
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
gcloud services enable artifactregistry.googleapis.com
git clone https://github.com/GoogleCloudPlatform/java-docs-samples
cd java-docs-samples/container-registry/container-analysis
```
### Task 1. Standard repositories
- create a standard maven repository for java artifacts
```shell
gcloud artifacts repositories create container-dev-java-repo \
    --repository-format=maven \
    --location=us-central1 \
    --description="Java package repository for Container Dev Workshop"
```
- Cloud Console > Artifact Registry > Repositories
- Review repo in the terminal
```shell
gcloud artifacts repositories describe container-dev-java-repo \
    --location=us-central1
```

### Task 2. Configure Maven for Artifact Registry
- print settings [repo config]
```shell
gcloud artifacts print-settings mvn \
    --repository=container-dev-java-repo \
    --location=us-central1
```
- open editor
```shell
cloudshell workspace .
```
- copy pom.xml
```xml
  ...

  <distributionManagement>
    <snapshotRepository>
      <id>artifact-registry</id>
      <url>artifactregistry://us-central1-maven.pkg.dev/qwiklabs-gcp-04-3c51830ea757/container-dev-java-repo</url>
    </snapshotRepository>
    <repository>
      <id>artifact-registry</id>
      <url>artifactregistry://us-central1-maven.pkg.dev/qwiklabs-gcp-04-3c51830ea757/container-dev-java-repo</url>
    </repository>
  </distributionManagement>

  <repositories>
    <repository>
      <id>artifact-registry</id>
      <url>artifactregistry://us-central1-maven.pkg.dev/qwiklabs-gcp-04-3c51830ea757/container-dev-java-repo</url>
      <releases>
        <enabled>true</enabled>
      </releases>
      <snapshots>
        <enabled>true</enabled>
      </snapshots>
    </repository>
  </repositories>

  <build>
    <extensions>
      <extension>
        <groupId>com.google.cloud.artifactregistry</groupId>
        <artifactId>artifactregistry-maven-wagon</artifactId>
        <version>2.2.0</version>
      </extension>
    </extensions>
  </build>

</project>
```
- upload java package to artifact registry
```shell
mvn deploy -DskipTests
```
- Artifact Registry > Repositories
- container-dev-java-repo
  
### Task 3. Remote repositories
- create remote repo
```shell
gcloud artifacts repositories create maven-central-cache \
    --project=$PROJECT_ID \
    --repository-format=maven \
    --location=us-central1 \
    --description="Remote repository for Maven Central caching" \
    --mode=remote-repository \
    --remote-repo-config-desc="Maven Central" \
    --remote-mvn-repo=MAVEN-CENTRAL
```
- Artifact Registry > Repositories
- maven-central-cache
- review the repo in terminal
```shell
gcloud artifacts repositories describe maven-central-cache \
    --location=us-central1
```
- print repo config
```shell
gcloud artifacts print-settings mvn \
    --repository=maven-central-cache \
    --location=us-central1
```
- add repository section to pom.xml
- change ID -> central
- create extensions.xml
```shell
mkdir .mvn 
cat > .mvn/extensions.xml << EOF
<extensions xmlns="http://maven.apache.org/EXTENSIONS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/EXTENSIONS/1.0.0 http://maven.apache.org/xsd/core-extensions-1.0.0.xsd">
  <extension>
    <groupId>com.google.cloud.artifactregistry</groupId>
    <artifactId>artifactregistry-maven-wagon</artifactId>
    <version>2.2.0</version>
  </extension>
</extensions>
EOF
```
- compile app
```shell
rm -rf ~/.m2/repository 
mvn compile
```
- Artifact Registry > Repositories

### Task 4. Virtual repositories
- Create a policy file
```shell
cat > ./policy.json << EOF
[
  {
    "id": "private",
    "repository": "projects/${PROJECT_ID}/locations/us-central1/repositories/container-dev-java-repo",
    "priority": 100
  },
  {
    "id": "central",
    "repository": "projects/${PROJECT_ID}/locations/us-central1/repositories/maven-central-cache",
    "priority": 80
  }
]

EOF
```
- Create the virtual repository
```shell
gcloud artifacts repositories create virtual-maven-repo \
    --project=${PROJECT_ID} \
    --repository-format=maven \
    --mode=virtual-repository \
    --location=us-central1 \
    --description="Virtual Maven Repo" \
    --upstream-policy-file=./policy.json
```
- print repo config
```shell
gcloud artifacts print-settings mvn \
    --repository=virtual-maven-repo \
    --location=us-central1
```
- replace repositories section from the output
- Pull dependencies from the Virtual Repository
  - recreate cache repository
```shell
gcloud artifacts repositories delete maven-central-cache \
    --project=$PROJECT_ID \
    --location=us-central1 \
    --quiet

gcloud artifacts repositories create maven-central-cache \
    --project=$PROJECT_ID \
    --repository-format=maven \
    --location=us-central1 \
    --description="Remote repository for Maven Central caching" \
    --mode=remote-repository \
    --remote-repo-config-desc="Maven Central" \
    --remote-mvn-repo=MAVEN-CENTRAL

```
  - build project
```shell
rm -rf ~/.m2/repository 
mvn compile
```
## Secure Software Delivery: Challenge Lab [GSP512]
### Task 1. Enable APIs and Set up the environment
```shell
gcloud services enable \
  cloudkms.googleapis.com \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  container.googleapis.com \
  containerregistry.googleapis.com \
  artifactregistry.googleapis.com \
  containerscanning.googleapis.com \
  ondemandscanning.googleapis.com \
  binaryauthorization.googleapis.com
mkdir sample-app && cd sample-app
gcloud storage cp gs://spls/gsp521/* .
```
- create two Artifact Registry repositories
```shell
export ZONE=$(gcloud compute project-info describe \
--format="value(commonInstanceMetadata.items[google-compute-default-zone])")
export REGION=$(echo "$ZONE" | cut -d '-' -f 1-2)
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID \
    --format='value(projectNumber)')

gcloud artifacts repositories create artifact-scanning-repo \
  --repository-format=docker \
  --location=$REGION \
  --description="Docker Scanning repository"

gcloud artifacts repositories create artifact-prod-repo \
  --repository-format=docker \
  --location=$REGION \
  --description="Docker Production repository"

gcloud auth configure-docker $REGION-docker.pkg.dev
```
### Task 2. Create the Cloud Build pipeline
- add Cloud Build service account roles
```shell
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
        --member="serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
        --role="roles/iam.serviceAccountUser"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
        --member="serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
        --role="roles/ondemandscanning.admin"
```
- open and complete sample-app/cloudbuild.yaml
```shell
cat > cloudbuild.yaml <<EOF
steps:

# TODO: #1. Build Step. Replace the <image-name> placeholder with the correct value.
- id: "build"
  name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', '${REGION}-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image', '.']
  waitFor: ['-']

# TODO: #2. Push to Artifact Registry. Replace the <image-name> placeholder with the correct value.
- id: "push"
  name: 'gcr.io/cloud-builders/docker'
  args: ['push',  '${REGION}-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image']


## More steps will be added here in a later section

# TODO: #8. Replace <image-name> placeholder with the value from the build step.
images:
  - ${REGION}-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image
EOF
```
- submit the build
```shell
gcloud builds submit
```

### Task 3. Set up Binary Authorization
- create an Attestor
```shell
cat > ./vulnz_note.json << EOM
{
  "attestation": {
    "hint": {
      "human_readable_name": "Container Vulnerabilities attestation authority"
    }
  }
}
EOM
```
- store the note
```shell
NOTE_ID=vulnerability_note

curl -vvv -X POST \
    -H "Content-Type: application/json"  \
    -H "Authorization: Bearer $(gcloud auth print-access-token)"  \
    --data-binary @./vulnerability_note.json  \
    "https://containeranalysis.googleapis.com/v1/projects/${PROJECT_ID}/notes/?noteId=${NOTE_ID}"
```
- use the container analysis api
```shell
curl -vvv  \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    "https://containeranalysis.googleapis.com/v1/projects/${PROJECT_ID}/notes/${NOTE_ID}"
```
- create Attestor
```shell
ATTESTOR_ID=vulnerability-attestor

gcloud container binauthz attestors create $ATTESTOR_ID \
    --attestation-authority-note=$NOTE_ID \
    --attestation-authority-note-project=${PROJECT_ID}
```
- verify
```shell
gcloud container binauthz attestors list
```
- construct IAM policy
```shell
PROJECT_NUMBER=$(gcloud projects describe "${PROJECT_ID}"  --format="value(projectNumber)")

BINAUTHZ_SA_EMAIL="service-${PROJECT_NUMBER}@gcp-sa-binaryauthorization.iam.gserviceaccount.com"


cat > ./iam_request.json << EOM
{
  'resource': 'projects/${PROJECT_ID}/notes/${NOTE_ID}',
  'policy': {
    'bindings': [
      {
        'role': 'roles/containeranalysis.notes.occurrences.viewer',
        'members': [
          'serviceAccount:${BINAUTHZ_SA_EMAIL}'
        ]
      }
    ]
  }
}
EOM

curl -X POST  \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    --data-binary @./iam_request.json \
    "https://containeranalysis.googleapis.com/v1/projects/${PROJECT_ID}/notes/${NOTE_ID}:setIamPolicy"
```
- Generate a KMS Pair
  - environment setup
```shell
KEY_LOCATION=global
KEYRING=binauthz-keys
KEY_NAME=lab-key
KEY_VERSION=1
```
  - create a keyring to hold a set of keys
```shell
gcloud kms keyrings create "${KEYRING}" --location="${KEY_LOCATION}"
```
  - create a asymmetric signing key pair
```shell
gcloud kms keys create "${KEY_NAME}" \
    --keyring="${KEYRING}" --location="${KEY_LOCATION}" \
    --purpose asymmetric-signing   \
    --default-algorithm="ec-sign-p256-sha256"
```
- link key to Attestor
```shell
gcloud beta container binauthz attestors public-keys add  \
    --attestor="${ATTESTOR_ID}"  \
    --keyversion-project="${PROJECT_ID}"  \
    --keyversion-location="${KEY_LOCATION}" \
    --keyversion-keyring="${KEYRING}" \
    --keyversion-key="${KEY_NAME}" \
    --keyversion="${KEY_VERSION}"
```
- update Binary Authorization Policy
```shell
gcloud container binauthz policy export > my_policy.yaml

cat > my_policy.yaml << EOM
defaultAdmissionRule:
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  evaluationMode: REQUIRE_ATTESTATION
  requireAttestationsBy:
    - projects/${PROJECT_ID}/attestors/vulnerability-attestor
globalPolicyEvaluationMode: ENABLE
name: projects/${PROJECT_ID}/policy
EOM

gcloud container binauthz policy import my_policy.yaml
```
  - 
### Task 4. Create a Cloud Build CI/CD pipeline with vulnerability scanning
- grant the Cloud Build service account IAM roles
```shell
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role roles/binaryauthorization.attestorsViewer \

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role roles/cloudkms.signerVerifier

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com \
  --role roles/cloudkms.signerVerifier

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role roles/containeranalysis.notes.attacher

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
        --member="serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
        --role="roles/iam.serviceAccountUser"
        
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
        --member="serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" \
        --role="roles/ondemandscanning.admin"

```
- install the custom build step
```shell
git clone https://github.com/GoogleCloudPlatform/cloud-builders-community.git
cd cloud-builders-community/binauthz-attestation
gcloud builds submit . --config cloudbuild.yaml
cd ../..
rm -rf cloud-builders-community

cat > ./cloudbuild.yaml << EOF
steps:

- id: "build"
  name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'us-central1-docker.pkg.dev/qwiklabs-gcp-01-d814794f4541/artifact-scanning-repo/sample-image:latest', '.']
  waitFor: ['-']

- id: "push"
  name: 'gcr.io/cloud-builders/docker'
  args: ['push',  'us-central1-docker.pkg.dev/qwiklabs-gcp-01-d814794f4541/artifact-scanning-repo/sample-image:latest']

- id: scan
  name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
      (gcloud artifacts docker images scan \
      us-central1-docker.pkg.dev/qwiklabs-gcp-01-d814794f4541/artifact-scanning-repo/sample-image:latest \
      --location us \
      --format="value(response.scan)") > /workspace/scan_id.txt

- id: severity check
  name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
      gcloud artifacts docker images list-vulnerabilities $(cat /workspace/scan_id.txt) \
      --format="value(vulnerability.effectiveSeverity)" | if grep -Fxq CRITICAL; \
      then echo "Failed vulnerability check for CRITICAL level" && exit 1; else echo \
      "No CRITICAL vulnerability found, congrats !" && exit 0; fi

- id: 'create-attestation'
  name: 'gcr.io/qwiklabs-gcp-01-d814794f4541/binauthz-attestation:latest'
  args:
    - '--artifact-url'
    - 'us-central1-docker.pkg.dev/qwiklabs-gcp-01-d814794f4541/artifact-scanning-repo/sample-image:latest'
    - '--attestor'
    - 'projects/qwiklabs-gcp-01-d814794f4541/attestors/vulnerability-attestor'
    - '--keyversion'
    - 'projects/qwiklabs-gcp-01-d814794f4541/locations/global/keyRings/binauthz-keys/cryptoKeys/lab-key/cryptoKeyVersions/1'

- id: "push-to-prod"
  name: 'gcr.io/cloud-builders/docker'
  args: 
    - 'tag' 
    - 'us-central1-docker.pkg.dev/qwiklabs-gcp-01-d814794f4541/artifact-scanning-repo/sample-image:latest'
    - 'us-central1-docker.pkg.dev/qwiklabs-gcp-01-d814794f4541/artifact-prod-repo/sample-image:latest'
- id: "push-to-prod-final"
  name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'us-central1-docker.pkg.dev/qwiklabs-gcp-01-d814794f4541/artifact-prod-repo/sample-image:latest']

- id: 'deploy-to-cloud-run'
  name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    gcloud run deploy auth-service --image=us-central1-docker.pkg.dev/qwiklabs-gcp-01-d814794f4541/artifact-scanning-repo/sample-image:latest     --binary-authorization=default --region=us-central1 --allow-unauthenticated

images:
  - us-central1-docker.pkg.dev/qwiklabs-gcp-01-d814794f4541/artifact-scanning-repo/sample-image:latest

EOF

gcloud builds submit
```

### Task 5. Fix the vulnerability and redeploy the CI/CD pipeline
- update Dockerfile
```dockerfile
FROM python:3.8-alpine
 
# App
WORKDIR /app
COPY . ./
 
RUN pip3 install Flask==3.0.3
RUN pip3 install gunicorn==23.0.0
RUN pip3 install Werkzeug==3.0.4
 
CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 main:app
```
- Re-trigger the Build
```shell
gcloud builds submit
```
- Verify Build Success

