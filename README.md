# jenkins-distributed-builds
Create a continuous integration system using on demand Jenkins agents and Compute Engine

# Overview
1. Create a new project in Google Cloud
2. Create a service account and grant it the required priviledges
3. Create a Jenkins agent image

# Create a service account from the cloud shell inside your new project and grant it the requisite priviledges
1. Use the gcloud command to create the service account
`gcloud iam service-accounts create jenkins --display-name jenkins`
2. Store your service account email address and Google Cloud Project ID in environment variables for future use
```
export SA_EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:jenkins" --format='value(email)')
export PROJECT=$(gcloud info --format='value(config.project)')
```
3. Grant the following roles to the newly created service account: storage admin, compute instance admin, compute network admin, compute security admin, iam service account actor
```
gcloud projects add-iam-policy-binding $PROJECT \
    --role roles/storage.admin --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.instanceAdmin.v1 \
    --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.networkAdmin \
    --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/compute.securityAdmin \
    --member serviceAccount:$SA_EMAIL
gcloud projects add-iam-policy-binding $PROJECT --role roles/iam.serviceAccountActor \
    --member serviceAccount:$SA_EMAIL
```
## Create the service account key to authenticate with the Compute Engine API in the future
1. Create the service account key and download it to your local machine for future use when configuring the JClouds plugin to authenticate with the Compute Engine API. The following command will generate a key file called `jenkins-sa.json`.
```
gcloud iam service-accounts keys create jenkins-sa.json --iam-account $SA_EMAIL
``` 
2. Click the button at the top of the cloud console that consists of three vertically stacked dots. Then click "Download File".
3. Click the folder icon then click the arrow next to the directory that pops up to browse a list of files. Find the `jenkins-sa.json` file and click it.
4. Click "download" to save the file to your local machine
# Create a Jenkins agent image
We will create a base image for a Compute Engine instance that can be reused when needed and that will contain the necessary software and tools to act as the Jenkins Executor.
## Create an SSH key
Later we will use Packer to build images, which requires SSH to communicate with the build instances. To enable SSH, we must create an SSH key from the cloud shell. 
1. The following command will check for an existing SSH key and use it if it exists, else, it will create a new key pair. 
`ls ~/.ssh/id_rsa.pub || ssh-keygen -N ""`
2. Add the public key to your project's metadata. If you receive a jq error similar to the following `jq: error (at <stdin>:247): Cannot iterate over null (null)`, try adding a `?` directly after the `[]` in the following code block.
```
gcloud compute project-info describe \
    --format=json | jq -r '.commonInstanceMetadata.items[] | select(.key == "ssh-keys") | .value' > sshKeys.pub
echo "$USER:$(cat ~/.ssh/id_rsa.pub)" >> sshKeys.pub
gcloud compute project-info add-metadata --metadata-from-file ssh-keys=sshKeys.pub
```
## Create the base image for the Compute Engine instances
Next we will use Packer to create a base image for our Compute Engine VMs which will act as on-demand temporary build executors in Jenkins. You can customize your build image by adding shell commands to the `provisioners` section of the Packer configuration or by adding other Packer provisioners. 
1. Download and install the latest build of Packer from the Hashicorp website. The following command is using version 1.9.1.
```
wget https://releases.hashicorp.com/packer/1.9.1/packer_1.9.1_linux_amd64.zip
unzip packer_1.9.1_linux_amd64.zip
```
2. Create the configuration file for your Packer image builds
```
export PROJECT=$(gcloud info --format='value(config.project)')
cat > jenkins-agent.json <<EOF
{
  "builders": [
    {
      "machine_type": "n2-standard-1",
      "type": "googlecompute",
      "project_id": "$PROJECT",
      "source_image_family": "ubuntu-2004-lts",
      "source_image_project_id": "ubuntu-os-cloud",
      "zone": "us-central1-f",
      "disk_size": "50",
      "image_name": "jenkins-agent-{{timestamp}}",
      "image_family": "jenkins-agent",
      "ssh_username": "ubuntu"
    }
  ],
  "provisioners": [
    {
      "valid_exit_codes": ["0", "1", "2"],  
      "type": "shell",
      "inline": ["sudo apt-get update && sudo apt-get install -y default-jdk"]
    }
  ]
}
EOF
```
3. Build the image by executing Packer
`./packer build jenkins-agent.json`
4. A successful build will yield an output similar to the following
```
Build 'googlecompute' finished after 3 minutes 14 seconds.

==> Wait completed after 3 minutes 14 seconds

==> Builds finished. The artifacts of successful builds are:
--> googlecompute: A disk image was created: jenkins-agent-1689889295
```
# Install Jenkins
We will use Cloud Marketplace by Bitnami to provision a Jenkins instance that can use the image we built in the previous section.
1. Go to Cloud Marketplace for Jenkins
2. Click on Launch
3. Enable the required APIs
4. Add the correct zone to the configuration
5. Change the Machine Type field to 4 vCPUs 15 GB Memory, n1-standard-4.
6. Click "Deploy"
7. Navigate to the Jenkins login page by clicking "Site Address"
8. Log in to Jenkins using the Admin User credentials listed on the details panel
9. Jenkins is now ready to use. If you experience a 404 error after logging in, simply remove `/jenkins` after the IP address in your browser and it will take you to the Jenkins dashboard.

