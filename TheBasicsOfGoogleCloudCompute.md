# The Basics of Google Cloud Compute
# Create a Virtual Machine

```shell
sudo apt-get update
sudo apt-get install nginx -y
ps auwx | grep nginx
```

```shell
gcloud compute instances create gcelab2 --machine-type e2-medium --zone=us-east4-a
```

# Creating a Persistent Disk
- Create a new instance
```shell
ZONE=us-east1-d
gcloud compute instances create gcelab --zone $ZONE --machine-type e2-standard-2

```
- Create a new persistent disk
```shell
gcloud compute disks create mydisk --size=200GB \
--zone $ZONE
```
- Attaching a disk
```shell
gcloud compute instances attach-disk gcelab --disk mydisk --zone $ZONE
```
- Finding the persistent disk in the virtual machine
```shell
gcloud compute ssh gcelab --zone $ZONE
ls -l /dev/disk/by-id/
```
- Formatting and mounting the persistent disk
```shell
sudo mkdir /mnt/mydisk
sudo mkfs.ext4 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/disk/by-id/scsi-0Google_PersistentDisk_persistent-disk-1
sudo mount -o discard,defaults /dev/disk/by-id/scsi-0Google_PersistentDisk_persistent-disk-1 /mnt/mydisk
```
- Automatically mount the disk on restart
```shell
sudo nano /etc/fstab
# add following line
# /dev/disk/by-id/scsi-0Google_PersistentDisk_persistent-disk-1 /mnt/mydisk ext4 defaults 1 1
```

# Hosting a Web App on Google Cloud Using Compute Engine
## Task 1. Enable Compute Engine API
```shell
gcloud services enable compute.googleapis.com
```

## Task 2. Create Cloud Storage bucket
```shell
gsutil mb gs://fancy-store-$DEVSHELL_PROJECT_ID
```

## Task 3. Clone source repository
```shell
git clone https://github.com/googlecodelabs/monolith-to-microservices.git
cd ~/monolith-to-microservices
./setup.sh
nvm install --lts
cd microservices
npm start
```

## Task 4. Create Compute Engine instances
- Create a startup script to configure instances.
```shell
touch ~/monolith-to-microservices/startup-script.sh
```
```bash
#!/bin/bash

# Install logging monitor. The monitor will automatically pick up logs sent to
# syslog.
curl -s "https://storage.googleapis.com/signals-agents/logging/google-fluentd-install.sh" | bash
service google-fluentd restart &

# Install dependencies from apt
apt-get update
apt-get install -yq ca-certificates git build-essential supervisor psmisc

# Install nodejs
mkdir /opt/nodejs
curl https://nodejs.org/dist/v16.14.0/node-v16.14.0-linux-x64.tar.gz | tar xvzf - -C /opt/nodejs --strip-components=1
ln -s /opt/nodejs/bin/node /usr/bin/node
ln -s /opt/nodejs/bin/npm /usr/bin/npm

# Get the application source code from the Google Cloud Storage bucket.
mkdir /fancy-store
gsutil -m cp -r gs://fancy-store-[DEVSHELL_PROJECT_ID]/monolith-to-microservices/microservices/* /fancy-store/

# Install app dependencies.
cd /fancy-store/
npm install

# Create a nodeapp user. The application will run as this user.
useradd -m -d /home/nodeapp nodeapp
chown -R nodeapp:nodeapp /opt/app

# Configure supervisor to run the node app.
cat >/etc/supervisor/conf.d/node-app.conf << EOF
[program:nodeapp]
directory=/fancy-store
command=npm start
autostart=true
autorestart=true
user=nodeapp
environment=HOME="/home/nodeapp",USER="nodeapp",NODE_ENV="production"
stdout_logfile=syslog
stderr_logfile=syslog
EOF

supervisorctl reread
supervisorctl update
```
    - check LF not CRLF at the bottom of the right
```shell
gsutil cp ~/monolith-to-microservices/startup-script.sh gs://fancy-store-$DEVSHELL_PROJECT_ID
```

- Clone source code and upload to Cloud - Storage.
```shell
cd ~
rm -rf monolith-to-microservices/*/node_modules
gsutil -m cp -r monolith-to-microservices gs://fancy-store-$DEVSHELL_PROJECT_ID/
```

- Deploy a Compute Engine instance to host the backend microservices.
```shell
gcloud compute instances create backend \
    --zone=$ZONE \
    --machine-type=e2-standard-2 \
    --tags=backend \
   --metadata=startup-script-url=https://storage.googleapis.com/fancy-store-$DEVSHELL_PROJECT_ID/startup-script.sh
gcloud compute instances list
```

