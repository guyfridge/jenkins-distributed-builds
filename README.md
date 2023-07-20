# jenkins-distributed-builds-gke
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
4. 




# Resources
1. https://cloud.google.com/architecture/using-jenkins-for-distributed-builds-on-compute-engine
2. https://stackoverflow.com/questions/28213232/docker-error-jq-error-cannot-iterate-over-null
3. https://stackoverflow.com/questions/52684656/the-zone-does-not-have-enough-resources-available-to-fulfill-the-request-the-re
4. https://stackoverflow.com/questions/73337907/packer-build-json-file-for-linux-image-creation-builds-finished-but-no-artifact
5. https://stackoverflow.com/questions/52586941/got-an-error-zone-resource-pool-exhausted-when-creating-a-new-instance-on-goog
6. https://groups.google.com/g/gce-discussion/c/nV1Ym57OCj8