## Configure Jenkins
First we must install plugins that will allow Jenkins to create agents using Compute Engine and store artifacts from those agents in Cloud Storage.
### Install plugins
1. From the Jenkins dashboard, click "Manage Jenkins"
2. Click "Plugins"
3. Click the "Available Plugins" tab
4. Use the filter to search for the "Google Compute Engine" and "Google Cloud Storage" plugins
5. Select these plugins and click "Download now and install after restart"
6. Click the "Restart Jenkins when installation is complete and no jobs are running" checkbox

### Create plugin credentials
1. From the Jenkins dashboard, click "Manage Jenkins"
2. Click "Credentials"
3. Under the heading "Stores scoped to Jenkins" click the downward arrow next to "global" and select "Add credentials"
4. Set Kind to Google Service Account from private key
5. In the Project Name field, enter your Google Cloud project ID
6. Next to the JSON key option click "Choose File"
7. Add the `jenkins-sa.json` key that you downloaded to your local machine earlier
8. Click "Create"

### Configure Compute Engine plugin
Configure the Jenkins Compute Engine plugin with the credentials it uses to provision agent instances.
1. From the Jenkins dashboard, click "Manage Jenkins"
2. Click "Nodes and Clouds"
3. Click the "Clouds" tab
4. Click "Add" and select "Compute Engine"
5. Under the "Name" field enter `gce`
6. Under the "Project ID" field enter your Google Cloud project ID
7. Under the "Instance Cap" field enter `8`
8. Under "Service Account Credentials" select your service account which will be listed as your Google Cloud project ID
9. Click "Save"

### Configure Jenkins instance configurations
Under Clouds and at the bottom of the Compute Engine configuration panel from the last section, there is a section called "Instance Configurations" that we will use to configure our agent VMs.
1. Click "Add"
2. Under the "Name Prefix" field enter `ubuntu-2004`
3. Under the "Description" field enter `Ubuntu agent`
4. Under the "Labels" field enter `ubuntu-2004`
5. Under the "Region" field select "us-central1"
6. Under the "Zone" field select "us-central1-f"
7. Click "Advanced"
8. Under the "Machine Type" field select "n1-standard-1"
9. Under "Networking" select the "default" setting for both the "Network" and "Subnetwork" fields
10. Check the "Attach External IP?" box
11. Under "Boot Disk" select your Google Cloud project ID under the "Image project" field
12. Under "Image name" select the image you built earlier using Packer
13. Under "Size" enter `50`
14. Click "Save"

# Create a Jenkins job to test the configuration
1. From the Jenkins dashboard, click "Create a job"
2. Enter `test` as the item name
3. Click "Freestyle project"
4. Click "OK"
5. Select the "Execute concurrent builds if necessary" and "Restrict where this project can be run boxes"
6. Under "Label Expression" enter `ubuntu-2004`
7. Under "Build Steps" click "Add build step" and select "Execute shell"
8. Enter `echo "This is a test!"` into the text box
9. Click "Save"
10. Click the "Build Now" tab to start the build