- Reconfigure the frontend code to utilize the backend microservices instance.
monolith-to-microservices > react-app
View > Toggle Hidden Files > .env
replace localhost with BACKEND_ADDRESS copied from above step
save
```shell
cd ~/monolith-to-microservices/react-app
npm install && npm run-script build
cd ~
rm -rf monolith-to-microservices/*/node_modules
gsutil -m cp -r monolith-to-microservices gs://fancy-store-$DEVSHELL_PROJECT_ID/
```

- Deploy a Compute Engine instance to host the frontend microservice.
```shell
gcloud compute instances create frontend \
    --zone=$ZONE \
    --machine-type=e2-standard-2 \
    --tags=frontend \
    --metadata=startup-script-url=https://storage.googleapis.com/fancy-store-$DEVSHELL_PROJECT_ID/startup-script.sh
gcloud compute firewall-rules create fw-fe \
    --allow tcp:8080 \
    --target-tags=frontend
gcloud compute firewall-rules create fw-be \
    --allow tcp:8081-8082 \
    --target-tags=backend
gcloud compute instances list
```

## Task 5. Create managed instance groups
- Create instance template from source instance
```shell
gcloud compute instances stop frontend --zone=$ZONE
gcloud compute instances stop backend --zone=$ZONE
gcloud compute instance-templates create fancy-fe \
    --source-instance-zone=$ZONE \
    --source-instance=frontend
gcloud compute instance-templates create fancy-be \
    --source-instance-zone=$ZONE \
    --source-instance=backend
gcloud compute instance-templates list
gcloud compute instances delete backend --zone=$ZONE
gcloud compute instance-groups managed create fancy-fe-mig \
    --zone=$ZONE \
    --base-instance-name fancy-fe \
    --size 2 \
    --template fancy-fe
gcloud compute instance-groups managed create fancy-be-mig \
    --zone=$ZONE \
    --base-instance-name fancy-be \
    --size 2 \
    --template fancy-be
gcloud compute instance-groups set-named-ports fancy-fe-mig \
    --zone=$ZONE \
    --named-ports frontend:8080
gcloud compute instance-groups set-named-ports fancy-be-mig \
    --zone=$ZONE \
    --named-ports orders:8081,products:8082
```
- Configure autohealing
```shell
gcloud compute health-checks create http fancy-fe-hc \
    --port 8080 \
    --check-interval 30s \
    --healthy-threshold 1 \
    --timeout 10s \
    --unhealthy-threshold 3
gcloud compute health-checks create http fancy-be-hc \
    --port 8081 \
    --request-path=/api/orders \
    --check-interval 30s \
    --healthy-threshold 1 \
    --timeout 10s \
    --unhealthy-threshold 3
gcloud compute firewall-rules create allow-health-check \
    --allow tcp:8080-8081 \
    --source-ranges 130.211.0.0/22,35.191.0.0/16 \
    --network default
gcloud compute instance-groups managed update fancy-fe-mig \
    --zone=$ZONE \
    --health-check fancy-fe-hc \
    --initial-delay 300
gcloud compute instance-groups managed update fancy-be-mig \
    --zone=$ZONE \
    --health-check fancy-be-hc \
    --initial-delay 300
```

## Task 6. Create load balancers
- create health-check that will determine which instances are capable of serving traffic for each service
```shell
gcloud compute http-health-checks create fancy-fe-frontend-hc \
  --request-path / \
  --port 8080
gcloud compute http-health-checks create fancy-be-orders-hc \
  --request-path /api/orders \
  --port 8081
gcloud compute http-health-checks create fancy-be-products-hc \
  --request-path /api/products \
  --port 8082
```

- Create backend services that are the target for load-balanced traffic. The backend services will use the health checks and named ports you created
```shell
gcloud compute backend-services create fancy-fe-frontend \
  --http-health-checks fancy-fe-frontend-hc \
  --port-name frontend \
  --global
gcloud compute backend-services create fancy-be-orders \
  --http-health-checks fancy-be-orders-hc \
  --port-name orders \
  --global
gcloud compute backend-services create fancy-be-products \
  --http-health-checks fancy-be-products-hc \
  --port-name products \
  --global
gcloud compute backend-services add-backend fancy-fe-frontend \
  --instance-group-zone=$ZONE \
  --instance-group fancy-fe-mig \
  --global
gcloud compute backend-services add-backend fancy-be-orders \
  --instance-group-zone=$ZONE \
  --instance-group fancy-be-mig \
  --global
gcloud compute backend-services add-backend fancy-be-products \
  --instance-group-zone=$ZONE \
  --instance-group fancy-be-mig \
  --global
gcloud compute url-maps create fancy-map \
  --default-service fancy-fe-frontend
gcloud compute url-maps add-path-matcher fancy-map \
   --default-service fancy-fe-frontend \
   --path-matcher-name orders \
   --path-rules "/api/orders=fancy-be-orders,/api/products=fancy-be-products"
gcloud compute target-http-proxies create fancy-proxy \
  --url-map fancy-map
gcloud compute forwarding-rules create fancy-http-rule \
  --global \
  --target-http-proxy fancy-proxy \
  --ports 80
```