# Upload build artifacts to Cloud Storage
More than likely, you will want to upload artifacts from your builds to Cloud Storage for future analysis or testing. We can configure our Jenkins job to generate a log and build artifact that are both uploaded to Cloud Storage.
1. From Cloud Shell, create a storage bucket for the build artifacts
```
export PROJECT=$(gcloud info --format='value(config.project)')
gsutil mb gs://$PROJECT-jenkins-artifacts
```
2. In the jobs list on the Jenkins UI, select the job we just created. Then click "Configure".
3. Under "Build Steps" change the command text field to `env > build_environment.txt`
4. Under "Post-build Actions" click "Add post-build action"
5. Click "Google Cloud Storage Plugin"
6. Under "Storage Location" enter `gs://[YOUR_PROJECT_ID]-jenkins-artifacts/$JOB_NAME/$BUILD_NUMBER` but substitute your Google Cloud Project ID where indicated
7. Click "Add Operation" and then select "Classic Upload"
8. Under the "File Pattern" heading enter `build_environment.txt`
9. Under "Storage Location" enter `gs://[YOUR_PROJECT_ID]-jenkins-artifacts/$JOB_NAME/$BUILD_NUMBER` but substitute your Google Cloud Project ID where indicated
10. Click the "For failed jobs?" box
11. Click "Save"
12. Click the "Build Now" tab to begin the build
13. From Cloud Shell, access the build artifact using the `gsutil` command
```
export PROJECT=$(gcloud info --format='value(config.project)')
gsutil cat gs://$PROJECT-jenkins-artifacts/test/2/build_environment.txt
```
## Manage artifact lifecycle
Most likely, you will typically be accessing recent build artifacts rather than older ones. To save costs, use object lifecycle management to move older artifacts from higher-performance storage classes to lower-cost and higher-latency storage classes.
1. From Cloud Shell, create a lifecycle configuration file that transfers all build artifacts to Nearline storage after 30 days and all Nearline objects to Coldline storage after 365 days
```
cat > artifact-lifecycle.json <<EOF
{
"lifecycle": {
  "rule": [
  {
    "action": {
      "type": "SetStorageClass",
      "storageClass": "NEARLINE"
    },
    "condition": {
      "age": 30,
      "matchesStorageClass": ["MULTI_REGIONAL", "STANDARD", "DURABLE_REDUCED_AVAILABILITY"]
    }
  },
  {
    "action": {
      "type": "SetStorageClass",
      "storageClass": "COLDLINE"
    },
    "condition": {
      "age": 365,
      "matchesStorageClass": ["NEARLINE"]
    }
  }
]
}
}
EOF
```
2. Upload the configuration file to your artifact storage bucket
```
export PROJECT=$(gcloud info --format='value(config.project)')
gsutil lifecycle set artifact-lifecycle.json gs://$PROJECT-jenkins-artifacts
```
# Deleting Resources
1. Delete any Jenkins agents that are still running
`gcloud compute instances list --filter=metadata.jclouds-group=ubuntu-2004 --uri | xargs gcloud compute instances delete`
2. Using Cloud Deployment Manager, delete the Jenkins instance
`gcloud deployment-manager deployments delete jenkins-1`
3. Delete the Cloud Storage bucket
```
export PROJECT=$(gcloud info --format='value(config.project)')
gsutil -m rm -r gs://$PROJECT-jenkins-artifacts
```
4. Delete the Service Account
```
export SA_EMAIL=$(gcloud iam service-accounts list --filter="displayName:jenkins" --format='value(email)')
gcloud iam service-accounts delete $SA_EMAIL
```





# Resources
1. https://cloud.google.com/architecture/using-jenkins-for-distributed-builds-on-compute-engine
2. https://stackoverflow.com/questions/28213232/docker-error-jq-error-cannot-iterate-over-null
3. https://stackoverflow.com/questions/52684656/the-zone-does-not-have-enough-resources-available-to-fulfill-the-request-the-re
4. https://stackoverflow.com/questions/73337907/packer-build-json-file-for-linux-image-creation-builds-finished-but-no-artifact
5. https://stackoverflow.com/questions/52586941/got-an-error-zone-resource-pool-exhausted-when-creating-a-new-instance-on-goog
6. https://groups.google.com/g/gce-discussion/c/nV1Ym57OCj8