- Update the configuration
```shell
cd ~/monolith-to-microservices/react-app/
gcloud compute forwarding-rules list --global
```
Return to the Cloud Shell Editor and edit the .env file again to point to Public IP of Load Balancer. [LB_IP] represents the External IP address of the backend instance determined above.

REACT_APP_ORDERS_URL=http://[LB_IP]/api/orders
REACT_APP_PRODUCTS_URL=http://[LB_IP]/api/products

save

- rebuild
```shell
cd ~/monolith-to-microservices/react-app
npm install && npm run-script build
cd ~
rm -rf monolith-to-microservices/*/node_modules
gsutil -m cp -r monolith-to-microservices gs://fancy-store-$DEVSHELL_PROJECT_ID/
gcloud compute instance-groups managed rolling-action replace fancy-fe-mig \
    --zone=$ZONE \
    --max-unavailable 100%
```

- test the website
```shell
watch -n 2 gcloud compute backend-services get-health fancy-fe-frontend --global
```

## Task 7. Scaling Compute Engine
- Automatically resize by utilization
```shell
gcloud compute instance-groups managed set-autoscaling \
  fancy-fe-mig \
  --zone=$ZONE \
  --max-num-replicas 2 \
  --target-load-balancing-utilization 0.60
gcloud compute instance-groups managed set-autoscaling \
  fancy-be-mig \
  --zone=$ZONE \
  --max-num-replicas 2 \
  --target-load-balancing-utilization 0.60
```

- Enable content delivery network
```shell
gcloud compute backend-services update fancy-fe-frontend \
    --enable-cdn --global
```

## Task 8. Update the website
- Updating instance template
```shell
gcloud compute instances set-machine-type frontend \
  --zone=$ZONE \
  --machine-type e2-small
gcloud compute instance-templates create fancy-fe-new \
    --region=$REGION \
    --source-instance=frontend \
    --source-instance-zone=$ZONE
gcloud compute instance-groups managed rolling-action start-update fancy-fe-mig \
  --zone=$ZONE \
  --version template=fancy-fe-new
watch -n 2 gcloud compute instance-groups managed list-instances fancy-fe-mig \
  --zone=$ZONE
gcloud compute instances describe [VM_NAME] --zone=$ZONE | grep machineType
```

- Make changes to the website
```shell
cd ~/monolith-to-microservices/react-app/src/pages/Home
mv index.js.new index.js
cat ~/monolith-to-microservices/react-app/src/pages/Home/index.js
cd ~/monolith-to-microservices/react-app
npm install && npm run-script build
cd ~
rm -rf monolith-to-microservices/*/node_modules
gsutil -m cp -r monolith-to-microservices gs://fancy-store-$DEVSHELL_PROJECT_ID/
gcloud compute instance-groups managed rolling-action replace fancy-fe-mig \
  --zone=$ZONE \
  --max-unavailable=100%
watch -n 2 gcloud compute backend-services get-health fancy-fe-frontend --global
gcloud compute forwarding-rules list --global
```
- simulate failure
```shell
gcloud compute instance-groups list-instances fancy-fe-mig --zone=$ZONE
gcloud compute ssh [INSTANCE_NAME] --zone=$ZONE
sudo supervisorctl stop nodeapp; sudo killall node
exit
watch -n 2 gcloud compute operations list \
--filter='operationType~compute.instances.repair.*'
```

# The Basics of Google Cloud Compute: Challenge Lab
Challenge scenario
You recently started a new role as a Cloud Architect. One of your responsibilities is to build and operate web applications using Google Cloud.

Your challenge
Your manager asked you to build a website to provide products and orders information. You are not asked to write code or scripts as it is another team's responsibility. Your job is to build infrastructure for the web application using Google Cloud. Here are the requirements:

Create a new Cloud Storage bucket to store files.
Create and attach a persistent disk to a Compute Engine virtual machine (VM) instance.
Use Compute Engine to host a web application using a NGINX web server.
## Task 1. Create a Cloud Storage bucket
- 

## Task 2. Create and attach a persistent disk to a Compute Engine instance
- Create a new instance
```shell
ZONE=us-east4-a
gcloud compute instances create my-instance --zone $ZONE --machine-type e2-standard-2

```
- Create a new persistent disk
```shell
gcloud compute disks create mydisk --size=200GB \
--zone $ZONE
```
- Attaching a disk
```shell
gcloud compute instances attach-disk my-instance --disk mydisk --zone $ZONE
```

## Task 3. Install a NGINX web server
- SSH into the Compute Engine instance
```shell
sudo apt-get update
sudo apt-get install -y nginx
ps auwx | grep nginx
```